# Universal Patterns — 8 레포 공통 운영 패턴

8 portfolio 레포에서 반복적으로 사용하는 패턴. 한 레포에서 본 패턴을 다른 레포에서 그대로 찾을 수 있도록 형식 통일.

각 패턴: **Problem** (왜 필요한가) → **Solution** (어떻게 해결하나) → **Where** (어느 레포에서 보이나) → **Code** (대표 위치) → **Notes** (주의점).

---

## 1. HikariCP 풀 사이즈 + leak detection

### Problem
JDBC connection 누수 (close 누락) 는 운영 중 silent 하게 시작해서 어느 순간 풀 고갈로 전체 요청 cascade. stack trace 없이는 원인 추적 불가.

### Solution
- `maximum-pool-size` 산정: worker × 평균 동시 처리 + relay/배치 + REST 동시 요청 여유. virtual thread 라도 JDBC 는 OS thread 점유.
- `minimum-idle` 5 — 갑작스런 트래픽 신규 connection 생성 latency 회피.
- `connection-timeout` 3000 — fail-fast 로 호출 측에 빨리 알림.
- `max-lifetime` 1740000 (29분) — DB `idle_in_transaction_session_timeout` (Postgres 30분) / firewall idle close 보다 짧게.
- `idle-timeout` 600000.
- **`leak-detection-threshold` 30000 — connection 30s 넘게 잡으면 stack trace 로 누수 위치 추적**.
- test 프로파일은 `leak-detection-threshold: 0` (Spring context tear-down 이 connection close 보다 늦으면 false positive).

### Where
8 레포 모두.

### Code
- `notification-hub/notification-bootstrap/src/main/resources/application.yml`
- `auth-service/auth-bootstrap/src/main/resources/application.yml`

### Notes
- prod 의 풀 크기는 외부 ENV 로 노출 (`${DB_POOL_MAX:20}`).
- pool-name 명시 — 메트릭 (`hikaricp_connections_active{pool=...}`) 으로 분리 관측.

---

## 2. K8s 3종 probe (startup / readiness / liveness)

### Problem
- `liveness` 만 두면 startup slow 한 앱이 부팅 중 KILL 받음.
- `readiness` 만 두면 deadlock pod 가 영구 unready.
- 외부 의존 (Kafka / Redis / DB) health 가 readiness 에 안 들어가면 트래픽이 죽은 의존으로 흘러감.

### Solution
- **startup**: 부팅 완료까지 max 60s 허용 (`failureThreshold` 12 × `periodSeconds` 5). 그 동안 liveness 평가 안 함.
- **readiness**: 외부 의존 (Kafka / Redis 의존 service) 까지 체크 — `ApplicationReadinessCoordinator` 가 readinessState 직접 갱신 (KafkaHealthIndicator / RedisHealthIndicator 끄고 단일 indicator 로 중복 fail 방지).
- **liveness**: process alive 만. deadlock 만 잡음. 외부 의존 X.
- actuator 의 `health.probes.enabled: true` + `health.group.{readiness,liveness}` 로 매핑.

### Where
8 레포 모두.

### Code
- `notification-hub/notification-bootstrap/src/main/resources/application.yml` (`management.endpoint.health.group`)
- `notification-hub/.../ApplicationReadinessCoordinator.java`

### Notes
- liveness 가 외부 의존 체크하면 **외부 장애가 자기 pod restart 폭주로 이어짐**. 분리 필수.
- K8s `terminationGracePeriodSeconds` + `preStop sleep` 와 함께 운영해야 graceful shutdown 효과.

---

## 3. Graceful shutdown

### Problem
SIGTERM 받자마자 종료하면 in-flight 요청 끊김 + Kafka consumer commit 누락 + JDBC connection drain 안 됨. K8s rolling restart 마다 사용자 에러.

### Solution
- `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase: 25-30s`
- K8s `terminationGracePeriodSeconds: 60` + `preStop: sleep 10s` (kube-proxy iptables 갱신 + endpoint 제거 propagation 시간)
- `SmartLifecycle` 로 vendor 호출 / Outbox relay 단계 await

### Where
8 레포 모두 (가장 명시적: notification-hub, resell-orderbook).

### Code
- `notification-hub/notification-bootstrap/src/main/resources/application.yml` (`spring.lifecycle.timeout-per-shutdown-phase: 25s`)
- `resell-orderbook/infrastructure/k8s/deployment.yaml` (`terminationGracePeriodSeconds: 60` + `preStop sleep 10s`)

