# Kubernetes Lab - 実行ログ (Runbook)

このドキュメントは、Kubernetes HAクラスター構築の実行内容を記録したものです。後から振り返るための学習資料としても活用できるように、各手順の目的を解説しています。

## フェーズ1: 高可用性（HA）クラスターの構築

Kubernetesの「コントロールプレーン（マスターノード）」を複数用意することで、1台がダウンしてもシステム全体が停止しない「高可用性（High Availability）」な環境を構築しました。

### ステップ 1.1: ロードバランサー（lb）の構築
`lb` ノードに HAProxy をインストールし、APIサーバーへの通信を `master-1` と `master-2` に振り分ける（ロードバランシングする）設定を行いました。

**実行コマンド (on `lb`):**
```bash
# HAProxyのインストール
sudo apt-get update
sudo apt-get install -y haproxy

# 設定の追記
# 6443番ポート（K8s APIサーバーの標準ポート）宛ての通信を、
# ラウンドロビン（順番）で master-1 と master-2 に転送します。
# tcp-check により、死活監視を行ってダウンしたノードを自動で切り離します。
cat <<EOF | sudo tee -a /etc/haproxy/haproxy.cfg

# Kubernetes API Server Load Balancer
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master-1 192.168.56.11:6443 check fall 3 rise 2
    server master-2 192.168.56.12:6443 check fall 3 rise 2
EOF

# 設定を反映してサービスを再起動
sudo systemctl restart haproxy
```

---

### ステップ 1.2: 最初のコントロールプレーン初期化
`master-1` で `kubeadm init` を実行し、クラスターの「核」を立ち上げました。

**実行コマンド (on `master-1`):**
```bash
# 【オプションの解説】
# --control-plane-endpoint: 
#   ワーカーノードなどがAPIサーバーにアクセスする際の「窓口」となるIPです。
#   今回はロードバランサー（lb）のIPを指定しています。
# --upload-certs: 
#   他のマスターノード（master-2）を追加する際に必要な証明書を自動で共有します。
# --pod-network-cidr: 
#   コンテナ（Pod）に割り当てるIPアドレスの範囲です。後で入れるCalicoの要件です。
# --apiserver-advertise-address: 
#   このノード自身がAPIサーバーとして通信を待ち受けるIPアドレスです。
#   Vagrant環境でNAT IP(10.0.x.x)が誤認識されるのを防ぐために指定します。

sudo kubeadm init \
  --control-plane-endpoint "192.168.56.10:6443" \
  --upload-certs \
  --pod-network-cidr="192.168.0.0/16" \
  --apiserver-advertise-address="192.168.56.11"

# kubectlコマンドを使用するための権限設定
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### ステップ 1.3: CNI (Calico) の導入
初期化直後は、ノード間やPod間で通信するためのネットワーク機能が存在しません。そのため、CNI（コンテナ・ネットワーク・インターフェース）と呼ばれるプラグインである「Calico」を導入しました。

**実行コマンド (on `master-1`):**
```bash
# Calicoの公式マニフェスト（YAMLファイル）を直接クラスターに適用します。
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
```

---

### ステップ 1.4: マスターノードの追加（HA構成の完成）
クラスターの頭脳を冗長化するため、`master-2` を2台目のコントロールプレーンとして参加させました。

**実行コマンド (on `master-2`):**
```bash
# --control-plane オプションが付いているため、単なる作業員ではなく
# 管理者（マスター）として参加します。
sudo kubeadm join 192.168.56.10:6443 \
        --token 4bnwh5.ygh4keb1yij0qn6f \
        --discovery-token-ca-cert-hash sha256:909b63fd4851d61ba52859c19fbba65091a4c670fc3c8249da2bb25a50212722 \
        --control-plane \
        --certificate-key df93f57702a0c6195dcaad6bb37fd406067b2b5616858035e0a02360468d427b \
        --apiserver-advertise-address="192.168.56.12"
```

---

### ステップ 1.5: ワーカーノードの追加
実際にアプリケーション（Pod）が起動して稼働する場所である `worker-1` を参加させました。

**実行コマンド (on `worker-1`):**
```bash
# --control-plane がないため、通常のワーカーノードとして参加します。
sudo kubeadm join 192.168.56.10:6443 \
        --token 4bnwh5.ygh4keb1yij0qn6f \
        --discovery-token-ca-cert-hash sha256:909b63fd4851d61ba52859c19fbba65091a4c670fc3c8249da2bb25a50212722
```

---

## フェーズ2: Kubernetesの基本操作とワークロード管理

### ステップ 2.1: Pod と Deployment の作成
Kubernetesでアプリケーションを動かす基本単位であるPodと、それを管理するDeploymentを作成し、スケールアウト（負荷分散のための増強）を体験しました。

**実行コマンド (on `master-1`):**
```bash
# Nginxのコンテナを2つ持つDeploymentを作成
kubectl create deployment my-nginx --image=nginx:latest --replicas=2

# Podの動作状況と配置されているノードを確認
kubectl get pods -o wide

# レプリカ数を4つに増やす（スケールアウト）
kubectl scale deployment my-nginx --replicas=4
```

### ステップ 2.2: Service を使ったネットワークへの公開
Deploymentで増やしたPod群に対して、共通のアクセス窓口（Service）を作成しました。今回は `NodePort` というタイプを使用し、各ノード（VM）の特定のポートを開放して外部（Windowsホスト）からアクセスできるようにしました。

**実行コマンド (on `master-1`):**
```bash
# 80番ポートで動いているNginxを、NodePortを使って外部公開する
kubectl expose deployment my-nginx --port=80 --type=NodePort

# 作成されたServiceの確認（割り当てられた30000番台のポート番号を確認）
kubectl get svc
```
*(Windowsホストのブラウザから `http://192.168.56.x:30000番台` にアクセスし、Nginxの画面が表示されることを確認済)*

### ステップ 2.3: トラブルシューティングの基礎
エラーが発生した際の原因調査として、Podの詳細情報を見る `describe` コマンドと、アプリケーションの出力を見る `logs` コマンドの使い方を学びました。

**実行コマンド (on `master-1`):**
```bash
# 存在しないイメージを指定して、意図的にエラーになるPodを作成
kubectl run fail-pod --image=nginx:typo

# Podのステータスが ImagePullBackOff / ErrImagePull になることを確認
kubectl get pods

# 【重要】Podの詳細なエラー理由（Events）を確認する
kubectl describe pod fail-pod

# 【重要】正常に動いているDeploymentのアプリケーションログ（アクセスログ等）を確認する
kubectl logs deployment/my-nginx
```
