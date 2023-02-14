# AKS クラスターをデプロイする

[ACR インスタンスはデプロイされ、クラスターのブートストラップに対応していることを確認しました](./05-bootstrap-prep.md)。次のステップは、[AKS ベースライン リファレンス実装](./)の次のステップで、AKS クラスターとその他の Azure リソースをデプロイすることです。

## 手順

1. ブートストラップのリポジトリを特定します

   > このリポジトリをクローンした場合、値は元の mspnp GitHub 組織のリポジトリになります。これは、クラスターがパブリック コンテナー イメージを使用してブートストラップされることを意味します。代わりにこのリポジトリをフォークした場合、GitOps リポジトリはあなた自身のリポジトリになり、クラスターがコンテナー イメージ参照を使用してブートストラップされます。前の手順ページでは、そのマニフェストを使用して ACR インスタンスを使用するように更新する機会がありました。プライベート ブートストラップ リポジトリの使用についてのガイダンスについては、[プライベート ブートストラップ リポジトリ](./cluster-manifests/README.md#private-bootstrapping-repository)を参照してください。

   ```bash
   GITOPS_REPOURL=$(git config --get remote.origin.url)
   echo GITOPS_REPOURL: $GITOPS_REPOURL

   GITOPS_CURRENT_BRANCH_NAME=$(git branch --show-current)
   echo GITOPS_CURRENT_BRANCH_NAME: $GITOPS_CURRENT_BRANCH_NAME
   ```

2. クラスター ARM テンプレートをデプロイします

   :exclamation: デフォルトでは、このデプロイメントでは、クラスターの API サーバーへの制限なしのアクセスが許可されます。すべてのデプロイメント オプションで `clusterAuthorizedIPRanges` パラメーターを設定することで、API サーバーへのアクセスを一連の既知の IP アドレス (つまり、ジャンプ ボックス サブネット (Azure Bastion に接続)、ビルド エージェント、またはクラスターを管理するネットワークのいずれか) に制限できます。この設定は、API サーバーを使用しようとしてクラスター内から発生するトラフィックにも影響を与えるため、出口 Azure Firewall で使用されるすべてのパブリック IP を含める必要があります。詳細については、[API サーバーへのアクセスを認証された IP アドレス範囲を使用して安全にする](https://docs.microsoft.com/azure/aks/api-server-authorized-ip-ranges#create-an-aks-cluster-with-api-server-authorized-ip-ranges-enabled)を参照してください。

   ```bash
   # [1o 分ほどかかります]
   az deployment group create -g rg-bu0001a0008 -f cluster-stamp.bicep -p targetVnetResourceId=${RESOURCEID_VNET_CLUSTERSPOKE_AKS_BASELINE} clusterAdminAadGroupObjectId=${AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE} a0008NamespaceReaderAadGroupObjectId=${AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE} k8sControlPlaneAuthorizationTenantId=${TENANTID_K8SRBAC_AKS_BASELINE} appGatewayListenerCertificate=${APP_GATEWAY_LISTENER_CERTIFICATE_AKS_BASELINE} aksIngressControllerCertificate=${AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64_AKS_BASELINE} domainName=${DOMAIN_NAME_AKS_BASELINE} gitOpsBootstrappingRepoHttpsUrl=${GITOPS_REPOURL} gitOpsBootstrappingRepoBranch=${GITOPS_CURRENT_BRANCH_NAME} location=japaneast
   ```

   > 代わりに、[`azuredeploy.parameters.prod.json`](./azuredeploy.parameters.prod.json) ファイルを更新し、上記の方法で `-p "@azuredeploy.parameters.prod.json"` を使用してデプロイできます。

## コンテナー レジストリの注意点

:warning: このクラスターのデプロイとワークロードの実験を容易にするために、Azure Policy と Azure Firewall は現在、Docker Hub などの _パブリック コンテナー レジストリ_ からイメージをプルできるように、クラスターに設定されています。本番システムでは、`cluster-stamp.bicep` ファイルの Azure Policy パラメーターである `allowedContainerImagesRegex` を、使用するコンテナー レジストリのみをリストにして、そのポリシーが適用される名前空間を指定し、Azure Firewall にも同じものを許可するように更新する必要があります。これにより、未承認のレジストリが使用されてイメージをプルしようとした場合に、SLA 保証がないレジストリからイメージをプルしようとして問題が発生するのを防ぐことができます。

このデプロイでは、クラスターのニーズに応じて SLA 対応の Azure コンテナー レジストリが作成されます。組織では、使用するための中央コンテナー レジストリがある場合があります。また、アプリケーション固有のインフラストラクチャに結びついているレジストリがある場合もあります (この実装で示されています)。**アプリケーションのセキュリティと可用性のニーズを満たすコンテナー レジストリのみを使用してください。**

## アプリケーション ゲートウェイの配置

Azure アプリケーション ゲートウェイは、このリファレンス実装では、クラスター ノードと同じ仮想ネットワークに配置されます (サブネットと関連する NSG によって分離されます)。これにより、アプリケーション ゲートウェイからクラスターのプライベート ロード バランサーに直接ネットワーク ライン オブ サイトが確立され、強力なネットワーク境界制御を維持できます。更に重要なのは、クラスターのポイント オブ イングレスを所有しているクラスター操作チームと一致していることです。一部の組織では、代わりに Application Gateway を中央で管理し、完全に分離された仮想ネットワークに存在するパーミア ネットワークを利用することもできます。このトポロジーも問題ありませんが、そのパーミア ネットワークとクラスターのプライベート ロード バランサーとの間に安全で制限されたルーティングがあることを確認する必要があります。また、クラスター/ワークロード操作チームと Application Gateway を所有するチームとの間に、追加の調整が必要になります。

## 次のステップ

:arrow_forward: [クラスターのブートストラップを検証する](./07-bootstrap-validation.md)