### Notes
- 합산: K8s `preStop` (10s) + `timeout-per-shutdown-phase` (25s) + buffer = grace 60s 안에 in-flight 정리 + connection drain.
- `preStop` 가 없으면 endpoint 제거 전 트래픽이 종료 중 pod 로 들어감.

---

## 4. Resilience4j retry + CB + Bulkhead (jitter 포함)

### Problem
- 단순 retry 는 thundering herd → 다중 pod 가 외부 장애 시점에 동시에 같은 hot spot 폭주.
- CB 없이 retry 만 두면 외부 장애가 풀릴 때까지 모든 요청 fail-flood.
- bulkhead 없으면 한 vendor 의 slow 가 다른 vendor 호출 thread 까지 점유.

### Solution
- **retry**: max 3, exponential backoff (200ms → 400ms → 800ms, factor 2.0), `randomized-wait-factor: 0.5` (±50% jitter)
- **CB**: `failure-rate-threshold` 50%, `slow-call-rate-threshold` 50%, `slow-call-duration-threshold` 2s, `permitted-calls-in-half-open-state` 3
- **Bulkhead**: vendor 별 별도 (FCM / SES / Twilio / Kakao 분리)
- whitelist 방식 — `retry-exceptions` 로 5xx/network 만, `ignore-exceptions` 로 4xx 즉시 fail
- attempt-level retry (도메인) 와 vendor-call retry (resilience4j) 를 직렬로 분리 — 같은 호출에 대해 중복으로 polling + retry 가 돌지 않음

### Where
notification-hub (가장 정교), gpu-job-orchestrator, billing-platform.

### Code
- `notification-hub/notification-bootstrap/src/main/resources/application.yml` (`resilience4j.retry.instances`)

### Notes
- jitter 없는 retry 는 *문제를 더 키운다* — 다중 pod 가 같은 backoff 경계에 동시 도달.
- CB open 시 `Fallback` 으로 `markAttemptFailed` → 다음 polling 으로 미루는 방식이 안전.

---

## 5. Outbox + SKIP LOCKED

### Problem
- `kafka.send().get()` 직후 DB transaction commit 사이에 죽으면 메시지는 갔는데 DB 는 PENDING → 재발행 시 중복.
- 한 transaction 에 N 행을 묶으면 N번째 markSent flush 실패가 1..N-1 도 같이 롤백 → Kafka 메시지는 이미 나갔는데 DB 는 PENDING → *재발행*.
- 다중 인스턴스가 같은 행을 동시 picking 하면 또 중복.

### Solution
- **행마다 트랜잭션 분리** — 한 행 fail 영향이 한 행에 갇힘.
- **`SELECT ... FOR UPDATE SKIP LOCKED`** — PostgreSQL: 다른 transaction 이 잡고 있는 행은 건너뛰기. 다중 인스턴스가 같은 행 동시 잡지 않음.
- `KafkaTemplate.send().get(timeout)` — broker 지연으로 행 락이 무기한 잡히지 않게 상한.
- **at-least-once 가 보장** — consumer 측 멱등성 (UNIQUE 제약 등) 으로 중복 흡수.

### Where
notification-hub, mini-shop-observability (order-service), resell-orderbook, gpu-job-orchestrator, billing-platform, security-log-search.

### Code
- `mini-shop-observability/services/order-service/src/main/java/io/minishop/order/outbox/OutboxPoller.java`
- `notification-hub/notification-adapter-out/.../OutboxRelay.java`

### Notes
- Kafka send 를 transaction *안* 에서 호출하는 건 일반적인 안티패턴이지만, 이 경우 *자기 행* 의 lock 만 잡고 발행해야 다른 인스턴스 중복 발행 차단. trade-off 명시.
- 행마다 트랜잭션이라 N 회 commit 이지만, batch 단위 throughput 은 충분 (50 행/초 이상).

---

## 6. Saga + 보상 (REQUIRES_NEW 격리)

### Problem
- 분산 transaction (XA) 은 운영 부담 + 외부 service 가 안 받음.
- 보상 transaction 자체가 실패하면 inconsistency 영구화.
- 같은 saga 가 재시작되면 보상 step 이 중복 실행.

