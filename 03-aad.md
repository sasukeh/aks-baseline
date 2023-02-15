# Azure Active Directory の統合の準備

前のステップで、[ユーザー向けの TLS 証明書を生成](./02-ca-certificates.md)しました。次に、Azure AD を Kubernetes のロールベースアクセス制御 (RBAC) に対応させます。これにより、Azure AD セキュリティ グループとユーザーが Kubernetes コントロール プレーンへのグループベースのアクセスを確保します。

## 期待される結果

以下の手順に従うと、Kubernetes コントロール プレーン (Cluster API) の認証に使用される Azure AD 構成が作成されます。

| オブジェクト | 目的 |
|------------------------------------|---------------------------------------------------------|
| クラスター管理者セキュリティ グループ | `cluster-admin` Kubernetes ロールにマップされます。 |
| クラスター管理者ユーザー | 最低 1 人のブレーク グラス クラスター管理者ユーザーを表します。 |
| クラスター管理者グループのメンバーシップ | クラスター管理者ユーザーとクラスター管理者セキュリティ グループの間の関連付け。 |
| 名前空間リーダーのセキュリティ グループ | 特定の名前空間に対して読み取り専用アクセスを持つユーザーを表します。 |
| _追加のセキュリティ グループ_ | _オプションです。_ 他のビルトインおよびカスタム Kubernetes ロールを使用する予定のセキュリティ グループ (およびそのメンバーシップ)。 |

これは、ワークロード アイデンティティに関連する構成を設定しません。この構成は、クラスター管理を実行するための RBAC アクセスを設定するためにのみ使用されます。

## 手順

> :book: コントソ自転車 Azure AD チームは、すべての AKS クラスターへの管理アクセスをセキュリティ グループに基づいて行う必要があります。これは、BU0001 ビジネス ユニットの Application ID a0008 用に構築されている新しい AKS クラスターに適用されます。Kubernetes RBAC は AAD ベースであり、ユーザーの AAD グループ メンバーシップに基づいてアクセスが付与されます。

1. Azure サブスクリプションのテナントID をクエリして保存します。

   ```bash
   export TENANTID_AZURERBAC_AKS_BASELINE=$(az account show --query tenantId -o tsv)
   echo TENANTID_AZURERBAC_AKS_BASELINE: $TENANTID_AZURERBAC_AKS_BASELINE
   ```

2. Contoso Bicycle Azure AD チームとして役割を果たして、Kubernetes Cluster API の認証が関連付けられるテナントにログインします。

   > :bulb: Kubernetes 認証に現在のユーザー アカウントの Azure AD テナントを使用する予定の場合は、`az login` コマンドをスキップします。

   ```bash
   az login -t <Replace-With-ClusterApi-AzureAD-TenantId> --allow-no-subscriptions
   export TENANTID_K8SRBAC_AKS_BASELINE=$(az account show --query tenantId -o tsv)
   echo TENANTID_K8SRBAC_AKS_BASELINE: $TENANTID_K8SRBAC_AKS_BASELINE
   ```

3. Azure AD セキュリティ グループを作成/識別します。これは、[Kubernetes Cluster Admin](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) ロール `cluster-admin` にマップされます。

   既存の適切なクラスター管理サービス アカウントのセキュリティ グループをすでに持っている場合は、そのグループを使用し、新しいグループを作成しないでください。既存のグループを使用するか、Azure AD 管理者が使用するために作成したグループを使用する場合は、参照実装全体でグループ名と ID を更新する必要があります。

   ```bash
   export AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE=[既存のクラスター管理グループの Object ID をここに貼り付けます。]
   echo AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE: $AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE
   ```

   新しいグループを作成する場合は、次のコードを使用できます。

   ```bash
   export AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE=$(az ad group create --display-name 'cluster-admins-bu0001a000800' --mail-nickname 'cluster-admins-bu0001a000800' --description "Principals in this group are cluster admins in the bu0001a000800 cluster." --query id -o tsv)
   echo AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE: $AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE
   ```

   この Azure AD グループ Object ID は、クラスターを作成するときに使用されます。このようにして、クラスターがデプロイされると、新しいグループに Kubernetes の適切な Cluster Role バインディングが付与されます。


5. AKS クラスターの "break-glass" クラスター管理者ユーザーを作成します。

   > :book: 組織は、重要なインフラストラクチャに対する break-glass 管理者ユーザーの価値を理解しています。アプリケーション チームはクラスター管理者ユーザーを要求し、Azure AD 管理者チームは Azure AD でユーザーを作成します。

   この手順をスキップするには、前の手順で識別されたグループにすでにクラスター管理者としてメンバーが割り当てられている必要があります。

   ```bash
   TENANTDOMAIN_K8SRBAC=$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d '@' -f 2 | sed 's/\"//')
   AADOBJECTNAME_USER_CLUSTERADMIN=bu0001a000800-admin
   AADOBJECTID_USER_CLUSTERADMIN=$(az ad user create --display-name=${AADOBJECTNAME_USER_CLUSTERADMIN} --user-principal-name ${AADOBJECTNAME_USER_CLUSTERADMIN}@${TENANTDOMAIN_K8SRBAC} --force-change-password-next-sign-in --password ChangeMebu0001a0008AdminChangeMe --query id -o tsv)
   echo TENANTDOMAIN_K8SRBAC: $TENANTDOMAIN_K8SRBAC
   echo AADOBJECTNAME_USER_CLUSTERADMIN: $AADOBJECTNAME_USER_CLUSTERADMIN
   echo AADOBJECTID_USER_CLUSTERADMIN: $AADOBJECTID_USER_CLUSTERADMIN
   ```

