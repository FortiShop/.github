<img src="https://github.com/user-attachments/assets/f3cabc7c-bf5e-40e5-ad15-501e2ad20f48" alt="FortiShop 대시보드" width="100%" />

# FortiShop

> **장애 복원력을 갖춘 MSA 기반 이커머스 플랫폼**
> Kafka 기반 이벤트 아키텍처와 Redis, Elasticsearch, Vault를 활용한 실전형 트래픽 장애 대응 경험 프로젝트

---

## 프로젝트 개요

FortiShop은 실제 대규모 트래픽 상황에서 발생할 수 있는 장애를 직접 실험하고 복구하는 것을 목표로 설계된 MSA 기반 이커머스 플랫폼입니다.

* **MSA 기반 서비스 분리 및 Kafka 기반 비동기 이벤트 아키텍처**
* **장애 발생 → 전파 → 감지 → 복구 전과정 시뮬레이션**
* **적립금 전송, 재고 동시성, 결제 보상 트랜잭션 등의 복잡한 실전 시나리오 구현**

---

## 기술 스택

| 영역           | 기술 스택                                    |
| ------------ | ---------------------------------------------- |
| Language     | Java 17                                        |
| Framework    | Spring Boot, Spring Cloud, Spring Security     |
| API Gateway  | Spring Cloud Gateway (MVC 기반), JWT 인증       |
| Database     | MySQL (Master-Slave), MongoDB                  |
| Cache        | Redis, Redisson                                |
| Messaging    | Apache Kafka                                   |
| Search       | Elasticsearch                                  |
| Config       | Spring Cloud Config + Vault                    |
| Monitoring   | Prometheus, Grafana, Filebeat, Logstash, Kibana |
| Infra & Test | Docker, Testcontainers, JMeter, GitHub Actions |

---

## 서비스 구조

| 서비스명                          | 설명                                   |
| ----------------------------- | ------------------------------------ |
| **edge-service**              | Gateway + 인증/인가 + 회원 관리 + 적립금 API 통합         |
| **config-service**            | 중앙 설정 관리 (Vault 연동)                  |
| **product-inventory-service** | 상품/재고 관리, Redis 캐싱, Elasticsearch 검색 |
| **order-payment-service**     | 주문/결제 처리, Kafka 기반 Saga 트랜잭션         |
| **delivery-service**          | 배송 상태 전이, DLQ 및 중복 방지 처리              |
| **notification-service**      | 이메일/SMS/Slack 알림 전송, DLQ 및 중복 방지 처리  |

---

## 서비스 연동 흐름

```
[사용자 요청]
   ↓
edge-service
   ↓
order-payment-service (Kafka 발행)
   ├─> product-inventory-service (재고 차감)
   ├─> delivery-service (배송 준비)
   └─> notification-service (알림 전송)
```

---

## 서비스별 주요 기능 요약

### edge-service

* 회원 관리
* JWT 인증/인가 + Role 기반 접근 제어
* 적립금 확인, 적립, 사용, 전송 API 제공
* Redis 기반 Rate Limiter, Circuit Breaker 적용
* Gateway + 인증 필터 통합 설계

### product-inventory-service

* 상품 CRUD 및 Redis Sorted Set 기반 인기 상품 캐싱
* Elasticsearch 기반 상품 검색 (장애 시 DB fallback)
* Redisson 분산락 기반 재고 차감 및 Deadlock 실험
* Kafka 기반 재고 보상 트랜잭션 연동

### order-payment-service

* 주문 생성 시 Kafka로 Saga 트랜잭션 시작 (`order.created`)
* 결제 성공/실패 이벤트 발행 → 배송/알림/포인트 서비스 연계
* 실패 시 재고 복구, 결제 취소, 적립금 취소 이벤트 발행

### delivery-service

* 배송 상태 전이 (`READY` → `SHIPPED` → `DELIVERED`)
* 배송 상태 기반 Kafka 이벤트 발행 → 알림 트리거

### notification-service

* Kafka 기반 메시지 수신 → 이메일, SMS, Slack 전송
* DLQ(Dead Letter Queue) 처리 및 메시지 중복 방지 로직 포함
* 관리자용 시스템 경고 메시지 기능 제공

### config-service

* Spring Cloud Config + Vault 기반 설정/보안키 관리
* 서비스별 profile 구성 및 Actuator 연동 재시작 가능

---

## Kafka 메시지 흐름 요약

