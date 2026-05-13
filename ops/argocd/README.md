# ops/argocd — 9 service GitOps 통합 배포

`ssa1004` 9 개 레포의 Helm chart 를 ArgoCD ApplicationSet 한 묶음으로 배포한다.
각 레포가 자기 chart 를 관리하고, profile repo 는 이를 묶는 manifest 만 들고 있다.

## 구성

| 파일 | 역할 |
|------|------|
| `projects.yaml` | AppProject `ssa1004-portfolio` — sourceRepos / destination / resource whitelist |
| `applicationset.yaml` | 기본 (single env) ApplicationSet — `values.yaml` 만 사용 |
| `applicationset-dev.yaml` | dev 환경 — namespace `<name>-dev`, `values-dev.yaml` override |
| `applicationset-prod.yaml` | prod 환경 — namespace `<name>-prod`, `values-prod.yaml` override, prune 수동 |

## 사용법

```bash
# 1. AppProject 먼저 생성
kubectl apply -f ops/argocd/projects.yaml

# 2. ApplicationSet 적용 — single env (default)
kubectl apply -f ops/argocd/applicationset.yaml

# 또는 env 별 분리
kubectl apply -f ops/argocd/applicationset-dev.yaml
kubectl apply -f ops/argocd/applicationset-prod.yaml

# 3. 생성된 Application 확인
kubectl get applications -n argocd
argocd app list
```

## 9 service chart 매핑

| service | repo | chart path | namespace (single env) |
|---------|------|------------|-----------------------|
| auth-service | `ssa1004/auth-service` | `helm/auth-service` | `auth-service` |
| security-log-search | `ssa1004/security-log-search` | `helm/security-log-search` | `security-log-search` |
| notification-hub | `ssa1004/notification-hub` | `helm/notification-hub` | `notification-hub` |
| search-service | `ssa1004/search-service` | `helm/search-service` | `search-service` |
| billing-platform | `ssa1004/billing-platform` | `helm/billing-platform` | `billing-platform` |
| resell-orderbook | `ssa1004/resell-orderbook` | `helm/resell-orderbook` | `resell-orderbook` |
| gpu-job-orchestrator | `ssa1004/gpu-job-orchestrator` | `helm/gpu-job-orchestrator` | `gpu-job-orchestrator` |
| mini-shop | `ssa1004/commerce-ops` | `helm/mini-shop` | `mini-shop` |
| realtime-feed-service | `ssa1004/realtime-feed-service` | `helm/realtime-feed-service` | `realtime-feed-service` |

> `security-log-search` 만 chart 가 `infrastructure/helm/` 하위에 있다. 나머지 8 개는 `helm/<chart-name>/` 표준 위치.

## 환경 분기 가이드

두 가지 방법 중 선택:

1. **env 별 ApplicationSet 분리** (현재 방식)
   - `applicationset-dev.yaml`, `applicationset-prod.yaml` 각각 적용.
   - namespace 가 `<name>-dev` / `<name>-prod` 로 분리되고 `values-{env}.yaml` 이 자동 override.
   - 운영 / 개발 cluster 가 분리되어 있을 때 권장.

2. **단일 ApplicationSet + matrix generator**
   - 한 ApplicationSet 안에서 list (service) × list (env) 형태로 cross product.
   - 단순하지만 env 별 정책 분기가 ApplicationSet 한 파일에 모이게 됨.

`applicationset-prod.yaml` 은 `syncPolicy.automated.prune: false` 로 prune 을 수동 처리하도록 했다.
prod 에서 chart 변경 후 의도치 않은 리소스 삭제가 일어나지 않게 보호한다.

## sync wave / 의존성 순서

ArgoCD `sync-wave` 어노테이션으로 배포 순서를 제어할 수 있다.
`auth-service` 가 JWT 검증 IdP 라 다른 서비스보다 먼저 떠야 하므로 `sync-wave: -1` 을 주고,
나머지 8 서비스는 `sync-wave: 0` 으로 동시 배포한다.

template 의 annotations 에 다음을 추가하면 된다 (필요 시):

```yaml
template:
  metadata:
    annotations:
      argocd.argoproj.io/sync-wave: '{{syncWave}}'
```

그리고 list generator 의 각 element 에 `syncWave: "-1"` (auth-service) / `"0"` (그 외) 를 부여.
현재 manifest 는 ApplicationSet 단위에서만 묶음 배포를 한다 (개별 서비스 startup probe 가
외부 의존을 직접 체크하므로 의존성이 만족될 때까지 readiness 가 false 로 유지된다).

## 사전 준비

- ArgoCD v2.x 가 설치된 cluster (`argocd` namespace).
- profile repo 가 ArgoCD source repo 로 등록되어 있을 필요는 없다 (ApplicationSet 의 source 는
  9 개 service repo 라 ApplicationSet manifest 자체는 `kubectl apply` 로 적용한다).
- 각 service repo 에 Helm chart 가 `Chart.yaml` + `values.yaml` 로 존재해야 한다 (Phase 3a).
- dev / prod 분기를 쓰려면 각 chart 에 `values-dev.yaml` / `values-prod.yaml` 추가 필요.

## 점검

```bash
# ApplicationSet 자체가 9 개 Application 을 생성했는지
kubectl get appset ssa1004-portfolio -n argocd
kubectl get applications -n argocd -l app.kubernetes.io/part-of=ssa1004-portfolio

# 각 Application sync 상태
argocd app list --selector app.kubernetes.io/part-of=ssa1004-portfolio

# 특정 service drift 확인
argocd app diff auth-service
```
