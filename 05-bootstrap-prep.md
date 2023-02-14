# クラスターのブートストラップの準備

[ハブ・スポークネットワークがプロビジョニングされた](./04-networking.md)後、次のステップは、AKSベースラインリファレンス実装の[AKSベースラインリファレンス実装](./)で、AKSクラスターがブートストラップされるべきものを準備することです。

## 期待される結果

コンテナレジストリは、単一のクラスターのライフサイクルを超えて延長されることがよくあります。組織レベルまたはビジネスユニットレベルで広くスコープされることができますが、通常は特定のクラスターインスタンスに直接関連付けられているわけではありません。たとえば、青/緑のクラスターインスタンスのデプロイを行い、同じコンテナレジストリを使用することができます。クラスターが消えてもレジストリはそのままです。

* Azureコンテナレジストリ（ACR）がデプロイされ、プライベートエンドポイントとして公開されます。
* クラスターのブートストラッププロセスに必要な画像がACRに配置されます。
* Log Analyticsがデプロイされ、ACRプラットフォームのログが構成されます。このワークスペースは、クラスターでも使用されます。

事前に存在するACRインスタンスの役割は、クラスターのブートストラップを考えるとより顕著になります。これは、Azureリソースのデプロイ後に発生するプロセスで、最初のワークロードがクラスターに到着する前のプロセスです。クラスターはリソースのデプロイ後 _すぐに自動的に_ ブートストラップされます。つまり、ブートストラップに使用される必要がある画像とヘルムチャートを、公式のOCIアーティファクトリポジトリとしてACRを使用して提供する必要があるということです。

### 方法

Flux GitOpsエージェントをAKS拡張としてインストールすることで、このクラスターをブートストラップします。この特定の選択肢は、Flux、またはGitOps全般がブートストラップの唯一のアプローチであることを意味するものではありません。このようなツールの組織的な知識と受容度を考慮し、クラスターのブートストラップがGitOpsまたはデプロイメントパイプラインを介して実行されるべきかを決定します。クラスターのフリートを実行している場合は、一貫性と簡単なガバナンスのために、GitOpsアプローチが強く推奨されます。クラスターを数個しか実行していない場合は、GitOpsは「多すぎる」と見なされ、ブートストラッププロセスを1つ以上のデプロイメントパイプラインに統合して、ブートストラップが実行されるようにすることができます。どのような方法でも、クラスターのデプロイを開始する前にブートストラップアーティファクトを準備しておく必要があります。Flux AKS拡張を使用すると、クラスターがすでにブートストラップされて開始され、今後も堅牢な管理基盤を構築できます。

## ステップ

1. AKS クラスターリソースグループを作成する

   > :book: アプリケーションID：0008のアプリを作成しているアプリチームは、ビジネスユニット0001（BU001）の代理でAKSクラスターを作成しようとしています。組織のネットワーキングチームと協力して、クラスターとネットワークに対応した外部リソース（たとえばApplication Gateway）をレイアウトするために、スポークネットワークが割り当てられました。彼らはその情報を取得し、[`acr-stamp.json`](./acr-stamp.json)、[`cluster-stamp.json`](./cluster-stamp.json)、および[`azuredeploy.parameters.prod.json`](./azuredeploy.parameters.prod.json)ファイルに追加しました。
   >
   > これらのファイルを使用して、アプリケーションの親グループとしてこのリソースグループを作成します。

   ```bash
   # [This takes less than one minute.]
   az group create --name rg-bu0001a0008 --location japaneast
   ```

2. Get the AKS cluster spoke virtual network resource ID.

   > :book: The app team will be deploying to a spoke virtual network, that was already provisioned by the network team.

   ```bash
   export RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   echo RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE: $RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE
   ```

2. AKS クラスターのスポーク仮想ネットワークリソースIDを得る

   > :book: アプリチームは、ネットワークチームによってすでにプロビジョニングされたスポーク仮想ネットワークにデプロイします。

   ```bash
   export RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   echo RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE: $RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE
   ```

3. コンテナレジストリテンプレートをデプロイする

   ```bash
   # [This takes about four minutes.]
   az deployment group create -g rg-bu0001a0008 -f acr-stamp.bicep -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE} location=japaneast
   ```

4. コンテナレジストリにクラスター管理用のイメージをインポートする

   > 公開コンテナレジストリは、アウトアウトやリクエストのスロットリングなどの故障によって影響を受ける可能性があります。これらの中断は、今すぐイメージを取得する必要があるシステムにとって致命的なものになる可能性があります。公開レジストリを使用するリスクを最小限に抑えるために、Azure Container Registryのようなコントロールできるレジストリにすべての適用可能なコンテナイメージを格納します。

   ```bash
   # Get your ACR instance name
   export ACR_NAME_AKS_BASELINE=$(az deployment group show -g rg-bu0001a0008 -n acr-stamp --query properties.outputs.containerRegistryName.value -o tsv)
   echo ACR_NAME_AKS_BASELINE: $ACR_NAME_AKS_BASELINE

   # Import core image(s) hosted in public container registries to be used during bootstrapping
   az acr import --source ghcr.io/kubereboot/kured:1.12.0 -n $ACR_NAME_AKS_BASELINE
   ```

   > このウォークスルーでは、ブートストラッププロセスに含まれるイメージは1つだけです。これはこのプロセスの参考として含まれています。Kubernetes Reboot Daemon（Kured）やその他のイメージ、およびハンドルチャートをブートストラップの一部として使用するかどうかは、あなた次第です。

5. ACRインスタンスからプルするためにブートストラップマニフェストを更新する。_任意の操作。フォークが必要です。_

   > クラスターは、適用されるブートストラップ構成により、すぐに[`cluster-manifests/`](./cluster-manifests/)内のマニフェストの処理を開始します。したがって、クラスターをデプロイする前に、次の変更をフォークにプッシュして、オリジナルのmspnpリポジトリのファイルではなく、公開コンテナレジストリを指すファイルを使用するようにしてください。
   >
   > * [`kured.yaml`](./cluster-manifests/cluster-baseline-settings/kured.yaml)の1つの`image:`値を、公開コンテナレジストリの代わりにコンテナレジストリを使用するように更新します。ファイル内のコメントに指示があります（または、次のコマンドを実行するだけです）。

   :warning: これらのファイルを更新せずに、自分のフォークを使用しない場合、クラスターをデプロイすると、公開コンテナレジストリに依存するようになります。これは一般に探索/テストには問題ありませんが、本番環境には適していません。本番環境に移行する前に、クラスターに持ち込むすべてのイメージ参照が、_あなた自身の_コンテナレジストリ（前のステップでインポートしたリンク）またはあなたが信頼できるものから来ていることを確認してください。

   ```bash
   sed -i '' 's:ghcr.io:'"${ACR_NAME_AKS_BASELINE}"'.azurecr.io:g' ./cluster-manifests/cluster-baseline-settings/kured.yaml
   ```

   これで、リポジトリに変更をコミットします。

   ```bash
   git commit -a -m "Update image source to use my ACR instance instead of a public container registry."
   git push
   ```
```

### 進捗を保存する

```bash
# saveenv.shスクリプトをいつでも実行して、上記で作成された環境変数を aks_baseline.env に保存します。
./saveenv.sh

# ターミナルセッションがリセットされた場合は、ファイルをソース化して環境変数を再読み込みできます。
# source aks_baseline.env
```

### ネクストステップ

:arrow_forward: [AKSクラスターをデプロイする](./06-aks-cluster.md)