### Solution
- **Saga compensation log** — 각 step 의 input + 결과 + fingerprint 를 별도 테이블 (`compensation_log`) 에 `REQUIRES_NEW` 로 기록. 본 transaction 의 rollback 영향 받지 않음.
- **fingerprint 멱등** — 동일 saga 재시작 시 이미 보상한 step 은 skip.
- **CompensationGuard** — catch 절에서 `store.fail()` 자체가 throw 시 `addSuppressed` 로 묶어 원래 도메인 예외 잠식 방지.

### Where
resell-orderbook (거래 라이프사이클), billing-platform (settlement 흐름), mini-shop-observability (order saga).

### Code
- `resell-orderbook/.../compensation/CompensationGuard.java`
- `billing-platform/.../saga/SettlementSaga.java`

### Notes
- 도메인 예외와 인프라 예외 (compensation log write fail) 를 분리 — log fail 이 원래 예외를 *묻으면* root cause 추적 불가.

---

## 7. Idempotency-Key (Stripe-style 24h 응답 캐싱)

### Problem
- 클라이언트가 timeout 후 retry 하면 결제 / 주문 / 환불 같은 *non-idempotent operation* 이 중복 실행.
- HTTP 응답 자체를 같은 key 에 대해 같은 응답으로 돌려주지 않으면 클라이언트가 다시 retry.

### Solution
- **`Idempotency-Key` 헤더** + 24h TTL 응답 캐싱.
- **body fingerprint 검증** — 같은 key 로 *다른 body* 가 오면 409 Conflict (header 위조 방지).
- 캐싱 storage: Redis (TTL 자동) 또는 DB 테이블.

### Where
resell-orderbook, billing-platform, notification-hub.

### Code
- `resell-orderbook/.../IdempotencyKeyStore.java`
- `notification-hub/.../IdempotencyPort.java`

### Notes
- key 만 같고 body 다른 경우의 처리 미정의 시 보안 표면 — body fingerprint 필수.
- 24h 가 표준 (Stripe API). 시스템에 따라 1h / 7d 도 OK.

---

## 8. Cache stampede 방어 (XFetch + SETNX)

### Problem
- Hot key 의 cache miss 가 동시 발생하면 N 개 요청이 동시 DB 쿼리 → DB 부하 폭주 + 결과를 N번 cache 에 set.
- TTL 만료 직후도 같은 문제 (thundering herd).

### Solution
- **XFetch (probabilistic early refresh)** — TTL 만료 전 확률적으로 미리 cache rebuild. hot key 만 자주 refresh, cold key 는 정상 TTL.
- **`SETNX` lock** — cache rebuild 시 한 인스턴스만 DB query. 나머지는 기존 stale 값 또는 wait.
- 2-tier (L1 caffeine + L2 redis) 시 같은 패턴을 양 layer 에 적용.

### Where
resell-orderbook (`TwoTierMarketStatsCache`), search-service.

### Code
- `resell-orderbook/.../TwoTierMarketStatsCache.java`

### Notes
- polling 시간 (loader 호출 timeout) 이 loader 실제 시간보다 짧으면 fallback 으로 빠져 stampede 방어 무력화 — 충분한 timeout 필수.

---

## 9. Cursor pagination + Snowflake ID

### Problem
- offset/limit 페이지네이션은 (a) 깊은 page 에서 O(N) 스캔 (b) inflight insert/delete 시 row 누락/중복.
- timestamp 만 cursor 로 쓰면 같은 ms 안에 여러 row 시 안정성 깨짐.

### Solution
- **Snowflake ID** — 시간 + machine + sequence 의 단조 증가 ID. timestamp 충돌 없음.
- **cursor = (sortKey, ID)** — sortKey 동률 시 ID tiebreaker.
- `WHERE (sortKey, id) > (?, ?)` 형식 — strict gt (gte 면 경계 row 중복).

### Where
resell-orderbook (`PriceTickRepository`), search-service (saved search), security-log-search (event search).

### Code
- `resell-orderbook/.../SnowflakeIdGenerator.java`
- `search-service/.../ElasticsearchSavedSearchMatchFinder.java` (cursor 의 strict `gt` 사용)

### Notes
- score-tied unstable sort (예: ES `_score` 단독) — `_id` 보조 키 추가로 안정화 필수.
- 새 row inflight 시 cursor 가 stable sort key 위에 있어야 누락/중복 없음.

