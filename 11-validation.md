# エンドツーエンドの検証

ワークロードがデプロイされたので、[ASP.NET Core サンプル Web アプリ](./10-workload.md)の[AKS ベースライン クラスター](./)の参考実装を検証および探索できます。ワークロードに加えて、観測性の検証も実施できます。

## Web アプリの検証

このセクションは、ワークロードが正しく公開され、HTTP リクエストに応答していることを確認するのに役立ちます。

### Steps

### 手順

1. アプリケーション ゲートウェイのパブリック IP アドレスを取得します。

   > :book: App チームは、最終的な受入テストを実施して、エンド ツー エンドで期待どおりのトラフィックが流れていることを確認します。そのため、Azure Application Gateway エンドポイントに対してリクエストを行います。

   ```bash
   # Azure Application Gateway のパブリック IP アドレスを問い合わせます
   APPGW_PUBLIC_IP=$(az deployment group show --resource-group rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.appGwPublicIpAddress.value -o tsv)
   ```

2. DNS に `A` レコードを作成します。

   > :bulb: ローカルホストファイルの変更からシミュレートできます。特定のデプロイメントのアプリケーション ドメイン名に実際の DNS エントリを追加することもできます（アクセス権がある場合）。

   Azure Application Gateway のパブリック IP アドレスをアプリケーション ドメイン名にマッピングします。そのため、ホスト ファイル（`C:\Windows\System32\drivers\etc\hosts` または `/etc/hosts`）を編集し、次のレコードを末尾に追加します：`${APPGW_PUBLIC_IP} bicycle.${DOMAIN_NAME_AKS_BASELINE}`（例：`

3. サイトにアクセスします（例： <https://bicycle.contoso.com>）。

   > :bulb: ブラウザのアドレスバーに URL を入力するときは、プロトコル接頭辞 `https://` を忘れないでください。自己署名証明書を使用しているため、TLS 警告が表示されます。無視するか、ユーザーの信頼されたルート ストアに自己署名証明書（`appgw.pfx`）をインポートすることができます。

   Web ページを数回更新し、ページの下部に表示される値 `Host name` を観察します。Traefik Ingress コントローラーが、Web ページをホストする 2 つの Pod の間でリクエストをバランスするため、ホスト名は、クエリを実行するときに、1 つの Pod 名からもう 1 つの Pod 名に変更されます。

## a0008 名前空間へのリーダー アクセスの検証。_オプションです。_

[Azure AD セキュリティグループ](./03-aad.md) を設定したとき、a0008 名前空間の「リーダー」として使用するグループを作成しました。この RBAC の例を体験したい場合は、そのグループにユーザーを追加する必要があります。

Azure RBAC がクラスターの Kubernetes RBAC のバックストアとして使用されている場合は、これだけで十分です。

もし、Kubernetes RBAC が Azure AD に直接設定されている場合は、[Azure AD 設定ページ](./03-aad.md) の最後にある手順に従って、[`rbac.yaml`](./cluster-manifests/a0008/rbac.yaml) を更新して適用する必要があります。

どちらのバックストアを使用していても、グループに割り当てられたユーザーは、クラスターに `az aks get-credentials` を実行できるようになり、a0008 名前空間の _読み取り専用_ ビューに制限されていることを検証できます。

## Azure ポリシーの確認

