# TL;DR

- helm を利用して、Kubernetes に Kong Gateway をデプロイする
- 前提
  - AdminAPI を利用
  - DB モード: 有効 -> RDS や Azure などのマネージドサービス上の postgres を利用する

# ツールインストール

## Helm のインストール

- https://helm.sh/ja/

## kubens, kubectx をインストール

- https://github.com/ahmetb/kubectx#windows-installation-using-chocolatey

# Kong インストール

## Namespace の作成と切り替え

```sh
$ kubectl create namespace kong
$ kubens kong
```

## Helm で kong リポジトリとチャートを取得する

```sh
$ helm repo add kong https://charts.konghq.com
$ helm repo update
```

## 設定ファイルのコピー

```sh
$ cp ./config.example.yaml ./config.yaml
```

## データベースホストやデータベース名などの設定

- config.yaml

```yaml
env:
  database: "postgres"
  # データベースのIPアドレスもしくはホスト名を指定する
  pg_host: "YOUR_IP_ADDRESS OR HOSTNAME"
  pg_port: "5432"
  pg_database: "kong"
  pg_user: "kong"
  pg_password: "postgres"
```

```sh
$ cp ./config.example.yaml ./config.yaml
```

## AdminAPI を有効にし、Kong チャートをリリースする

```sh
$ helm install -f ./config.yaml kube-kong kong/kong
```

## クラスター内の NodePort や ClusterIP にアクセスするための Proxy を起動する

```sh
$ kubectl proxy
```

## Proxy 経由で AdminAPI 用の Service にアクセス

- http://127.0.0.1:8001/api/v1/namespaces/kong/services/kube-kong-kong-admin:8001/proxy/

```sh
$ curl http://127.0.0.1:8001/api/v1/namespaces/kong/services/kube-kong-kong-admin:8001/proxy/
```

# Admin API を実行し、サービス・ルートなどを登録

## サービスの作成

```sh
$ curl -i -X POST \
  --url http://127.0.0.1:8001/api/v1/namespaces/kong/services/kube-kong-kong-admin:8001/proxy/services/ \
  --data 'name=example-service' \
  --data 'url=http://mockbin.org'
```

- 実行できない場合は以下を試す

curl -i -X POST --url http://127.0.0.1:8001/api/v1/namespaces/kong2/services/kube-kong-kong-admin:8001/proxy/services/ --data 'name=example-service' --data 'url=http://mockbin.org'

## ルートの作成

```sh
$ curl -i -X POST \
  --url http://127.0.0.1:8001/api/v1/namespaces/kong/services/kube-kong-kong-admin:8001/proxy/services/example-service/routes \
  --data 'paths[]=/example-route'
  --data 'name=example-route'
```

- 実行できない場合は以下を試す

curl -i -X POST --url http://127.0.0.1:8001/api/v1/namespaces/kong/services/kube-kong-kong-admin:8001/proxy/services/example-service/routes --data 'paths[]=/example-route' --data 'name=example-route'

## 作成したサービスにアクセス

- IP アドレスとポート番号を取得

```sh
# Kong GatewayのIPアドレスを表示
$ kubectl get svc --namespace kong kube-kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
#ここにIPアドレスが表示される
$ kubectl get svc --namespace kong kube-kong-kong-proxy -o jsonpath='{.spec.ports[0].port}'
#ここにポート番号が表示される
```

- 取得した情報を元にアクセス
  - http://{your_ip}:{port}/example-route

# その他

## Helm チャートや設定ファイルの更新を反映する

```sh
$ helm upgrade -f ./config.yaml kube-kong kong/kong
```