| Topic                | Producer      | Consumer                    |
| -------------------- | ------------- | --------------------------- |
| `order.created`      | order-payment | inventory, delivery, member |
| `inventory.reserved` | inventory     | order-payment               |
| `payment.completed`  | order-payment | delivery, notification      |
| `point.changed`      | order-payment | member, notification        |
| `delivery.completed` | delivery      | notification                |

메시지마다 `traceId`, `timestamp` 포함하여 분산 추적 가능
Saga 기반 이벤트 순서: `order.created` → `inventory.reserved` → `payment.completed` → `point.changed` → `delivery.started` → `delivery.completed`

---

## 모니터링 구성

FortiShop은 장애를 신속하게 감지하고 원인을 파악하기 위해 다음과 같은 모니터링 시스템을 구축했습니다.

### 구성 도구

* **ELK Stack (Filebeat + Logstash + Elasticsearch + Kibana)**
  → 각 서비스의 로그를 수집 및 정제하여 Kibana를 통해 시각화
* **Prometheus + Grafana**
  → 시스템 메트릭(TPS, GC, Thread, Memory 등)을 실시간 수집 및 대시보드 구성

### 모니터링 항목

| 항목       | 설명                                                  |
| -------- | --------------------------------------------------- |
| 서비스 로그   | Filebeat로 수집한 로그를 Logstash에서 필터링 후 Elasticsearch 저장 |
| 장애 로그 탐지 | Kibana에서 `ERROR`, `WARN`, `StackTrace` 중심으로 탐색      |
| 메트릭 수집   | Prometheus Exporter를 통해 JVM, Redis, Kafka 관련 지표 수집  |
| 지표 시각화   | Grafana 대시보드에서 서비스별 리소스 상태(TPS, GC, 메모리 등) 확인       |
| 알림 설정    | Grafana Alert 또는 외부 연동을 통한 Slack 알림 가능              |

> 장애 실험 중 Redis TTL 만료, Kafka DLQ 처리 여부, DB Deadlock 발생 등을 실시간으로 추적하고 시각화해 원인 분석에 활용했습니다.

---

## 대규모 트래픽 테스트

FortiShop은 실제 트래픽 환경에서의 안정성을 검증하기 위해 **JMeter**를 사용하여 트래픽 부하 테스트를 진행했습니다. 테스트 시나리오는 수백 명의 사용자가 동시에 주문을 생성하는 상황을 가정하였으며, TPS 1,000 이상의 부하 조건에서 시스템의 안정성을 점검했습니다.

### 주요 이슈

* Kafka 메시지가 중복 소비되며 포인트가 이중 적립되는 문제
* 동시에 주문이 발생할 경우 재고 정합성이 맞지 않은 동시성 충돌 현상

### 해결 방안

* Kafka 메시지에 `transactionId` 필드를 추가하고, 포인트 내역 테이블에 UNIQUE 제약 조건을 설정하여 중복 처리 방지
* 재고 차감 시 Redisson 분산 락과 TransactionSynchronizationManager을 조합해 동시 접근 방지
* Kafka DLQ(Dead Letter Queue) 구성을 통해 소비 실패 메시지의 안전한 재처리 구조 마련

### 테스트 결과

* TPS 1,000 이상 조건에서도 서비스는 정상 작동
* Kafka 기반 Saga 트랜잭션이 유실 없이 안정적으로 전파됨
* 장애 상황 발생 시에도 재고 및 포인트 처리의 데이터 정합성이 유지됨

---

## ⚠ 장애 복원 실험 항목

| 장애 유형           | 실험 내용                          |
| --------------- | ------------------------------ |
| Redis           | TTL 만료, 락 실패    |
| Kafka           | 메시지 유실, DLQ 재처리, 중복 소비         |
| DB              | Master 장애 → Slave 전환, Deadlock |
| Circuit Breaker | 장애 전파 차단, fallback 응답 실험       |
| Rate Limiting   | Redis 기반 토큰 버킷 방식              |
| Saga 트랜잭션       | 중간 실패 시 보상 이벤트 발행              |
| Config/Vault    | 설정 파일 손실/장애 시 대처               |

---

## 테스트 전략

* **Testcontainers 기반 Kafka, Zookeeper, MySQL, Redis 환경 테스트**
* **Awaitility, KafkaConsumer 등 활용한 End-to-End 통합 테스트**
* **JMeter 기반 TPS 1000 이상의 부하 테스트**

---

> 기능만 구현하는 개발자를 넘어, 실전 장애를 직접 설계하고 해결할 수 있는 백엔드 시스템을 만들고자 했습니다. FortiShop은 그 도전의 결과물입니다.
