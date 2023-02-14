# ハブ-スポーク ネットワークトポロジーの展開


[AKS ベースライン クラスター](./) の前提条件は、前のステップで[ Azure AD グループとユーザーの作業](./03-aad.md)が完了したので、これで最初の Azure リソース展開、ネットワーク リソースに入ります。


## サブスクリプションとリソース グループのトポロジー

このリファレンス実装は、単一のサブスクリプション内の複数のリソース グループに分割されています。これは、多くの組織が特定の責任を専門化したサブスクリプション (例: リージョン ハブ/ vwan は _Connectivity_ サブスクリプション、ワークロードはランディング ゾーン サブスクリプション) に分割することを反映するためです。このリファレンス実装を単一のサブスクリプション内で探索することを期待していますが、組織でこのクラスターを実装するときは、ここで学んだことを適用して、期待するサブスクリプションとリソース グループのトポロジー (例: [Cloud Adoption Framework によって提供されるもの](https://learn.microsoft.com/azure/cloud-adoption-framework/decision-guides/subscriptions/) ) に従う必要があります。単一のサブスクリプション、複数のリソース グループモデルは、デモンストレーションのための単純化のためにのみ使用されます。

## 期待される結果

### リソース グループ

次の 2 つのリソース グループが、以下のステップでネットワーク リソースで作成され、埋められます。

| 名前                            | 目的                                   |
|---------------------------------|-------------------------------------------|
| rg-enterprise-networking-hubs   | すべての組織のリージョン ハブを含みます。リージョン ハブには、出口ファイアウォールとネットワーク ログ記録のための Log Analytics が含まれます。 |
| rg-enterprise-networking-spokes | すべての組織のリージョン スポークと関連するネットワーク リソースを含みます。すべてのスポークは、リージョン ハブに接続し、サブネットはハブのリージョン ファイアウォールを通じて外部接続します。 |

### Resources

* Regional Azure Firewall in hub virtual network
* Network spoke for the cluster
* Network peering from the spoke to the hub
* Force tunnel UDR for cluster subnets to the hub
* Network Security Groups for all subnets that support them

### リソース

* ハブ仮想ネットワークのリージョン ファイアウォール
* クラスターのネットワーク スポーク
* ハブからスポークへのネットワーク ピアリング
* クラスター サブネットの強制トンネル UDR にハブを使用
* サポートするすべてのサブネットのネットワーク セキュリティ グループ

## Steps

## ステップ

1. デプロイする Azure サブスクリプションにログインします。

   > :book: ネットワーク チームは、リージョン ハブを含む Azure サブスクリプションにログインします。Contoso Bicycle では、すべてのリージョン ハブは、同じ中央で管理されるサブスクリプションにあります。

   ```bash
   az login -t $TENANTID_AZURERBAC_AKS_BASELINE
   ```

2. ネットワーク ハブのリソース グループを作成します。

   > :book: ネットワーク チームは、次のリソース グループにすべてのリージョン ハブを配置しています。グループのデフォルトの場所は、リソースの場所に関連付けられていないため、関係ありません。 (このリソース グループはすでに存在しているはずです。)

   ```bash
   # [This takes less than one minute to run.]
   az group create -n rg-enterprise-networking-hubs -l centralus
   ```

3. ネットワークスポークリソースグループを作成します。

   > :book: ネットワーク チームは、すべてのスポークを中央で管理されるリソース グループにも保持します。ハブのリソース グループと同様に、このグループの場所は関係ありません。 (このリソース グループはすでに存在しているはずです。)

   ```bash
   # [This takes less than one minute to run.]
   az group create -n rg-enterprise-networking-spokes -l centralus
   ```

4. リージョン ハブを作成します

  > :book: ネットワーク チームは、eastus2 のリージョン ハブを作成したとき、まだ定義されているスポークはありませんでしたが、ネットワーク チームは、常に標準のパターン ( `hub-default.bicep` で定義されています) に従ってベース ハブを配置します。ハブには、Azure Firewall (いくつかの組織全体のポリシー)、Azure Bastion、VPN 接続のためのゲートウェイ サブネット、およびネットワークの観測性のための Azure Monitor が含まれます。Microsoft が推奨するサブネットのサイジングに従います。
  >
  > ネットワーク チームは、 `10.200.[0-9].0` が組織のネットワーク スペースですべてのリージョン ハブが配置される場所になることを決めました。以下で作成される `eastus2` ハブは `
  >
  > 注: Azure Bastion とオンプレミス接続のためのサブネットは、このリファレンス アーキテクチャでデプロイされますが、リソースはデプロイされません。このリファレンス実装は、既存のインフラストラクチャと分離されてデプロイされることを想定しているため、これらの IP アドレスは、既存のネットワークと競合しないはずです。これらの IP アドレスが重複している場合でも。既存のネットワークに接続する必要がある場合は、リファレンス ARM テンプレートに従って、要件に応じて IP スペースを調整する必要があります。

   ```bash
   # [６分ほどかかります]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-default.bicep -p location=eastus2
   ```

   ハブ作成は、次のものを出力します。

   * `hubVnetId` - これは、接続されたリージョン スポークを作成するときに、将来のステップで照会するものです。例: `/subscriptions/[id]/resourceGroups/rg-enterprise-networking-hubs/providers/Microsoft.Network/virtualNetworks/vnet-eastus2-hub`

1. AKS クラスターとその隣接リソースを配置するネットワーク スポークを作成します。

   > :book: ネットワーク チームは、ビジネス ユニット (BU) 0001 のアプリ チームから、AKS ベースの新しいアプリケーション (内部的には Application ID: A0008 として知られています) を配置するためのネットワーク スポークのリクエストを受けます。ネットワーク チームは、アプリ チームと話し合い、要件を理解し、一般的な AKS クラスターのデプロイメントの Microsoft のベスト プラクティスに合わせます。これらの特定の要件をキャプチャし、それらの仕様に合わせてスポークをデプロイし、マッチングするリージョン ハブに接続します。

   ```bash
   RESOURCEID_VNET_HUB=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-default --query properties.outputs.hubVnetId.value -o tsv)
   echo RESOURCEID_VNET_HUB: $RESOURCEID_VNET_HUB

   # [これは約4分かかります。]
   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0008.bicep -p location=eastus2 hubVnetResourceId="${RESOURCEID_VNET_HUB}"
   ```

   スポーク作成は、次のものを出力します。

   * `appGwPublicIpAddress` - ワークロードのトラフィックを受信する Azure Application Gateway (WAF) のパブリック IP アドレス。
   * `clusterVnetResourceId` - クラスター、App Gateway、および関連リソースがデプロイされる仮想ネットワークのリソース ID。例: `/subscriptions/[id]/resourceGroups/rg-enterprise-networking-spokes/providers/Microsoft.Network/virtualNetworks/vnet-spoke-BU0001A0008-00`
   * `nodepoolSubnetResourceIds` - スポーク内の AKS ノード プールのサブネット リソース ID を含む配列。例: `[ "/subscriptions/[id]/resourceGroups/rg-enterprise-networking-spokes/providers/Microsoft.Network/virtualNetworks/vnet-hub-spoke-BU0001A0008-00/subnets/snet-clusternodes" ]`

2. スポークの要件を考慮して、共有されたリージョン ハブのデプロイメントを更新します。

   > :book: リージョン ハブに最初のスポークがあるため、ハブはもはや一般的なハブ テンプレートから実行できなくなります。ネットワーク チームは、この特定のハブとこの特定のハブが必要とする機能を永久に表す名前付きのハブ テンプレート (例: `hub-eastus2.bicep`) を作成します。新しいスポークがアタッチされ、リージョン ハブの新しい要件が発生すると、このテンプレート ファイルに追加されます。

   ```bash
   RESOURCEID_SUBNET_NODEPOOLS=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.nodepoolSubnetResourceIds.value -o json)
   echo RESOURCEID_SUBNET_NODEPOOLS: $RESOURCEID_SUBNET_NODEPOOLS

   # [これは約10分かかります。]
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-regionA.bicep -p location=eastus2 nodepoolSubnetResourceIds="${RESOURCEID_SUBNET_NODEPOOLS}"
   ```

   > :book: この時点で、ネットワーク チームは、BU 0001 のアプリ チームが AKS クラスター (ID: A0008) を配置できるスポークを提供しました。ネットワーク チームは、アプリ チームに、インフラストラクチャーのコード アーティファクトで参照するために必要な情報を提供します。
   >
   > ハブとスポークは、ネットワークチームの GitHub Actions ワークフローによって制御されます。この自動化は、この参考実装では AKS ベースラインに焦点を当てているため、ネットワーク チームの CI/CD プラクティスには含まれていません。


### 次のステップ

:arrow_forward: [クラスターのブートストラッピングの準備](./05-bootstrap-prep.md)