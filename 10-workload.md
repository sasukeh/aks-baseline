# ワークロードのデプロイ（ASP.NET Core Docker Web アプリ）

クラスターは [TLS 証明書を使用した Traefik が構成されています](./09-secret-management-and-ingress-controller.md)。プロセスの最後のステップは、ワークロードをデプロイすることです。これにより、システムの機能が示されます。

## 手順

> :book: コントソー・アプリチームはこの旅の最後のステップに近づいていますが、新しいインフラストラクチャをテストするためのアプリが必要です。このタスクには、古くから愛されてきた [ASP.NET Core Docker サンプル Web アプリ](https://github.com/dotnet/dotnet-docker/tree/main/samples/aspnetapp) を選択しました。

1. カスタマイズされたドメイン名を使用して、Ingress リソースのホスト名を変更します。 _(ドメインが contoso.com のままの場合は、このステップをスキップできます。)_

   ```bash
   sed -i "s/contoso.com/${DOMAIN_NAME_AKS_BASELINE}/" workload/aspnetapp-ingress-patch.yaml
   ```

   > macOS を使用している場合は、次のコマンドを使用する必要があることに注意してください。

   ```bash
   sed -i '' 's/contoso.com/'"${DOMAIN_NAME_AKS_BASELINE}"'/g' workload/aspnetapp-ingress-patch.yaml
   ```

2. ASP.NET Core Docker サンプル Web アプリをデプロイします

   > ワークロードの定義には、Pod Disruption Budget ルール、Ingress 設定、および参照用の Pod (反) アフィニティ ルールが含まれています。

   ```bash
   kubectl apply -k workload/
   ```

3. 処理リクエストを実行する準備ができたるまで待ちます

   ```bash
   kubectl wait -n a0008 --for=condition=ready pod --selector=app.kubernetes.io/name=aspnetapp --timeout=90s
   ```

4. Ingress リソースのステータスを確認して、AKS で管理されている Internal Load Balancer が機能していることを確認します

   > この瞬間、Ingress コントローラー (Traefik) は Ingress リソースオブジェクトの構成を読み取り、ステータスを更新し、新しい公開されたワークロードのルートを満たすルーターを作成しています。これを見てみて、構成されたサブネットから Internal Load Balancer IP が設定されていることに注意してください。

   ```bash
   kubectl get ingress aspnetapp-ingress -n a0008
   ```

   > この時点で、ワークロードへのルートが確立され、SSL オフロードが構成され、Traefik だけがワークロードに接続できるようにネットワークポリシーが設定され、Traefik が App Gateway からのリクエストのみを受け入れるように構成されています。

5. 許可されていないネットワークロケーションからワークロードに直接アクセスするテストを実行します。 _オプションです。_


   > App Gateway を介さずに接続しようとすると、Ingress コントローラーから `403` HTTP レスポンスを受け取るはずです。同様に、Ingress コントローラー以外のワークロードがワークロードに到達しようとすると、ネットワークポリシーによってトラフィックが拒否されます。

   ```bash
   kubectl run curl -n a0008 -i --tty --rm --image=mcr.microsoft.com/azure-cli --overrides='[{"op":"add","path":"/spec/containers/0/resources","value":{"limits":{"cpu":"200m","memory":"128Mi"}}},{"op":"add","path":"/spec/containers/0/securityContext","value":{"readOnlyRootFilesystem": true}}]' --override-type json  --env="DOMAIN_NAME=${DOMAIN_NAME_AKS_BASELINE}"

   # クラスター内のコンテナで実行中のオープンシェルから次のコマンドを実行します
   curl -kI https://bu0001a0008-00.aks-ingress.$DOMAIN_NAME -w '%{remote_ip}\n'
   exit
   ```

   > このコンテナ シェルから、`curl -I http://<aspnetapp-service-cluster-ip>` を介してワークロードに直接アクセスすることもできます。`200 OK` を取得する代わりに、[`allow-only-ingress-to-workload` ネットワーク ポリシー](./cluster-manifests/a0008/ingress-network-policy.yaml) が適用されているため、ネットワーク タイムアウトが発生します。

### 次のステップ

:arrow_forward: [エンド・ツー・エンドの検証](./11-validation.md)
