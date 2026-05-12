# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

このリポジトリは **MiroTalk P2P ビデオ会議アプリ** の Kubernetes マニフェストリポジトリです。
homelab-gitops (`ROBO358/homelab-gitops`) の `apps/` レイヤから Flux で管理されます。

- GitHub リポジトリ: `ROBO358/mirotalk-k8s`
- コンテナイメージ: `mirotalk/p2p`（Docker Hub 公式イメージ。バージョンタグなし）
- デプロイ先 namespace: `mirotalk`（namespace 自体は homelab-gitops 側で管理）

## リポジトリ構成

```
manifests/         # Kubernetes マニフェスト（Flux が直接 apply する）
  kustomization.yaml
  deployment.yaml
  service.yaml
  gateway.yaml
  httproute.yaml
  httproute-tunnel.yaml
  certificate.yaml
  ciliumnetworkpolicy.yaml
```

**このリポジトリに CI はない**。コンテナイメージは Docker Hub 公式の `mirotalk/p2p` を使用。
`manifests/` を変更すると Flux が自動で apply する（ポーリング間隔 1m）。

## イメージ更新手順

mirotalk/p2p は versioned タグを公開していないため、image digest で固定する。

```bash
# Docker Hub から最新の multi-arch manifest digest を取得
curl -s "https://hub.docker.com/v2/repositories/mirotalk/p2p/tags/latest" | jq -r '.digest'
```

取得した digest を `manifests/deployment.yaml` の image フィールドに反映してコミットする:

```yaml
image: mirotalk/p2p@sha256:<new-digest>
```

## manifests/ の各リソース

| ファイル | 内容 |
|---|---|
| `deployment.yaml` | mirotalk P2P コンテナ（port 3000）。strategy: Recreate（WebRTC はステートフル） |
| `service.yaml` | port 80 → 3000 |
| `gateway.yaml` | GatewayClass: cilium、LB IP: `192.168.1.104`、TLS 終端 (mirotalk-tls) |
| `httproute.yaml` | `mirotalk.yh.k8s.tsuru.run` → mirotalk-gateway（LAN アクセス用） |
| `httproute-tunnel.yaml` | `mirotalk-yh-k8s.tsuru.run` → cloudflare-gateway/yh-cluster（Cloudflare Tunnel 公開用） |
| `certificate.yaml` | `mirotalk.yh.k8s.tsuru.run` を letsencrypt-prod で発行 |
| `ciliumnetworkpolicy.yaml` | ingress: intra-ns / cilium gateway / cloudflare-gateway のみ許可。egress: kube-dns + world（WebRTC STUN/TURN） |

## homelab-gitops との責任分界

| 責務 | 配置先 |
|---|---|
| Namespace, PodSecurity | homelab-gitops `apps/mirotalk/namespace.yaml` |
| RBAC (SA / ClusterRoleBinding) | homelab-gitops `apps/mirotalk/rbac.yaml` |
| ReferenceGrant（cloudflare-gateway 参照許可） | homelab-gitops `apps/mirotalk/referencegrant.yaml` |
| GitRepository / Flux Kustomization 定義 | homelab-gitops `apps/mirotalk/source.yaml` |
| アプリ workload（Deployment / Service 等） | **このリポジトリ** `manifests/` |

**このリポジトリの `manifests/` に SA / ClusterRole / Binding を置いてはいけない**。RBAC monotonicity 制約により app リポ側に SA 作成権を委譲すると tenant が自己昇格できるため、RBAC 境界の定義は homelab-gitops 側が保持する。

## CiliumNetworkPolicy の変更注意点

ingress ルールは 3 種が必要（Prometheus スクレイプ対象なし）:
1. intra-namespace（同 namespace 内の通信）
2. `fromEntities: ingress`（Cilium Gateway proxy からの LAN HTTPS）→ port 3000
3. `io.kubernetes.pod.namespace: cloudflare-gateway`（Cloudflare Tunnel cloudflared pods）→ port 3000

egress:
- kube-dns（UDP/TCP 53）
- `world`（WebRTC STUN/TURN サーバーへの outbound が必要）
