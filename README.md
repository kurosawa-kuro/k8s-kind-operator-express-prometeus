# k8s-kind-operator-express-prometeus

| Port | Description |
|------|-------------|
| 22   | ssh |
| 80   | http |
| 443  | https |
| 8000 | backend‑api |
| 8080 | argo‑cd‑host‑port‑ix |
| 9090 | prometheus‑inner‑port |
| 3000 | grafana |
| 30080 | grafana |
| 30090 | prometheus |
| 31810 | argo‑cd‑node‑port‑auto |
| 6443 | k8s‑api |

下に **「Step 8 以降」** を追記するかたちで、  
*同じ ~/dev/k8s‑kind‑operator‑express ディレクトリ内* に Helm‑based 監視スタックを組み込む手順を追加しました。  
（既存の 0 〜 7 はそのまま。番号だけ続きます）

---

## 8 Helm のセットアップ

```bash
# Helm がまだ無ければインストール
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version   # v3.14.0 などと表示されれば OK
```

---

## 9 kube‑prometheus‑stack をデプロイ  
Prometheus Operator・Prometheus 本体・Grafana・Alertmanager 等を一括で入れる。

```bash
# 9‑1 リポジトリ登録 & 更新
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 9‑2 監視用 Namespace を作成し、一括インストール
helm install kp-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30090 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30080 \
  --set grafana.adminPassword=admin
```

> *NodePort は空いている番号に変更しても可。  
> Grafana 初期ユーザー `admin / admin` （パスワードは上で固定）。*

### 9‑3 Pod が立ち上がるまで待機

```bash
kubectl -n monitoring get pods -w
```

---

## 10 Express Service に監視用ラベルを付与

Prometheus Operator が `ServiceMonitor` を見つけやすいよう、  
Service ( `sample-api` ) に `app=sample-api` ラベルを追加。

```bash
kubectl label svc sample-api app=sample-api --overwrite
```

（**Operator側で自動付与する場合**は `serviceFor()` 内で  
`Labels: map[string]string{"app": app.Name},` を設定しておくと良い）

---

## 11 ServiceMonitor マニフェストを追加

`monitoring/servicemonitor-sample-api.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-api
  namespace: default
  labels:
    release: kp-stack           # ← Helm で入れたリリース名と合わせる
spec:
  selector:
    matchLabels:
      app: sample-api           # Service に付けたラベル
  endpoints:
    - port: http                # Service の port 名 (Type=ClusterIP 時は "http" がデフォルト)
      path: /metrics
      interval: 15s
```

適用：

```bash
kubectl apply -f monitoring/servicemonitor-sample-api.yaml
```

---

## 12 Grafana でメトリクスを確認

```bash
# 12‑1 Grafana ポートフォワード（NodePort を使わない場合）
kubectl -n monitoring port-forward svc/kp-stack-grafana 3000:30080
# → http://localhost:3000 でログイン (admin / admin)

# 12‑2 Prometheus データソースは既に登録済み
# 12‑3 「+ Import」から 11074 (Node.js dashboard) などをインポート
#      Legend / query を app="sample-api" に合わせると Express のメトリクスが可視化できる
```

---

## 13 (任意) Prometheus UI へ直接アクセス

```bash
kubectl -n monitoring port-forward svc/kp-stack-prometheus 9090:30090
# → http://localhost:9090 で "up{app=\"sample-api\"}" などを実行
```

---

## 14 まとめ – ここまでで出来上がる理想状態

* kind クラスタ上で ExpressApp (replicas=1+) が稼働  
* `/metrics` を **ServiceMonitor** が拾い、Prometheus がスクレイプ  
* Grafana でリアルタイムに可視化・アラート設定可能  
* すべて **~/dev/k8s-kind-operator-express** 配下 & `kubectl apply` 冪等運用  

これで **Operator + Helm + Prometheus/Grafana** の最小観測スタックが完成です。  
あとは **Loki スタック**（ログ用）や **Alertmanager → Slack** など、同じ Helm 流儀で追加していけば OK。  

不明点やエラーがあれば気軽にどうぞ！
