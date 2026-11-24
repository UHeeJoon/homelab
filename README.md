# homelab

홈랩 쿠버네티스 클러스터의 플랫폼 구성을 관리하는 GitOps 저장소입니다.
Argo CD가 이 저장소의 `release` 브랜치를 읽어서 클러스터에 배포합니다.

애플리케이션(프론트엔드, 백엔드 등)의 배포 매니페스트는 여기가 아니라
GitLab [`sagopanda/sapari-helm-manifest`](https://gitlab.com/sagopanda/sapari-helm-manifest)에 있습니다.

## 배포 흐름

```
main에 push → GitHub Actions (비밀 값 검사, YAML 검사) → 통과하면 release로 승격 → Argo CD가 반영
```

main에 push한다고 바로 배포되지 않습니다. 검사를 통과한 커밋만 `release`로 올라갑니다.

## 디렉터리

| 경로 | 내용 |
|---|---|
| `apps/<분류>/<이름>.yaml` | Argo CD Application. 어떤 차트나 경로를 어느 네임스페이스에 배포할지 정의 |
| `components/<분류>/<이름>/` | 실제 배포 내용. Helm values, CR, 매니페스트 |
| `bootstrap/` | Argo CD가 관리하지 않는 것. 클러스터를 처음 만들 때 직접 설치 |

`root` App이 `apps/` 아래를 전부 읽어 각 Application을 만듭니다.
Argo CD 화면에서는 AppProject(분류별 1개)로 묶어서 봅니다.

| 분류 | 구성 요소 |
|---|---|
| platform | namespaces, rbac, network-policies, storage-classes, argocd-projects, keycloak |
| operators | longhorn, external-secrets, vault, cloudnative-pg, strimzi, eck, minio-operator |
| secrets | cluster-secret-store, 네임스페이스별 ExternalSecret |
| data | minio, postgresql, mongodb, redis, kafka, elasticsearch |
| observability | kube-prometheus-stack, loki, alloy, tempo, kibana |
| networking | gateway, httproutes, lb-ipam |

차트 버전은 각 `apps/*.yaml`의 `targetRevision`을 보면 됩니다.

## 클러스터

| 항목 | 값 |
|---|---|
| 노드 | `kube-master` 192.168.123.105 (control-plane), `kube-worker` 192.168.123.100 (둘 다 고정 IP) |
| 네트워크 | Cilium (kube-proxy 대체, LB-IPAM, Gateway API) |
| LoadBalancer IP | 192.168.123.200-210 |
| 외부 진입 | `public-gateway` .201, `admin-gateway` .202 → Cloudflare Tunnel → `*.ascode.click` |
| 스토리지 | Longhorn (`longhorn-prod` 기본) |
| 비밀 값 | Vault에 저장하고 External Secrets로 주입. Git에는 참조만 있음 |

## 처음 설치할 때

```bash
helm repo add cilium https://helm.cilium.io
helm repo add argo https://argoproj.github.io/argo-helm

# 1. CNI
helm upgrade --install cilium cilium/cilium -n kube-system \
  --version 1.19.4 -f bootstrap/cilium-values.yaml

# 2. Argo CD
helm install argocd argo/argo-cd -n cicd --create-namespace \
  --version 9.5.16 -f bootstrap/argocd-values.yaml

# 3. root App
kubectl apply -f bootstrap/root-app.yaml

# 4. Vault unseal (Vault가 뜬 뒤)
kubectl -n security exec -it vault-0 -- vault operator unseal
```

4번 전까지는 Vault를 쓰는 App들이 실패와 재시도를 반복하는데, 정상입니다. unseal하면 알아서 맞춰집니다.
GitLab 애플리케이션 저장소의 읽기 자격도 이때 `cicd-secrets` App이 Vault(`secret/cicd/gitlab`)에서 만들어 줍니다.

## 운영 메모

- 노드를 재부팅하면 Vault가 잠깁니다. unseal을 다시 해야 합니다.
- Cilium과 Argo CD는 Argo CD가 관리하지 않습니다. 버전을 올릴 때는 `bootstrap/`의 values로 직접 `helm upgrade` 합니다.
- 새 구성 요소를 추가할 때는 `apps/<분류>/`에 Application을, `components/<분류>/`에 내용을 넣습니다.
