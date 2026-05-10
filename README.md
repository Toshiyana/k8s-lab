# Kubernetes Lab Environment

このリポジトリは、VagrantとVirtualBoxを使用したKubernetes（K8s）のマルチマスター（HA）クラスター学習用ラボ環境です。

## クラスター構成

| ノード名 | IPアドレス | CPU | メモリ | 役割 |
| :--- | :--- | :--- | :--- | :--- |
| `lb` | 192.168.56.10 | 1 | 512MB | ロードバランサー (APIサーバーへのトラフィック分散) |
| `master-1` | 192.168.56.11 | 2 | 2048MB | コントロールプレーン 1 |
| `master-2` | 192.168.56.12 | 2 | 2048MB | コントロールプレーン 2 |
| `worker-1` | 192.168.56.21 | 2 | 2048MB | ワーカーノード |

- **OS**: Ubuntu 22.04 (Jammy)
- **コンテナランタイム**: containerd
- **K8sバージョン**: v1.29

## 学習プラン

### フェーズ1: 高可用性（HA）クラスターの構築とネットワーク設定
1. **ロードバランサー（lb）の構築**: HAProxyを導入し、`master-1`と`master-2`のAPIサーバー(6443)へトラフィックを分散。
2. **コントロールプレーンの初期化**: `master-1` で `--control-plane-endpoint 192.168.56.10` を指定して `kubeadm init` を実行。
3. **CNIの導入**: Calico等のネットワークプラグインをインストール。
4. **コントロールプレーンの拡張**: `master-2` をコントロールプレーンとして参加させる。
5. **ワーカーノードの追加**: `worker-1` をクラスターに参加させる。

### フェーズ2: 基本操作とワークロード管理
- Pod, Deployment, Service (ClusterIP, NodePort) の作成と運用。

### フェーズ3: 設定と永続データ
- ConfigMap, Secret の利用。
- PV (PersistentVolume) / PVC (PersistentVolumeClaim) によるデータ永続化。

### フェーズ4: 高度なルーティングと運用管理
- NGINX Ingress Controllerの導入。
- Cordon / Drain を用いたノードの安全なメンテナンス。

### フェーズ5: HA構成の検証（カオスエンジニアリング）
- `master-1` を意図的に停止させ、`lb` 経由でAPIサーバーへ継続してアクセスできるか検証。
- ワーカーノードの障害シミュレーションと自己修復(Self-Healing)の確認。

## 使い方

```bash
# 全ノードの起動と初期設定(setup.sh)の実行
vagrant up

# 各ノードへのログイン
vagrant ssh <ノード名>
```
