# 事前準備

[AKS ベースラインリファレンス実装](./README.md) の展開に関する手順の開始地点です。 これを達成するために必要なアクセスとツールがあります。 次のページの手順に従って、 AKS クラスターの作成を進めるための環境を準備してください。

| :clock10: | これらの手順は意図的に冗長であり、コンテキスト、ストーリー、およびガイダンスとともに混在しています。 デプロイはすべて [Bicep テンプレート](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) で実行されますが、 `az cli` コマンドを介して手動で実行されます。 私たちは、これらの手順を通じて時間を割いて学習に焦点を当てて歩くことを強くお勧めします。 すべてのデプロイメントを完了するための「1 クリック」方法は提供していません。<br><br>コンポーネントが含まれていることを理解し、チームと組織の共有の責任を特定したら、最終的なインフラストラクチャとクラスターのブートストラップに囲む適切な繰り返し可能なデプロイメントプロセスを構築することをお勧めします。 [AKS ベースライン自動化ガイダンス](https://github.com/Azure/aks-baseline-automation#aks-baseline-automation) は、自分の自動化パイプラインを構築する方法を学ぶのに最適な場所です。 そのガイダンスは、ここで紹介されている AKS ベースラインの同じアーキテクチャの基礎に基づいており、すべてのコンポーネントを含むワークロードに基づいた GitHub Actions ベースのデプロイを示しています。 |
|-----------|:--------------------------|

## 手順

1. Azure サブスクリプション

   このデプロイで使用するサブスクリプションは [無料アカウント] にできません。 標準 EA、支払い、または Visual Studio ベンフィットサブスクリプションである必要があります。 これは、ここでデプロイされるリソースが無料のサブスクリプションのクォータを超えているためです。

   > :warning: デプロイメントプロセスを開始するユーザーまたはサービス プリンシパルは、次の最小限の Azure ロール ベースのアクセス制御 (RBAC) ロールを持っている必要があります:
   >
   > * [コントリビューター ロール] は、リソース グループを作成し、デプロイを実行する機能を持つためにサブスクリプション レベルで _必要_ です。
   > * [ユーザー アクセス管理者ロール] は、さまざまなリソース グループに対して管理されたアイデンティティにロール割り当てを実行するためにサブスクリプション レベルで _必要_ です。
   > * [リソース ポリシー コントリビューター ロール] は、カスタム Azure ポリシー定義を作成して AKS クラスターのリソースを管理するためにサブスクリプション レベルで _必要_ です。

2. Kuberntes RBAC クラスター API 認証を統合するための Azure AD テナント。

   > :warning: デプロイメントプロセスを開始するユーザーまたはサービス プリンシパルは、次の最小限の Azure AD 権限を割り当てる必要があります:
   >
   > * Azure AD [ユーザー管理者](https://docs.microsoft.com/ja-jp/azure/active-directory/users-groups-roles/directory-assign-admin-roles#user-administrator-permissions) は、"break glass" AKS 管理者 Active Directory セキュリティ グループとユーザーを作成するために _必要_ です。 代わりに、Azure AD 管理者に、これを作成するように指示することができます。
   >   * Azure サブスクリプションに関連付けられているテナントのユーザー管理者グループに属していない場合は、[新しいテナントを作成](https://docs.microsoft.com/ja-jp/azure/active-directory/fundamentals/active-directory-access-create-new-tenant#create-a-new-tenant-for-your-organization) して、この実装を評価するために使用してください。 クラスターの API RBAC をバックする Azure AD テナントは、Azure サブスクリプションに関連付けられているテナントと同じテナントである必要はありません。

3. 最新の [Azure CLI をインストール](https://docs.microsoft.com/ja-jp/cli/azure/install-azure-cli?view=azure-cli-latest) します (少なくとも 2.40 である必要があります)。 または、以下をクリックして Azure Cloud Shell から実行できます。

   [![Launch Azure Cloud Shell](https://docs.microsoft.com/ja-jp/azure/includes/media/cloud-shell-try-it/launchcloudshell.png)](https://shell.azure.com)

4. 次の機能はまだ _プレビュー_ ですが、ターゲット サブスクリプションで有効にしてください。

   1. [Defender for Containers (GA済) = `AKS-AzureDefender` を登録](https://learn.microsoft.com/ja-jp/azure/defender-for-cloud/defender-for-containers-enable)

   2. [Workload Identity プレビュー機能 = `EnableWorkloadIdentityPreview` を登録](https://learn.microsoft.com/ja-jp/azure/aks/workload-identity-overview)

   3. [ImageCleaner (Earser) プレビュー機能 = `EnableImageCleanerPreview` を登録](https://learn.microsoft.com/ja-jp/azure/aks/image-cleaner?tabs=azure-cli)

   ```bash
   az feature register --namespace "Microsoft.ContainerService" -n "AKS-AzureDefender"
   az feature register --namespace "Microsoft.ContainerService" -n "EnableWorkloadIdentityPreview"
   az feature register --namespace "Microsoft.ContainerService" -n "EnableImageCleanerPreview"

   # すべてが "Registered" と表示されるまで実行します (20 分かかる場合があります)。
   az feature list -o table --query "[?name=='Microsoft.ContainerService/AKS-AzureDefender' || name=='Microsoft.ContainerService/EnableWorkloadIdentityPreview' || name=='Microsoft.ContainerService/EnableImageCleanerPreview'].{Name:name,State:properties.state}"

   # すべてが "Registered" と表示されたら、AKS リソース プロバイダーを再登録します。
   az provider register --namespace Microsoft.ContainerService
   ```

5. このリポジトリをローカルにクローン/ダウンロードするか、さらにはこのリポジトリをフォークします。

   > :twisted_rightwards_arrows: このリファレンス実装リポジトリをフォークした場合、より個人的で本番環境に近い体験にするためにいくつかのファイルとコマンドをカスタマイズできます。 このウォークスルーで説明されているこの git リポジトリへの参照を更新して、自分のフォークを使用するようにしてください。

   > リポジトリをクローンするときに HTTPS (SSH ではなく) を使用することを確認してください。 (後で Flux を使用して GitOps を構成するためにリモート URL が使用されます。 これは、HTTPS エンドポイントを正しく動作させるために必要です。)

   ```bash
   git clone https://github.com/sasukeh/aks-baseline.git
   cd aks-baseline
   ```

6. この実装で使用される自己署名証明書を生成するために [OpenSSL をインストール](https://github.com/openssl/openssl#download) してください。 _OpenSSL は Azure Cloud Shell にすでにインストールされています。_

   > :warning: 一部のシェルでは、`openssl` コマンドが LibreSSL 用にエイリアスされている場合があります。 LibreSSL はここで見つかる手順では機能しません。 `openssl version` を実行して、`OpenSSL <version>` と表示され、`LibreSSL <version>` と表示されないことを確認できます。

### 次のステップ

:arrow_forward: [クライアント用の TLS 証明書の生成 ](./02-ca-certificates.md)