[クラスターのデプロイ ステップ](./06-aks-cluster.md) で、チームのガバナンス ルールに準拠するように、クラスターに組み込みポリシーとカスタム ポリシーが適用されます。効果 [`audit`](https://docs.microsoft.com/azure/governance/policy/concepts/effects#audit) のポリシー割り当ては、アクティビティ ログに警告を作成し、ポータルの Azure ポリシー ブレードで違反を表示し、コンプライアンス ステートと違反リソースを識別するオプションを提供することで、集約されたビューを表示します。効果 [`deny`](https://docs.microsoft.com/azure/governance/policy/concepts/effects#deny) のポリシー割り当ては、[Gatekeeper のアドミッション コントローラー Webhook](https://open-policy-agent.github.io/gatekeeper/website/docs/) の助けを借りて、ポリシーに違反する API リクエストを拒否することで強制されます。

:bulb: Gatekeeper ポリシーは、[ポリシー言語 'Rego'](https://docs.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes#policy-language) で実装されています。Azure プラットフォームでこのリファレンス アーキテクチャのポリシーをデプロイするには、Rego 仕様を Base64 でエンコードし、`nested_K8sCustomIngressTlsHostsHaveDefinedDomainSuffix.bicep` で定義されている Azure ポリシー リソースのフィールドに格納します。Base64 デコーダーを使用して文字列をデコードし、宣言的な実装を調べることができるかもしれません。

### 手順2：Azure ポリシーの確認

1. 次のコマンドを使用して、ワークロード名前空間に 2 つ目の `Ingress` リソースを追加してみましょう。

   `rules` と `tls` セクションで指定されたホスト値は、[証明書を作成する](./02-ca-certificates.md) ときに定義したドメイン サフィックスではなく、`invalid-domain.com` のドメイン サフィックスを定義しています。

   ```bash
   cat <<EOF | kubectl create -f -
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: aspnetapp-ingress-violating
     namespace: a0008
   spec:
     tls:
     - hosts:
       - bu0001a0008-00.aks-ingress.invalid-domain.com
     rules:
     - host: bu0001a0008-00.aks-ingress.invalid-domain.com
       http:
         paths:
         - path: /
           pathType: Prefix
           backend:
             service:
               name: aspnetapp-service
               port:
                 number: 80
   EOF
   ```

2. エラー メッセージを調べ、`bu0001a0008-00.aks-ingress.invalid-domain.com` を不適合ホストとして拒否する Gatekeeper のアドミッション Webhook が拒否したことに気づきます。

   ```output
   Error from server (Forbidden): error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request: [azurepolicy-k8scustomingresstlshostshavede-e64871e795ce3239cd99] TLS host must have one of defined domain suffixes. Valid domain names are ["contoso.com"]; defined TLS hosts are {"bu0001a0008-00.aks-ingress.invalid-domain.com"}; incompliant hosts are {"bu0001a0008-00.aks-ingress.invalid-domain.com"}.
   ```

## Web アプリケーション ファイアウォール機能の確認

ワークロードは、意図的に悪意のあるアクティビティを防ぐために設計されたルールを備えた Web アプリケーション ファイアウォール (WAF) の後ろに配置されています。これをテストするには、悪意のあるリクエストを発行して、組み込みルールの 1 つをトリガーします。

> :bulb: このリファレンス アーキテクチャでは、**Prevention** モードで OWASP 3.0 ルールセットの組み込みルールを有効にします。

### 手順3：Web アプリケーション ファイアウォール機能の確認

1. 以下の URL に追加してサイトにブラウズします。`?sql=DELETE%20FROM` (例: <https://bicycle.contoso.com/?sql=DELETE%20FROM>)。
2. リクエストがアプリケーション ゲートウェイの WAF ルールによってブロックされ、ワークロードがこの潜在的に危険なリクエストを見ることはなかったことに気づきます。
3. 接続された Log Analytics ワークスペースには、ブロックされたリクエスト (および他のゲートウェイ データ) が表示されます。

   リソースグループ `rg-bu0001-a0008` の Application Gateway に移動し、_Logs_ ブレードに移動します。次のクエリを実行して、WAF ログを表示し、_SQL Injection Attack_ (フィールド _Message_) によってリクエストが拒否されたことを確認します。

   > :warning: アプリケーションゲートウェイから Log Analytics ワークスペースにログが転送されるまでには、数分かかる場合があります。そのため、前の手順で https リクエストを送信した後、クエリがすぐに結果を返さない場合は、少し待ってからクエリを実行してください。

   ```
   AzureDiagnostics
   | where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"
   ```

## クラスターの Azure Monitor のインサイトとログの確認

本番環境クラスターを実行している場合、クラスターの監視は重要です。そのため、AKS クラスターは、[ブートストラップ ステップ](./05-bootstrap-prep.md) でデプロイされた Log Analytics ワークスペースに、カテゴリ _cluster-autoscaler_、_kube-controller-manager_、_kube-audit-admin_ および _guard_ のクラスター診断情報を送信するように構成されています。さらに、[Azure Monitor for containers](https://learn.microsoft.com/azure/azure-monitor/insights/container-insights-overview) がクラスターに構成され、ワークロード コンテナーからメトリックとログをキャプチャします。Azure Monitor は、クラスター ログを表面化するように構成されており、ここでは、生成されるときにこれらのログを確認できます。

:bulb: Kubernetes スケジューラーの動作を調べる必要がある場合は、_kube-scheduler_ ログ カテゴリを有効にします (AKS クラスターの _Diagnostic Settings_ ブレードを介して、または `cluster-stamp.bicep` テンプレートでカテゴリを有効にすることで)。このカテゴリは非常に冗長であり、Log Analytics ワークスペースのコストに影響を与えることに注意してください。

### 手順4：クラスターの Azure Monitor のインサイトとログの確認

1. Azure Portal で AKS クラスター リソースに移動します。
2. キャプチャーされたデータを確認するには、_Insights_ をクリックします。

[queries](https://learn.microsoft.com/azure/azure-monitor/logs/log-analytics-tutorial) を実行して、[キャプチャされたクラスター ログ](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-log-query) を確認できます。

1. Azure Portal で AKS クラスター リソースに移動します。
2. _Logs_ をクリックし、ログ データを確認します。
   :bulb: _Kubernetes Services_ カテゴリには、いくつかの例があります。

## Azure Monitor for containers の検証 (Prometheus メトリック)

Azure モニターは、クラスターの [Prometheus メトリックをスクレイプ](https://learn.microsoft.com/azure/azure-monitor/insights/container-insights-prometheus-integration) するように構成されています。このリファレンス実装では、[`container-azm-ms-agentconfig.yaml`](./cluster-baseline-settings/container-azm-ms-agentconfig.yaml) で構成されているように、2 つのネームスペースから Prometheus メトリックを収集するように構成されています。Prometheus メトリックを出力するように構成されている 2 つのポッドがあります。

- [Traefik](./workload/traefik.yaml) (`a0008` namespace の中)
- [Kured](./cluster-baseline-settings/kured.yaml) ( `cluster-baseline-settings` namespace の中)

:bulb: このリファレンス実装には、Log Analytics クエリ パックに 2 つのクエリ (_All collected Prometheus information_ および _Kubenertes node reboot requested_) が含まれています。これらは、ARM テンプレートを介して自分自身を書き込み、管理する方法の例です。

### 手順5：Azure Monitor for containers の検証 (Prometheus メトリック)

1. In the Azure Portal, navigate to your AKS cluster resource group (`rg-bu0001a0008`).
1. Select your Log Analytic Workspace resource and open the _Logs_ blade.
1. Find the one of the above queries in the _Containers_ category.
1. You are able to select and execute the saved query over the scraped metrics.

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動します。
2. Log Analytics ワークスペース リソースを選択し、_Logs_ ブレードを開きます。
3. _containers_ カテゴリの上記のクエリの 1 つを見つけます。
4. 保存されたクエリを選択して、スクレイプされたメトリックに対して実行できます。

## ワークロード ログの検証

例のワークロードでは、標準の dotnet ロガー インターフェイスが使用されています。これらは、Azure Monitor の `ContainerLogs` にキャプチャされます。ワークロードには、Application Insights などの追加のロギングとテレメトリ フレームワークを含めることもできます。ここでは、組み込みのアプリケーション ログを表示する手順を示します。

### 手順6：ワークロード ログの検証

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動します。
2. Log Analytics ワークスペース リソースを選択し、_Logs_ ブレードを開きます。
3. 下記のクエリを実行します。

   ```kusto
   ContainerLogV2
   | where ContainerName == "aspnet-webapp-sample"
   | project TimeGenerated, LogMessage, Computer, ContainerName, ContainerId
   | order by TimeGenerated desc
   ```

## Azure アラートの検証

Azure は、クラスターと隣接するリソースの状態に関するアラートを生成します。このリファレンス実装では、複数のアラートを設定して、サブスクライブできるようにします。

### 手順7：Azure アラートの検証

[Azure モニター for containers による Kusto クエリを使用した情報に基づくアラート](https://docs.microsoft.com/ja-jp/azure/azure-monitor/insights/container-insights-alerts) がこのリファレンス実装で構成されています。

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動します。
2. _Alerts_ を選択し、_Alert Rules_ を選択します。
3. "[your cluster name] Scheduled Query for Pod Failed Alert" というタイトルのアラートがあり、カスタム クエリの応答に基づいてトリガーされます。

[Azure Advisor アラート](https://docs.microsoft.com/ja-jp/azure/advisor/advisor-alerts) がこのリファレンス実装で構成されています。

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動します。
2. _Alerts_ を選択し、_Alert Rules_ を選択します。
3. "AllAzireAdvisorAlert" というタイトルのアラートがあり、新しい Azure Advisor アラートに基づいてトリガーされます。

メトリックアラートもこのリファレンス実装で構成されています。

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動します。
2. クラスターを選択し、_Insights_ を選択します。
3. _Recommended alerts_ を選択して、有効になっているアラートを確認します。 (必要に応じて有効化/無効化を行ってください。)

## Azure Container Registry イメージのプルの検証

サードパーティーイメージを Azure Container Registry からプルするように構成した場合、flux 構成を適用したときに、コンテナ レジストリ ログにクラスターの `Pull` ログが表示されることを確認できます。

### 手順8：Azure Container Registry イメージのプルの検証

1. Azure Portal で AKS クラスター リソースグループ (`rg-bu0001a0008`) に移動し、Azure Container Registry インスタンス (`acraks` で始まる) を選択します。
2. _Logs_ を選択します。
3. 以下のクエリを実行し、適切な時間範囲を選択します。

   ```kusto
   ContainerRegistryRepositoryEvents
   | where OperationName == 'Pull'
   ```

4. kured のログが表示されます。ReplicaSet/DaemonSet の配置を満たすために、複数のノードにイメージがプルされたため、いくつかのログが複数表示されます。

## 次のステップ

:arrow_forward: [Azure Resources のクリーンアップ](./12-cleanup.md)