6. クラスター管理者セキュリティグループにクラスター管理者ユーザーを追加します。

   > :book: 最近作成された break-glass 管理者ユーザーは、Azure AD から Kubernetes クラスター管理グループに追加されます。このステップを実行した後、Azure AD 管理者チームはアプリケーション チームの要求を完了します。

   前のステップで識別されたグループにすでにクラスター管理者が割り当てられている場合は、このステップをスキップします。

   ```bash
   az ad group member add -g $AADOBJECTID_GROUP_CLUSTERADMIN_AKS_BASELINE --member-id $AADOBJECTID_USER_CLUSTERADMIN
   ```

7. Create/identify the Azure AD security group that is going to be a namespace reader. _Optional_

   ```bash
   export AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE=$(az ad group create --display-name 'cluster-ns-a0008-readers-bu0001a000800' --mail-nickname 'cluster-ns-a0008-readers-bu0001a000800' --description "Principals in this group are readers of namespace a0008 in the bu0001a000800 cluster." --query id -o tsv)
   echo AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE: $AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE
   ```

7. 作成または識別する Azure AD セキュリティ グループを、ネームスペース リーダーとして識別します。_任意の手順_

   ```bash
   export AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE=$(az ad group create --display-name 'cluster-ns-a0008-readers-bu0001a000800' --mail-nickname 'cluster-ns-a0008-readers-bu0001a000800' --description "Principals in this group are readers of namespace a0008 in the bu0001a000800 cluster." --query id -o tsv)
   echo AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE: $AADOBJECTID_GROUP_A0008_READER_AKS_BASELINE
   ```

## Kubenetes RBAC のバックストア

AKS は、2 つの異なるモードで Kubernetes を Azure AD とバックアップできます。1 つは、Azure AD とクラスターの Kubernetes `ClusterRoleBindings`/`RoleBindings` の直接の関連付けです。これは、Kubernetes RBAC をバックアップする Azure AD を使用するかどうかにかかわらず、Azure リソースのバックアップを行っているテナントと同じか異なるかにかかわらず可能です。ただし、Azure リソース (Azure RBAC ソース) のバックアップを行っているテナントが、Kubernetes RBAC をバックアップする予定のテナントと同じ場合、Azure RBAC を使用して Azure AD とクラスターの間に間接的な関連付けを行うことで、クラスターの直接の `RoleBinding` の操作を介して Azure AD とバックアップできます。この手順を実行するときは、グループとユーザーを管理するために Azure AD で必要な高い権限を持っていないため、別のテナントにクラスターを関連付ける必要があったかもしれません。ただし、本番環境に適用する場合は、テナントが同じ場合は、Kubernetes RBAC のバックストアとして Azure RBAC を使用していることを確認してください。両方のケースは、Azure AD と AKS の間の統合された認証を利用します。Azure RBAC は、通常、組織のガバナンス戦略により良く合致するように、クラスター内の yaml ベースの管理の代わりに Azure RBAC による制御を高めます。


### AKS RBAC の _好ましい状態_

このウォークスルーで単一のテナントを使用している場合、後ほどクラスターの展開ステップが、上記で作成したグループに必要なロールの割り当てを行います。具体的には、上記の手順では、ネームスペース `a0008` のネームスペース リーダーとして機能する Azure AD セキュリティ グループ `cluster-ns-a0008-readers-bu0001a000800` を作成し、Azure AD セキュリティ グループ `cluster-admins-bu0001a000800` にはクラスター管理者が含まれます。これらのグループの Object ID は、それぞれ 'Azure Kubernetes Service RBAC Reader' と 'Azure Kubernetes Service RBAC Cluster Admin' RBAC ロールに関連付けられ、クラスター内の適切なレベルにスコープされます。

Azure RBAC を認証アプローチとして使用することは、Azure リソース、AKS、および Kubernetes リソースの統合された管理とアクセス制御を可能にするため、最終的に好ましい方法です。この時点で、典型的なクラスター アクセス パターンを表す 4 つの [Azure RBAC ロール](https://learn.microsoft.com/azure/aks/manage-azure-rbac#create-role-assignments-for-users-to-access-cluster) があります。

### 直接的な Kubernetes RBAC 管理 _[代替手段]_

1. 追加の Kubernetes RBAC 統合のセットアップ _任意、フォークが必要です_

   > :book: チームは、クラスターにグループ管理されたアクセスが必要なクラスター管理者だけでないことを知っています。Kubernetes には、名前空間とクラスターの両方で使用できる Azure AD グループにマップできる _admin_、_edit_、および _view_ などの他のロールもあります。同様に、カスタム ロールを作成でき、Azure AD グループにマップする必要があります。

   [`cluster-rbac.yaml` ファイル](./cluster-manifests/cluster-rbac.yaml) と様々な名前空間 [`rbac.yaml ファイル`](./cluster-manifests/cluster-baseline-settings/rbac.yaml) で、必要なものをコメントアウトし、`<replace-with-an-aad-group-object-id...>` プレースホルダーを、このクラスターまたは名前空間の目的にマップする新しいまたは既存の Azure AD グループに置き換えます。**このウォークスルーではこの操作を実行する必要はありません**。これらは参照のためにのみここにあります。

### Save your work in-progress

```bash
# run the saveenv.sh script at any time to save environment variables created above to aks_baseline.env
./saveenv.sh

# if your terminal session gets reset, you can source the file to reload the environment variables
# source aks_baseline.env
```

### ワークインプログレスを保存する

```bash
# いつでも saveenv.sh スクリプトを実行して、上記で作成された環境変数を aks_baseline.env に保存します
./saveenv.sh

# ターミナル セッションがリセットされた場合は、ファイルをソース化して環境変数を再読み込みできます
# source aks_baseline.env
```

### 次のステップ

:arrow_forward: [ハブ・スポーク ネットワーク トポロジーの展開](./04-networking.md)