---
layout: post
title: "Node.js 컨테이너, 왜 깔끔하게 안 죽을까? (feat. Graceful shutdown)"
subtitle: SIGTERM, PID 1, 그리고 Node.js 이벤트 루프
date: 2026-01-20 00:00:00 +0900
category: dev
background: "/img/simple-background.png"
author: sebastian, ruru
comments: true
tags:
  - nodejs
  - kubernetes
  - graceful-shutdown
  - nestjs
---

> 그레이스풀 셧다운(Graceful Shutdown)이란 서버가 종료 요청을 받았을 때, 진행 중인 작업을 안전하게 마무리하고 리소스를 정리한 뒤 종료하는 방식입니다.

이 글에서는 Node.js 컨테이너 환경에서 graceful shutdown이 제대로 동작하지 않는 원인을 분석하고, Linux PID 1 메커니즘과 이벤트 루프 관점에서 해결 방법을 다룹니다.

배치 컨슈머 앱을 운영하다 보면 "분명 종료 시그널 넣었는데 왜 안 죽지?"라는 상황을 한 번쯤 겪게 됩니다.

저희 팀 역시 최근 이 문제로 꽤 고생했습니다. 처음에는 단순히 시그널 처리 문제라고 생각했지만, 파고들다 보니 **Linux 커널의 PID 1 보호 메커니즘**과 **Node.js 이벤트 루프 동작 방식**이 함께 얽힌 문제였습니다. 그 과정에서 겪은 삽질을 정리해봅니다.

---

## 0. 배경

모두의주차장 서비스에서는 여러 배치 컨슈머 앱을 운영하고 있습니다.

운영 중, 배치 job이 아예 실행되지 않은 것은 아니지만 **배치는 도는 것처럼 보이는데 일부 작업만 반영되지 않은 채 끝난 것처럼 보이는 케이스**가 간헐적으로 발견되었습니다.

처음에는 배치 로직이나 트랜잭션 문제를 의심했습니다. 하지만 원인이 깔끔하게 재현되지 않았고, 상황에 따라 증상도 달랐습니다. 이 과정에서 배치 코드 자체뿐 아니라, **배치가 실행되는 동안 프로세스가 어떻게 종료되는지**도 함께 살펴볼 필요가 있겠다는 생각이 들었습니다.

배포 타이밍, 종료 시그널, graceful shutdown 역시 **가능성 있는 원인 중 하나로 열어두고** 고민을 시작했습니다.

이때부터 고민의 방향이 바뀌었습니다.

- 배치 컨슈머에서 트랜잭션은 어디까지 보장해야 할까?
- 배치가 실행 중일 때 새로운 배포가 나가면, 어디까지를 정상 종료로 봐야 할까?
- Kubernetes 환경에서 말하는 graceful shutdown은 실제로 어떤 의미일까?

단순히 "SIGTERM을 잘 받게 하자"는 문제는 아니라는 판단이 들었고, 결국 **프로세스 종료 과정을 처음부터 다시 이해해볼 필요가 있다**고 느꼈습니다.

---

## 1. 내 앱은 왜 시그널을 무시할까?

Kubernetes는 Pod를 종료할 때 먼저 SIGTERM을 보냅니다. 애플리케이션은 이 신호를 받고 하던 작업을 마무리해야 하지만, 제 경우에는 배치가 계속 실행되다가 결국 SIGKILL로 강제 종료되고 있었습니다.

### 흔한 오해

구글링을 해보면 "Node.js는 PID 1 역할을 하도록 설계되지 않아서 시그널을 못 받는다"라는 설명을 자주 볼 수 있습니다. 하지만 이는 절반만 맞는 이야기입니다.

실제로는 **Linux 커널이 PID 1 프로세스를 특별하게 보호**합니다. 일반 프로세스(PID ≥ 2)는 시그널 핸들러가 없으면 커널의 기본 동작에 따라 종료됩니다. 하지만 PID 1은 핸들러가 없을 경우 시그널을 무시합니다. 이는 "Global init gets no signals it doesn't want"라는 커널 설계 원칙 때문입니다.

NestJS에서 [`app.enableShutdownHooks()`](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown)를 통해 시그널 핸들러를 등록했다면 PID 1이라도 SIGTERM을 받을 수는 있습니다. 다만 실제 문제는 **좀비 프로세스 정리(reaping)**와 **자식/손자 프로세스에 대한 시그널 전파를 Node.js가 책임지지 않는다는 점**이었습니다.

### 해결: dumb-init 도입