---

## 10. MDC correlation (traceId / spanId 자동 전파)

### Problem
- 한 사용자 요청이 여러 service 를 거치면 로그가 여러 곳에 흩어짐 — 추적 불가.
- async / executor / Kafka consumer 로 thread 가 전환되면 MDC 가 사라짐.

### Solution
- **`OncePerRequestFilter`** 가 활성 OTel Span 의 `trace_id` / `span_id` 를 MDC 에 set, finally 정리.
- 외부 MDC 키 보존 (filter 가 set 한 키만 정리).
- async / executor / Kafka consumer 의 wrapping decorator 로 propagation.

### Where
mini-shop-observability (`correlation-mdc-starter` v0.1).

### Code
- `mini-shop-observability/modules/correlation-mdc-starter/.../CorrelationMdcFilter.java`

### Notes
- WebFlux / non-web 환경은 별도 Reactor context propagation 필요 — v0.1 은 Servlet 한정.
- log pattern 에 `%X{trace_id}` 추가해야 출력에 보임.

---

## 11. 멀티테넌트 격리 4-layer

### Problem
- 단일 tenant 격리만 하면 한 layer 우회 시 cross-tenant data leak.
- API 단 / DB 단 / 검색 단 / index 단 어디서 새는지 추적 어려움.

### Solution
- **Layer 1 — JWT claim**: 토큰의 `tenant_id` claim 이 모든 호출의 ground truth.
- **Layer 2 — query rewrite**: application 단에서 모든 DB / 검색 query 에 `tenant_id = ?` 자동 추가.
- **Layer 3 — DB Row Policy** (ClickHouse / PostgreSQL RLS): query rewrite 우회 시 DB 단에서 차단.
- **Layer 4 — index alias / partition**: OpenSearch alias 가 tenant 별 분리 — 다른 tenant index 접근 자체 불가.
- platform_admin / global_admin 의 cross-tenant 접근은 별도 audit + ABAC.

### Where
security-log-search.

### Code
- `security-log-search/.../enforceTenant.java`
- `security-log-search/.../OpenSearchEventSearchAdapter.java`
- `security-log-search/.../ClickHouseRowPolicyProvisioner.java`

### Notes
- `WHERE tenant_id IN (?, ?)` 같은 query 가 Row Policy 와 어떻게 상호작용하는지 검증 필요.
- ISMS-P 2.6 (접근 통제) / 2.10 (시스템 및 서비스 보안 관리) 매핑.

---

## 12. JWT 검증 + JWK rotation + grace

### Problem
- JWT signing key 회전 직후 발급된 토큰이 회전 전 client 에서 거부됨 → 사용자 logout 폭주.
- 회전 절차가 atomic 하지 않으면 검증 실패 윈도우 발생.

### Solution
- **previous key 보존** — JWKS 에 `current` + `previous` 동시 노출. 회전 후 grace period 동안 양쪽 모두 검증 통과.
- **kid 매칭** — JWT header 의 `kid` 로 정확한 key 선택. mismatch 시 reject.
- **24h cron rotation** — daily 자동 회전. cron 메서드 직접 호출 가능 (e2e 테스트용).
- **kid spoofing 방지** — `JWSVerificationKeySelector` 가 RS256 강제 + JWKSet 에 자기 IdP 키만.

### Where
auth-service.

### Code
- `auth-service/.../JwkRotationScheduler.java`
- `auth-service/.../auth-bootstrap/src/test/.../JwkRotationE2eTest.java`

### Notes
- timing attack 방지 — signature comparison 은 `MessageDigest.isEqual` 또는 `Arrays.equals` (constant time).
- private key 의 메모리 보호 (`Arrays.fill(byte[], 0)`) 는 별도 작업.

---

## 13. HMAC webhook + replay window

### Problem
- 콜백 URL 만 알면 가짜 "전송 성공" 마킹 가능 — auth 없이 받으면 보안 표면.
- HMAC 만 있고 timestamp 없으면 replay attack (이전 정상 webhook 을 재전송).

### Solution
- **HMAC-SHA256** signature header (`X-Signature: sha256=...`)
- **timestamp header** (`X-Timestamp: <unix_ms>`) + 5min replay window — 이전 / 미래 시각은 reject.
- **`MessageDigest.isEqual`** timing-safe comparison.
- secret rotation grace window — 새 secret 추가 후 7일 grace.

