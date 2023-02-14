# Prerequisites

# 事前準備

This is the starting point for the instructions on deploying the [AKS baseline reference implementation](./README.md). There is required access and tooling you'll need in order to accomplish this. Follow the instructions below and on the subsequent pages so that you can get your environment ready to proceed with the AKS cluster creation.

[AKS ベースラインリファレンス実装] の展開に関する手順の開始地点です。 これを達成するために必要なアクセスとツールがあります。 次のページの手順に従って、 AKS クラスターの作成を進めるための環境を準備してください。

| :clock10: | These steps are intentionally verbose, intermixed with context, narrative, and guidance. The deployments are all conducted via [Bicep templates](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview), but they are executed manually via `az cli` commands. We strongly encourage you to dedicate time to walk through these instructions, with a focus on learning. We do not provide any "one click" method to complete all deployments.<br><br>Once you understand the components involved and have identified the shared responsibilities between your team and your greater organization, you are encouraged to build suitable, repeatable deployment processes around your final infrastructure and cluster bootstrapping. The [AKS baseline automation guidance](https://github.com/Azure/aks-baseline-automation#aks-baseline-automation) is a great place to learn how to build your own automation pipelines. That guidance is based on the same architecture foundations presented here in the AKS baseline, and illustrates GitHub Actions based deployments for all components, including workloads. |
|-----------|:--------------------------|

| :clock10: | これらの手順は意図的に冗長であり、コンテキスト、ネarrative、およびガイダンスとともに混在しています。 デプロイはすべて [Bicep テンプレート] で実行されますが、 `az cli` コマンドを介して手動で実行されます。 私たちは、これらの手順を通じて時間を割いて学習に焦点を当てて歩くことを強くお勧めします。 すべてのデプロイメントを完了するための「1 クリック」方法は提供していません。<br><br>コンポーネントが含まれていることを理解し、チームと組織の共有の責任を特定したら、最終的なインフラストラクチャとクラスターのブートストラップに囲む適切な繰り返し可能なデプロイメントプロセスを構築することをお勧めします。 [AKS ベースライン自動化ガイダンス] は、自分の自動化パイプラインを構築する方法を学ぶのに最適な場所です。 そのガイダンスは、ここで紹介されている AKS ベースラインの同じアーキテクチャの基礎に基づいており、すべてのコンポーネントを含むワークロードに基づいた GitHub Actions ベースのデプロイを示しています。 |

## Steps

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

3. Latest [Azure CLI installed](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest) (must be at least 2.40), or you can perform this from Azure Cloud Shell by clicking below.

   [![Launch Azure Cloud Shell](https://learn.microsoft.com/azure/includes/media/cloud-shell-try-it/launchcloudshell.png)](https://shell.azure.com)

3. 最新の [Azure CLI をインストール](https://docs.microsoft.com/ja-jp/cli/azure/install-azure-cli?view=azure-cli-latest) します (少なくとも 2.40 である必要があります)。 または、以下をクリックして Azure Cloud Shell から実行できます。

   [![Launch Azure Cloud Shell](https://docs.microsoft.com/ja-jp/azure/includes/media/cloud-shell-try-it/launchcloudshell.png)](https://shell.azure.com)

4. 次の機能はまだ _プレビュー_ ですが、ターゲット サブスクリプションで有効にしてください。

   1. [Defender for Containers プレビュー機能 = `AKS-AzureDefender` を登録](https://docs.microsoft.com/ja-jp/azure/azure-defender-for-containers/quickstart-onboard-aks?pivots=client-operating-system-linux#register-the-defender-for-containers-preview-feature)

   2. [Workload Identity プレビュー機能 = `EnableWorkloadIdentityPreview` を登録](https://docs.microsoft.com/ja-jp/azure/aks/use-managed-identity#register-the-enableworkloadidentitypreview-feature-flag)

   3. [ImageCleaner (Earser) プレビュー機能 = `EnableImageCleanerPreview` を登録](https://docs.microsoft.com/ja-jp/azure/aks/clean-up-unused-container-images#prerequisites)

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

6. Ensure [OpenSSL is installed](https://github.com/openssl/openssl#download) in order to generate self-signed certs used in this implementation. _OpenSSL is already installed in Azure Cloud Shell._

   > :warning: Some shells may have the `openssl` command aliased for LibreSSL. LibreSSL will not work with the instructions found here. You can check this by running `openssl version` and you should see output that says `OpenSSL <version>` and not `LibreSSL <version>`.

6. この実装で使用される自己署名証明書を生成するために [OpenSSL をインストール](https://github.com/openssl/openssl#download) してください。 _OpenSSL は Azure Cloud Shell にすでにインストールされています。_

   > :warning: 一部のシェルでは、`openssl` コマンドが LibreSSL 用にエイリアスされている場合があります。 LibreSSL はここで見つかる手順では機能しません。 `openssl version` を実行して、`OpenSSL <version>` と表示され、`LibreSSL <version>` と表示されないことを確認できます。

### 次のステップ

:arrow_forward: [クライアント用の TLS 証明書の生成 ](./02-ca-certificates.md)