결국 시그널 전달과 프로세스 관리는 전문 init 시스템에 맡기는 것이 표준적인 접근이었습니다. [dumb-init](https://github.com/Yelp/dumb-init)은 컨테이너 환경에서 PID 1로 동작하며 시그널 전파와 좀비 프로세스 정리를 담당합니다.

**dumb-init을 활용한 Dockerfile 예시:**

```dockerfile
# Dockerfile - dumb-init을 PID 1로 설정하여 시그널 전달 보장
RUN apt-get update && apt-get install -y dumb-init

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["node", "dist/main"]
```

---

## 2. 2분 타임아웃 걸었는데 왜 5분을 버티지?

dumb-init을 적용한 뒤 시그널은 정상적으로 전달되기 시작했습니다. 하지만 또 다른 문제가 드러났습니다.

`onModuleDestroy` 훅에서 `Promise.race`를 사용해 최대 2분까지만 기다리도록 구현했습니다. 그런데 실제로는 배치가 끝날 때까지 약 5분 동안 Pod가 종료되지 않았습니다.

### 이벤트 루프의 문제

[Node.js 프로세스는 이벤트 루프가 완전히 비워져야 종료](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)됩니다. 상황을 정리하면 다음과 같았습니다.

- `Promise.race`에서 타임아웃이 먼저 완료되어 훅 함수는 return됨
- 하지만 패배한 `batchPromise`는 취소되지 않음
- 내부의 `await sleep(10000)` 같은 비동기 작업이 이벤트 루프에 계속 남아 있음
- Node.js는 "아직 처리할 작업이 남아 있다"고 판단하고 종료를 미룸

**타임아웃 후에도 프로세스가 종료되지 않는 로그 예시:**

```log
16:07:20  K8s SIGTERM 수신 -> onModuleDestroy 호출
16:09:20  2분 타임아웃 발생 -> 훅 함수 종료 (return)
16:09:26  (종료되어야 하는데) 배치 작업 계속 진행 중... Iteration 15...
16:11:56  5분 경과, 배치가 다 끝나서야 프로세스 종료
```

함수가 끝났다고 해서, 프로세스가 종료되는 것은 아니었습니다.

---

## 3. AbortController로 강제 중단해야 할까?

타임아웃 이후에도 배치가 계속 실행되는 상황을 보며 비동기 작업 자체를 강제로 중단해야 하는지 고민했습니다.

`AbortController`를 쓰면 백그라운드에서 돌고 있는 루프를 명시적으로 멈출 수 있거든요.

**검토했으나 미채택한 AbortController 패턴:**

```typescript
async doBatch() {
  for (const job of jobs) {
    if (this.abortController.signal.aborted) {
      console.log('작업 중단 요청 수신');
      break;
    }
    await perform(job);
  }
}

async onModuleDestroy() {
  const result = await Promise.race([waitAll, timeoutPromise]);
  if (result === 'timeout') {
    this.abortController.abort(); // 중단 신호 전송
  }
}
```

**다만 최종적으로는 이 방식을 도입하지 않았습니다.**

- 모든 루프와 await 지점마다 중단 체크가 필요해 코드 복잡도가 증가함
- DB 트랜잭션 등 외부 라이브러리가 중단을 안전하게 처리하지 못할 가능성
- 데이터 정합성 측면에서 오히려 더 위험해질 수 있음

결국 이미 시작된 배치는 끝까지 보장하고, 그 이후는 Kubernetes의 종료 정책에 맡기는 방향을 선택했습니다.

---

## 4. 최종 버전

### 애플리케이션 레벨: 셧다운 훅 + 타임아웃

**최종 적용한 NestJS 셧다운 훅:**

```typescript
// main.ts
app.enableShutdownHooks();

// batch.service.ts
async onModuleDestroy() {
  console.log(`onModuleDestroy 호출됨 (PID: ${process.pid})`);

  const waitAll = Promise.allSettled(this.runningBatches);
  const timeoutPromise = new Promise(resolve =>
    setTimeout(() => resolve('timeout'), 120000)
  );

  const result = await Promise.race([waitAll, timeoutPromise]);

  if (result === 'timeout') {
    console.error('Graceful Shutdown 타임아웃 - 배치 완료 대기 중');
    // 정리가 불가능하면 process.exit(1)로 강제 종료할 수도 있음
  }
}
```

### 인프라 레벨: K8s Grace Period 조정

애플리케이션 타임아웃보다 [`terminationGracePeriodSeconds`](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)를 더 길게 설정했습니다.

**앱 타임아웃(2분) < K8s Grace Period(3분)**

**앱 타임아웃보다 길게 설정한 K8s Grace Period:**

```yaml
# deployment.yaml
spec:
  template:
    spec:
      containers:
        - name: my-app
      terminationGracePeriodSeconds: 180
```

---

## 정리

이번 이슈를 통해 확인한 것은 단순합니다. 종료 훅을 추가했는지보다 중요한 것은, **종료 시점에 이벤트 루프에 어떤 작업이 남아 있는지를 이해하고 있는지**였습니다.

Node.js 이벤트 루프, PID 1 프로세스, Kubernetes 종료 정책은 서로 맞물려 동작합니다. 이 중 하나라도 놓치면 "종료됐다고 생각했지만 실제로는 살아 있는" 상태가 만들어질 수 있습니다.

모든 상황을 코드로 통제하려 하기보다는, 문제의 성격과 서비스 요구사항을 기준으로 **애플리케이션과 인프라의 책임을 나누는 선택**이 더 현실적이었습니다.

---

### 핵심 요약

- **Init 프로세스 사용**: [dumb-init](https://github.com/Yelp/dumb-init)이나 [tini](https://github.com/krallin/tini)로 시그널 전달 및 좀비 프로세스 방지
- **Node 직접 실행**: `npm start` 대신 `node dist/main`으로 (시그널 전달 방해 방지)
- **이벤트 루프 이해**: return만으로 프로세스가 종료되지 않음. [비동기 작업이 남았으면 끝날 때까지 살아있음](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- **설정 동기화**: 앱 타임아웃보다 K8s [terminationGracePeriodSeconds](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)를 더 길게

결국 종료 훅을 넣는 것보다 중요한 건, **내 앱의 비동기 작업이 이벤트 루프를 얼마나 점유하고 있는지 이해하고 제어하는 것**이었습니다. 개발적인 관점 너머, 여러가지 기술적인 해결책들이 있을때, 오버엔지니어링 보다는 이 문제의 배경이 무엇이고, 풀고자 하는 문제점이 무엇인지 정확하게 파악 후 해결책을 선택하는 것 또한 중요하다고 생각합니다.
