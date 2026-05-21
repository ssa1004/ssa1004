# API 문서

포트폴리오 REST 서비스의 OpenAPI 3 spec 을 한곳에 모은다.

각 서비스는 빌드 시 `springdoc-openapi-gradle-plugin` 으로 OpenAPI spec 을 export 해
자기 repo 의 `docs/openapi/` 에 둔다. 이 페이지는 그 spec 들로 가는 링크 표와
한 화면에서 여러 spec 을 탐색하는 통합 뷰어를 제공한다.

> GraphQL gateway (`graphql-gateway`) 는 GraphQL schema (SDL) 라 OpenAPI 대상이 아니다.

## 통합 뷰어

`docs/api/index.html` — Redoc standalone (CDN). 드롭다운으로 11 개 spec 을 전환한다.

GitHub 에서 HTML 은 raw 로 열면 렌더링되지 않으므로 아래 중 하나로 본다.

```bash
# 1) 로컬에서 바로 열기
open docs/api/index.html        # macOS

# 2) 정적 서버로 서빙 (CORS 안전)
npx http-server docs/api -p 8088 && open http://localhost:8088
```

GitHub Pages 를 켜면 `https://ssa1004.github.io/ssa1004/docs/api/` 로도 접근된다.

## Spec 링크

각 spec 은 해당 repo `main` 브랜치의 `docs/openapi/<service>.yaml` 이다.
(spec yaml 은 각 repo CI 에서 생성·갱신된다.)

| 서비스 | repo | OpenAPI spec (raw) |
|--------|------|--------------------|
| auth-service | [auth-service](https://github.com/ssa1004/auth-service) | [auth-service.yaml](https://raw.githubusercontent.com/ssa1004/auth-service/main/docs/openapi/auth-service.yaml) |
| security-log-search | [security-log-search](https://github.com/ssa1004/security-log-search) | [security-log-search.yaml](https://raw.githubusercontent.com/ssa1004/security-log-search/main/docs/openapi/security-log-search.yaml) |
| notification-hub | [notification-hub](https://github.com/ssa1004/notification-hub) | [notification-hub.yaml](https://raw.githubusercontent.com/ssa1004/notification-hub/main/docs/openapi/notification-hub.yaml) |
| search-service | [search-service](https://github.com/ssa1004/search-service) | [search-service.yaml](https://raw.githubusercontent.com/ssa1004/search-service/main/docs/openapi/search-service.yaml) |
| billing-platform | [billing-platform](https://github.com/ssa1004/billing-platform) | [billing-platform.yaml](https://raw.githubusercontent.com/ssa1004/billing-platform/main/docs/openapi/billing-platform.yaml) |
| resell-orderbook | [bid-ask-marketplace](https://github.com/ssa1004/bid-ask-marketplace) | [resell-orderbook.yaml](https://raw.githubusercontent.com/ssa1004/bid-ask-marketplace/main/docs/openapi/resell-orderbook.yaml) |
| gpu-job-orchestrator | [gpu-job-orchestrator](https://github.com/ssa1004/gpu-job-orchestrator) | [gpu-job-orchestrator.yaml](https://raw.githubusercontent.com/ssa1004/gpu-job-orchestrator/main/docs/openapi/gpu-job-orchestrator.yaml) |
| commerce-ops / order-service | [commerce-ops](https://github.com/ssa1004/commerce-ops) | [order-service.yaml](https://raw.githubusercontent.com/ssa1004/commerce-ops/main/docs/openapi/order-service.yaml) |
| commerce-ops / payment-service | [commerce-ops](https://github.com/ssa1004/commerce-ops) | [payment-service.yaml](https://raw.githubusercontent.com/ssa1004/commerce-ops/main/docs/openapi/payment-service.yaml) |
| commerce-ops / inventory-service | [commerce-ops](https://github.com/ssa1004/commerce-ops) | [inventory-service.yaml](https://raw.githubusercontent.com/ssa1004/commerce-ops/main/docs/openapi/inventory-service.yaml) |
| realtime-feed-service | [realtime-feed-service](https://github.com/ssa1004/realtime-feed-service) | [realtime-feed-service.yaml](https://raw.githubusercontent.com/ssa1004/realtime-feed-service/main/docs/openapi/realtime-feed-service.yaml) |

## 보는 법

### Redoc

읽기 전용 레퍼런스 문서. 통합 뷰어(`index.html`) 가 Redoc 기반이다. 단건은:

```bash
npx @redocly/cli preview-docs <spec-url-or-path>
```

### Swagger UI

요청을 직접 쏴 보는 인터랙티브 뷰. 각 서비스를 띄우면 `/swagger-ui.html` 에서 열린다.
(realtime-feed-service 는 WebFlux functional routing 이라 UI 미포함 — spec 만 노출.)

```bash
# 예: auth-service 를 띄운 뒤
open http://localhost:8080/swagger-ui.html
```

## SDK codegen

모인 spec 은 클라이언트 SDK 생성의 입력으로 쓸 수 있다.

```bash
# 예: openapi-generator 로 TypeScript 클라이언트 생성
npx @openapitools/openapi-generator-cli generate \
  -i https://raw.githubusercontent.com/ssa1004/auth-service/main/docs/openapi/auth-service.yaml \
  -g typescript-fetch \
  -o ./sdk/auth-service
```