### Where
notification-hub (vendor callback), resell-orderbook (PG webhook), billing-platform (PG webhook).

### Code
- `notification-hub/.../HmacWebhookVerifier.java`

### Notes
- timestamp 의 clock drift 허용 범위 (NTP 미동기화 시 ±60s).
- 운영은 KMS / Vault 로 secret 주입 (env 평문은 임시).

---

## 14. Rate limit (Redis Lua atomic)

### Problem
- Redis SETNX vs SET 의 race window — GET-then-SET 사이 다른 요청이 increment 하면 leak.
- multi-channel fan-out 시 channel#1 차감 후 channel#2 거절 throw → channel#1 토큰만 빠지는 leak.

### Solution
- **Lua script atomic** — 한 호출 안에 GET + 판정 + INCRBY + EXPIRE 가 atomic.
- **`tryConsumeAll(recipient, demand-map)`** — multi-channel 을 한 batch 로 묶어 atomic 차감/거절.
- token bucket (fixed window) — window-ms 별 limit. 더 정교한 sliding window / GCRA 는 별도.

### Where
notification-hub, resell-orderbook (호가 등록 rate limit), auth-service (`/token` endpoint).

### Code
- `notification-hub/.../RedisRateLimiter.java` (`LUA_BATCH_TRY_CONSUME`)

### Notes
- fixed window 의 startover 직후 2*limit 가능 — sliding window / GCRA 로 교체 시점 명시.
- X-Forwarded-For 위조 차단 (trusted proxy allowlist) 와 함께 운영해야 per-IP rate limit 의미 있음.

---

## 15. ShedLock + K8s Lease 이중 leader election

### Problem
- 단일 leader 가 필요한 작업 (예: scheduler / preemption) 이 multi-pod 에서 동시 실행되면 race.
- ShedLock (DB 락) 만 두면 DB 장애 시 leader 결정 못 함.
- K8s Lease 만 두면 K8s API 장애 시 leader 결정 못 함.

### Solution
- **이중 가드** — ShedLock + K8s `coordination.k8s.io/Lease` 둘 다 잡아야 leader.
- ShedLock `lockAtMostFor` 는 작업 시간보다 짧으면 다른 instance 가 takeover (위험), 너무 길면 leader 죽으면 takeover 지연.

### Where
gpu-job-orchestrator, search-service (saved search scheduler).

### Code
- `gpu-job-orchestrator/.../LeaderElection.java`

### Notes
- `lockAtMostFor` 는 작업 시간 + buffer (예: 작업 1s + 5s buffer = 6s, 5분 같은 과대값은 takeover 지연 만듦).
- split brain 가능성 명시 — 두 leader 동시 작동 시 race 가드.

---

## 16. OpenTelemetry — traces / metrics / logs 연결

### Problem
- traces (Tempo) / metrics (Prometheus) / logs (Loki) 가 분리되면 사고 시 한 view 로 못 봄.
- 같은 사용자 요청의 로그를 trace 에서 못 찾음.

### Solution
- **OpenTelemetry Java agent** + **micrometer-tracing-bridge-otel** + **logback-encoder** 의 `traceId` 인코딩.
- log 의 `traceId` 로 Loki → Tempo trace 점프, Tempo → Prometheus 메트릭 점프.
- exemplars — 메트릭에 trace 샘플 link.

### Where
mini-shop-observability (관측성 stack).

### Code
- `mini-shop-observability/infra/docker-compose.yml` (Tempo + Loki + Grafana data source 설정)

### Notes
- correlation-mdc-starter 가 Servlet → MDC propagation. Reactor / async 는 별도 wrapping.
- exemplar 는 Micrometer 1.10+ 필요.

---

## 종합

8 레포의 ADR 100+ 건이 위 16 패턴 위에 쌓여 있습니다. 같은 패턴이 다른 도메인 (결제 / 검색 / SIEM / GPU 스케줄러) 에 적용된 모양을 비교하면 *어디까지가 일반 패턴이고 어디부터가 도메인 특화인지* 가 보입니다.

각 패턴의 *trade-off* 와 *재검토 시점* 은 각 레포의 ADR 본문 참조.

---

## 연락처

- GitHub: [@ssa1004](https://github.com/ssa1004)
- Profile: [github.com/ssa1004](https://github.com/ssa1004)
- Email: wittyahn@gmail.com
