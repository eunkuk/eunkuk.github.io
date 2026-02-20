---
title: "SQS·Redis 대신 Hazelcast Embedded — 분산 메시징 경량화 검토기"
date: 2025-03-09 22:00:00 +0900
categories: [Dev, Architecture]
tags: [hazelcast, distributed-system, messaging, spring-boot, poc, sqs, redis]
description: "외부 인프라 없이 JVM 임베디드로 분산 큐·맵·토픽을 돌릴 수 있을까? SQS와 Redis 대신 Hazelcast를 검토한 PoC 기록."
---

> 외부 인프라 없이 JVM 임베디드로 분산 큐·맵·토픽을 돌릴 수 있을까? SQS와 Redis 대신 Hazelcast를 검토한 PoC 기록.

---

## 1. 상황

분산 환경에서 이벤트 큐 + 상태 저장 + 실시간 알림이 필요한 경우가 있다. 전형적인 선택지는 이렇다:

- **큐**: AWS SQS, RabbitMQ, Kafka
- **캐시/상태**: Redis
- **Pub/Sub**: SNS, Redis Pub/Sub

문제는 이것들이 전부 **외부 인프라 운영**을 전제한다는 점이다. SQS를 쓰려면 AWS 계정과 IAM 설정이 필요하고, Redis를 쓰려면 서버를 따로 띄우고 Sentinel이나 Cluster를 구성해야 한다. 네트워크 설정, 모니터링, 비용까지 따라온다.

"이벤트 처리가 필요하긴 한데, SQS + Redis까지 띄워야 하나?"

더 가벼운 방법은 없는지 검토해 봤다. Hazelcast Embedded — JVM 프로세스 안에 분산 데이터 구조를 내장하는 방식이 눈에 들어왔다.

---

## 2. 선택지: SQS vs Redis vs Hazelcast Embedded

| 항목 | AWS SQS | Redis | Hazelcast Embedded |
|------|---------|-------|--------------------|
| 인프라 | AWS 계정 + SQS 엔드포인트 | Redis 서버 별도 운영 | **JVM 내장, 외부 인프라 없음** |
| 노드 디스커버리 | N/A (관리형) | Sentinel/Cluster 설정 | **Multicast 자동 발견** |
| 큐 | SQS Queue | List/Stream | **IQueue** |
| 키-값 저장 | DynamoDB 등 별도 | Redis Hash/String | **IMap** |
| Pub/Sub | SNS 연동 | Redis Pub/Sub | **ITopic** |
| Visibility Timeout | 네이티브 지원 | 직접 구현 | 앱 레벨 구현 |
| 중복 제거 | FIFO Queue 네이티브 | 직접 구현 | 앱 레벨 구현 |
| 재시도/DLQ | DLQ 네이티브 | 직접 구현 | 앱 레벨 구현 |
| 운영 비용 | AWS 요금 | 서버 + 메모리 | **앱 힙 메모리** |

Hazelcast Embedded의 매력은 명확하다. **외부 프로세스가 없다.** 앱을 띄우면 분산 데이터 구조가 같이 뜬다. 같은 네트워크에 있는 다른 인스턴스를 multicast로 자동 발견하고 클러스터를 형성한다.

반면 SQS가 네이티브로 제공하는 Visibility Timeout, 중복 제거, DLQ 같은 기능은 직접 구현해야 한다. 이 부분이 실제로 얼마나 부담인지 확인하기 위해 PoC를 만들었다.

소스 코드: [eunkuk/hazelcast-test](https://github.com/eunkuk/hazelcast-test) (Spring Boot 3.4.4, Hazelcast 5.3.6)

---

## 3. PoC: 핵심 구현

### a) HazelcastConfig — 분산 데이터 구조 3종

```java
@Bean
public HazelcastInstance hazelcastInstance() {
    Config config = new Config();
    config.setClusterName(clusterName);

    // 네트워크 설정
    NetworkConfig networkConfig = config.getNetworkConfig();
    networkConfig.setPort(port);
    networkConfig.setPortAutoIncrement(true);

    // 멀티캐스트 설정
    JoinConfig joinConfig = networkConfig.getJoin();
    joinConfig.getMulticastConfig().setEnabled(multicastEnabled);

    // 분산 이벤트 큐 설정 (SQS 대체)
    QueueConfig eventQueueConfig = new QueueConfig();
    eventQueueConfig.setName("eventQueue");
    eventQueueConfig.setBackupCount(1);
    eventQueueConfig.setMaxSize(10000);
    config.addQueueConfig(eventQueueConfig);

    // 분산 맵 설정 (Redis 대체)
    MapConfig eventMapConfig = new MapConfig();
    eventMapConfig.setName("eventMap");
    eventMapConfig.setBackupCount(1);
    eventMapConfig.setTimeToLiveSeconds(0); // 무제한
    config.addMapConfig(eventMapConfig);

    // 분산 토픽 설정 (SNS/Redis Pub/Sub 대체)
    TopicConfig eventTopicConfig = new TopicConfig();
    eventTopicConfig.setName("eventTopic");
    config.addTopicConfig(eventTopicConfig);

    return Hazelcast.newHazelcastInstance(config);
}
```

세 가지 분산 데이터 구조를 하나의 JVM 안에 띄운다:

- **IQueue**: SQS 대체. `backupCount=1`로 동기 복제 1벌을 유지한다. 노드 하나가 죽어도 데이터가 남는다.
- **IMap**: Redis 대체. 이벤트 상태를 저장하고 추적한다.
- **ITopic**: SNS/Redis Pub/Sub 대체. 이벤트 발생을 실시간으로 알린다.

`portAutoIncrement(true)` 덕분에 같은 머신에서 여러 인스턴스를 실행하면 5701, 5702, ... 순서로 포트가 할당된다. Multicast가 켜져 있으면 같은 LAN의 인스턴스가 자동으로 클러스터에 합류한다. 별도 설정이 없다.

### b) SQS 기능 패리티 — 어디까지 재현했는가

`DistributedEvent` 모델이 SQS의 핵심 기능을 앱 레벨 필드로 구현한다:

```java
public class DistributedEvent implements Serializable {

    private String eventId;
    private String eventType;
    private String payload;
    private EventStatus status;

    // SQS Visibility Timeout 대체
    private long visibilityTimeoutMs = 30000;
    private Instant visibleAfter;

    // FIFO Queue 대체
    private String messageGroupId;

    // 중복 제거 대체
    private String deduplicationId;

    // DLQ 대체
    private int retryCount = 0;
    private int maxRetries = 3;
    private long retryDelayMs = 5000;

    // 멀티노드 추적
    private String sourceServerId;
    private String processingServerId;
}
```

SQS 기능별로 대응하면:

| SQS 기능 | Hazelcast PoC 구현 방식 |
|----------|----------------------|
| Visibility Timeout | `visibleAfter` 필드 + 컨슈머가 poll 시 체크, 미도래 시 re-enqueue |
| FIFO (순서 보장) | `messageGroupId` 필드 |
| 중복 제거 | `deduplicationId` + 로컬 캐시 (`Map<String, Instant>`) |
| DLQ (재시도 소진) | `retryCount`/`maxRetries` + 초과 시 FAILED 마킹 |
| 재시도 딜레이 | `retryDelayMs` + `visibleAfter` 재설정 |

### c) 컨슈머: 5-스레드 폴링 + Visibility Timeout 체크

`EventConsumerService`가 이벤트를 소비한다. 핵심 로직:

```java
@PostConstruct
public void init() {
    executorService = Executors.newFixedThreadPool(6); // 5 컨슈머 + 1 콜백 재시도

    for (int i = 0; i < 5; i++) {
        final int threadId = i;
        executorService.submit(() -> processEvents(threadId));
    }

    executorService.submit(this::processCallbackRetries);
}
```

각 스레드는 큐를 폴링하면서 세 가지를 체크한다:

```java
private void processEvents(int threadId) {
    IQueue<DistributedEvent> eventQueue = hazelcastInstance.getQueue("eventQueue");
    IMap<String, DistributedEvent> eventMap = hazelcastInstance.getMap("eventMap");

    while (running.get()) {
        DistributedEvent event = eventQueue.poll(3, TimeUnit.SECONDS);

        if (event != null) {
            // 1. Visibility Timeout 체크
            if (event.getVisibleAfter() != null && Instant.now().isBefore(event.getVisibleAfter())) {
                eventQueue.offer(event); // 아직 안 보이는 메시지 → 다시 큐로
                continue;
            }

            // 2. 중복 제거 체크
            if (event.getDeduplicationId() != null) {
                synchronized (processedDeduplicationIds) {
                    if (processedDeduplicationIds.containsKey(event.getDeduplicationId())) {
                        continue; // 이미 처리한 메시지
                    }
                }
            }

            // 3. 처리 + 재시도 로직
            try {
                event.setStatus(EventStatus.PROCESSING);
                event.setProcessingServerId(serverIdentificationService.getServerId());
                eventMap.put(event.getEventId(), event);

                processEvent(event); // 비즈니스 로직

                event.setStatus(EventStatus.COMPLETED);
                eventMap.put(event.getEventId(), event);
            } catch (Exception e) {
                event.setRetryCount(event.getRetryCount() + 1);
                if (event.getRetryCount() < event.getMaxRetries()) {
                    event.setVisibleAfter(Instant.now().plusMillis(event.getRetryDelayMs()));
                    eventQueue.offer(event); // 딜레이 후 재시도
                } else {
                    event.setStatus(EventStatus.FAILED); // DLQ 대체: FAILED 마킹
                    eventMap.put(event.getEventId(), event);
                }
            }
        }
    }
}
```

SQS에서 네이티브로 제공하는 것들 — Visibility Timeout, 중복 제거, 재시도/DLQ — 을 전부 앱 코드로 구현하고 있다. 동작은 하지만 코드가 상당히 길다.

### d) 프로듀서: IMap 저장 → IQueue 투입 → ITopic 발행

```java
public DistributedEvent produceEvent(String eventType, String payload, ...) {
    DistributedEvent event = DistributedEvent.builder()
            .eventType(eventType)
            .payload(payload)
            .sourceServerId(serverIdentificationService.getServerId())
            .build();

    // 1. 이벤트 상태 맵에 저장 (Redis 대체)
    IMap<String, DistributedEvent> eventMap = hazelcastInstance.getMap("eventMap");
    eventMap.put(event.getEventId(), event);

    // 2. 이벤트 큐에 추가 (SQS 대체)
    IQueue<DistributedEvent> eventQueue = hazelcastInstance.getQueue("eventQueue");
    boolean offered = eventQueue.offer(event, 5, TimeUnit.SECONDS);

    if (offered) {
        event.setStatus(EventStatus.QUEUED);
        eventMap.put(event.getEventId(), event);

        // 3. 이벤트 토픽에 발행 (실시간 알림)
        ITopic<DistributedEvent> eventTopic = hazelcastInstance.getTopic("eventTopic");
        eventTopic.publish(event);
    }

    return event;
}
```

IMap에 저장 → IQueue에 투입 → ITopic으로 브로드캐스트. 세 개의 분산 데이터 구조가 하나의 흐름으로 엮인다.

### e) 멀티노드 추적

`ServerIdentificationService`가 각 노드에 고유 ID를 부여한다:

```java
@PostConstruct
public void init() {
    serverHost = InetAddress.getLocalHost().getHostName();
    serverId = serverHost + ":" + serverPort + "-" + UUID.randomUUID().toString().substring(0, 8);
}
```

이벤트에 `sourceServerId`(생성 노드)와 `processingServerId`(처리 노드)를 기록한다. "서버 A가 만든 이벤트를 서버 B가 처리했다"를 추적할 수 있다. 분산 환경에서 디버깅할 때 필수적인 정보다.

---

## 4. 채택하지 않은 이유

검토 결과 Hazelcast Embedded의 한계가 명확했다.

| 항목 | 문제 |
|------|------|
| **SQS 기능 패리티가 앱 코드** | Visibility Timeout, 중복 제거, FIFO 등이 SQS에서는 네이티브지만, Hazelcast에서는 전부 앱 레벨 구현. 유지보수 부담이 큼 |
| **힙 메모리 의존** | 모든 데이터가 JVM 힙에 존재. 큐가 커지면 앱 메모리에 직접 영향 |
| **영속성** | 기본적으로 인메모리. 전체 클러스터가 내려가면 데이터 유실. MapStore로 DB 백업 가능하지만 추가 구현 필요 |
| **Multicast 제약** | 클라우드(AWS, GCP)에서 multicast 안 됨. TCP-IP 또는 Kubernetes 디스커버리 플러그인 필요 |
| **운영 가시성** | SQS Console, Redis CLI 같은 성숙한 운영 도구 부재. Hazelcast Management Center는 유료 |

PoC의 `EventConsumerService`를 보면 문제가 체감된다. Visibility Timeout 체크, 중복 제거, 재시도 딜레이, DLQ 마킹 — 이 로직이 전부 앱 코드에 있다. SQS를 쓰면 SDK 호출 한 줄로 끝나는 일이다.

> "SQS의 기능을 Hazelcast 위에 직접 만드는 건, SQS가 해주는 일을 내가 떠안는 것이다."

힙 메모리 문제도 실질적이다. `IQueue`의 `maxSize(10000)`은 JVM 힙에 10,000개의 직렬화된 객체가 올라간다는 뜻이다. 이벤트 크기가 커지거나 큐가 밀리면 앱 자체가 OOM으로 죽을 수 있다. SQS는 무제한, Redis도 별도 메모리 공간이니 앱과 분리된다.

클라우드 환경에서 multicast가 안 되는 것도 치명적이다. AWS VPC, GCP VPC 모두 multicast를 지원하지 않는다. 로컬 개발에서는 자동 디스커버리가 잘 되지만, 운영 환경에서는 TCP-IP 목록을 직접 관리하거나 Kubernetes 디스커버리 플러그인(`hazelcast-kubernetes`)을 써야 한다. "외부 인프라 없이"라는 장점이 반감된다.

---

## 5. 돌아보며

Hazelcast Embedded는 **외부 인프라 없이 분산 데이터 구조를 쓸 수 있다**는 점에서 매력적이다. `IQueue`, `IMap`, `ITopic` — 이 세 가지만으로 큐, 캐시, Pub/Sub를 하나의 JVM 안에서 해결한다. 클러스터 형성도 multicast로 자동이다.

하지만 SQS 수준의 메시징 시맨틱이 필요한 순간, 이야기가 달라진다. Visibility Timeout, DLQ, 중복 제거 — 이걸 앱 코드로 직접 구현하면 코드 복잡도가 올라간다. "가볍게 쓰고 싶어서" 시작했는데, SQS 기능을 재현하다 보면 결국 SQS를 직접 만들고 있는 셈이다.

**Hazelcast Embedded가 적합한 경우:**
- 캐시 + 간단한 분산 맵/큐가 필요한 경우
- 메시징 시맨틱이 단순한 경우 (Visibility Timeout, Exactly-Once 불필요)
- 인프라 운영이 불가능하거나 극도로 제한된 환경

**부적합한 경우:**
- SQS 수준의 보장이 필요한 경우 (Visibility Timeout, Exactly-Once, DLQ)
- 큐 크기가 커질 수 있는 경우 (힙 메모리 제약)
- 클라우드 환경에서 운영해야 하는 경우 (multicast 제약)

결론은 **인프라 비용 vs 코드 복잡도의 트레이드오프**다. 외부 인프라를 운영하는 비용을 줄이려 했지만, 그 비용이 앱 코드의 복잡도로 전이된다. 우리 경우 SQS/Redis를 유지하는 것이 더 단순했다.

> 인프라가 해주는 일을 코드로 옮기면, 인프라 비용은 줄지만 코드 유지보수 비용이 늘어난다. 둘 중 어느 쪽이 더 큰지는 상황에 따라 다르다.
