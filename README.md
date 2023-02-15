# Azure Kubernetes Service (AKS) ベースラインクラスター


このリファレンス実装は、一般的な [AKS クラスター](https://azure.microsoft.com/services/kubernetes-service) の _推奨される開始 (ベースライン) インフラストラクチャ アーキテクチャ_ を示します。この実装とドキュメントは、ネットワーキング、セキュリティ、および開発のような複数の異なるチームを通して、この一般的なベースラインインフラストラクチャを展開し、そのコンポーネントを理解するためのプロセスを導くことを目的としています。


ここでは、各コンポーネントを理解するために、 _冗長な_ 方法で展開を行います。理想的には、各レイヤーについて理解し、それを自分のワークロードに適用するために必要な知識を提供します。

## Azure Architecture Center のガイダンス

このプロジェクトには、セキュアな AKS クラスターの課題、デザイン パターン、およびベスト プラクティスのコンパニオン セットの記事があります。Azure Architecture Center の [Azure Kubernetes Service (AKS) ベースラインクラスター](https://aka.ms/architecture/aks-baseline) で見つけることができます。まだ読んでいない場合は、この実装に適用された考慮事項を追加するコンテキストを提供するために、それを読むことをお勧めします。最終的に、これは、その特定のアーキテクチャ ガイダンスの直接の実装です。

## アーキテクチャ


**このアーキテクチャはインフラストラクチャに焦点を当てています**。より多くのワークロードに焦点を当てています。AKS クラスター自体に焦点を当てています。これには、アイデンティティ、デプロイ後の構成、シークレット管理、およびネットワークトポロジーに関する懸念が含まれます。

この実装は、Azure サービスとの統合を通じて、観測性を提供し、複数のリージョンでの成長をサポートするネットワークトポロジーを提供し、クラスター内のトラフィックを安全に保つようにします。このアーキテクチャは、プレプロダクションとプロダクションのステージの開始点として考慮する必要があります。

ここの資料は、比較的密集しています。これらの手順を理解するために時間を割くことを強くお勧めします。ここでは、"一発クリック" デプロイは提供していません。ただし、コンポーネントを理解し、チームと組織の間の共有の責任を特定した後は、最終的なインフラストラクチャに対して適切で監査可能なデプロイメント プロセスを構築することをお勧めします。

リファレンス実装を通じて、_Contoso Bicycle_ への参照が表示されます。これは、北米西海岸にある顧客にオンライン Web サービスを提供する小さくて急成長している仮想のスタートアップです。彼らはオンプレミスのデータ センターを持っていません。すべてのコンテナ化されたビジネス アプリケーションは、安全で企業向けの AKS クラスターによってオーケストレートされるようになりました。[彼らの要件と IT チームの構成](./contoso-bicycle/README.md) について詳しく読むことができます。このネタは、いくつかの実装の詳細、命名規則などの基盤を提供します。必要に応じて適応してください。

最後に、この実装では、[ASP.NET Core Docker サンプル Web アプリ](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp)を例のワークロードとして使用します。このワークロードは、基本的なインフラストラクチャを体験するのに役立つために、無関心にしています。

### コア アーキテクチャ コンポーネント

#### Azure platform

#### Azure プラットフォーム

- AKS v1.25
  - システムとユーザー [ノード プールの分離](https://learn.microsoft.com/azure/aks/use-system-pools)
  - [AKS 管理の Azure AD](https://learn.microsoft.com/azure/aks/managed-aad)
  - Azure AD ベースの Kubernetes RBAC (_ローカルユーザーアカウントは無効になっています_)
  - Managed ID
  - Azure CNI
  - [Azure Monitor for containers](https://learn.microsoft.com/azure/azure-monitor/insights/container-insights-overview)
  - Azure Virtual Networks (hub-spoke)
    - Azure Firewall managed egress
  - Azure Application Gateway (WAF)
  - AKS 管理の内部ロード バランサー

#### クラスター内の OSS コンポーネント

- [Azure Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-overview) _[AKS-managed add-on]_
- [Flux GitOps Operator](https://fluxcd.io) _[AKS-managed extension]_
- [ImageCleaner (Eraser)](https://learn.microsoft.com/azure/aks/image-cleaner) _[AKS-managed add-on]_
- [Kubernetes Reboot Daemon](https://learn.microsoft.com/azure/aks/node-updates-kured)
- [Secrets Store CSI Driver for Kubernetes](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver) _[AKS-managed add-on]_
- [Traefik Ingress Controller](https://doc.traefik.io/traefik/v2.5/routing/providers/kubernetes-ingress/)


![Network diagram depicting a hub-spoke network with two peered VNets and main Azure resources used in the architecture.](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/images/secure-baseline-architecture.svg)

## リファレンス実装のデプロイ

AKS ホステッド ワークロードのデプロイでは、通常、前提条件、ホスト ネットワーク、クラスター インフラストラクチャ、最後にワークロード自体のエリアで責任の分離とライフサイクル管理が発生します。このリファレンス実装も同様です。また、ベースライン クラスターのトポロジーと決定を示すことを主な目的としていることに注意してください。私たちは、"ステップ バイ ステップ" フローがソリューションの一部を学び、それらの関係性について洞察を得るのに役立つと考えています。最終的には、クラスターとその依存関係のライフサイクル/SDLC 管理は、状況に応じて (チームの役割、組織の標準など) 、必要に応じて実装されます。

** この学習の旅を _Preparing for the cluster_ セクションから開始してください。 ** これを最後まで実行すると、推奨されるベースライン クラスターがインストールされ、終端から終端までのサンプル ワークロードが実行され、自分の Azure サブスクリプションで参照できるようになります。
### 1. :rocket: クラスターの準備

クラスターをデプロイする前に、考慮すべき点があります。このサイズのデプロイを行うには、サブスクリプションと AD テナントで十分な権限があるかどうか？これは、自分のチームが直接対応するのか、別のチームが責任を負うのか、といった点でどの程度対応するのか？などです。

- [ ] [事前準備](./01-prerequisites.md)に進み、必要なものをインストールします。
- [ ] [クライアント向けおよび AKS Ingress コントローラーの TLS 証明書を取得します](./02-ca-certificates.md)
- [ ] [Azure AD 統合を計画します](./03-aad.md)

### 2. ターゲット ネットワークの構築

Microsoft は、AKS を計画的に構築することをお勧めします。必要なサイズに適切にサイズされ、適切なネットワークの監視が可能である必要があります。組織では通常、従来のハブ-スポーク モデルが好まれます。この実装では、このモデルが反映されています。このモデルは標準のハブ-スポーク モデルですが、理解すべき基本的なサイジングと分割の考慮事項が含まれています。

- [ ] [ハブ-スポークネットワークを構築する](./04-networking.md)

### 3. クラスターのデプロイ

リファレンス実装の中心となるガイダンスです。これは、前のネットワーク トポロジ ガイダンスとペアリングされています。ここでは、クラスターの Azure リソースと、Azure Application Gateway WAF、Azure Monitor、Azure Container Registry、Azure Key Vault などの隣接サービスをデプロイします。ここで、クラスターがブートストラップされていることを検証します。

- [ ] [クラスターブートストラップの準備](./05-bootstrap-prep.md)
- [ ] [AKSクラスターとサポートサービスのデプロイ](./06-aks-cluster.md)
- [ ] [クラスターブートストラップの検証](./07-bootstrap-validation.md)

前段のステップは、ここで手動で実行することで、関連するコンポーネントを理解できますが、私たちは自動化された DevOps プロセスを推奨します。したがって、IaC と同様に、CI/CD パイプラインに前段のステップを組み込んでください。詳細については、専用の [AKS ベースライン自動化ガイダンス](https://github.com/Azure/aks-baseline-automation#aks-baseline-automation) を参照してください。

### 4. ワークロードのデプロイ

クラスターへのワークロードのデプロイは、この基盤がビジネスの信頼できるアプリケーション プラットフォームとして機能する方法を理解するためには、困難です。このワークロードのデプロイは、通常、CI/CD パターンに従い、さらに高度なデプロイ戦略（ブルー/グリーンなど）を含む可能性があります。次のステップは、この基盤の説明のために適した手動デプロイを表します。

- [ ] クラスターのために、[ワークロードの準備を行います](./08-workload-prerequisites.md)
- [ ] [Azure Keyvault と AKS Ingress コントローラーの統合を構成します](./09-secret-management-and-ingress-controller.md)
- [ ] [ワークロードをデプロイします](./10-workload.md)

### 5. :checkered_flag: 検証

クラスターとサンプル ワークロードがデプロイされたら、クラスターがどのように機能しているかを確認します。

- [ ] [エンドツーエンドの検証を実行します](./11-validation.md)

## :broom: リソースの削除

前段のステップでデプロイされた Azure リソースのほとんどは、削除されない限り、継続的な課金が発生します。

- [ ] [すべてのリソースの削除](./12-cleanup.md)

## プレビューと追加機能

Kubernetes と AKS は、急速に進化する製品です。[AKS ロードマップ](https://aka.ms/AKS/Roadmap) には、製品の変化がどのように速いかが示されています。このリファレンス実装では、AKS チームが「Shipped & Improving」として説明するプレビュー機能のいくつかに依存しています。その背景にある理由は、多くのプレビュー機能が数ヶ月しか続かないためです。GA に入る前に、今日クラスターをアーキテクチャリングしている場合、多くのプレビュー機能が近づいているか、すでに GA に入っている可能性があります。

以下の実装は、すべてのプレビュー機能を含みませんが、一般的なクラスターに大きな価値を追加するもののみを含みます。セキュリティ、管理可能性などのポスチャーを強化するために、プレプロダクションクラスターで評価することをお勧めします。これらの機能がプレビューから出てくると、このリファレンス実装はそれらを組み込むように更新される可能性があります。以下の機能を試してフィードバックを提供してください。

- [BYO Kubelet Identity](https://learn.microsoft.com/azure/aks/use-managed-identity#bring-your-own-kubelet-mi)
- [Planned maintenance window](https://learn.microsoft.com/azure/aks/planned-maintenance)
- [BYO CNI (`--network-plugin none`)](https://learn.microsoft.com/azure/aks/use-byo-cni)
- [Simplified application autoscaling with Kubernetes Event-driven Autoscaling (KEDA) add-on](https://learn.microsoft.com/azure/aks/keda)

## 関連リファレンス実装

AKS ベースラインは、次の追加のリファレンス実装の基礎として使用されました。これらは、AKS ベースラインの学習を基にしており、特定のトポロジー、要件、および/またはワークロード タイプに合わせてクラスターに特定のレンズを適用します。

- [AKS baseline for multi-region clusters](https://github.com/mspnp/aks-baseline-multi-region)
- [AKS baseline for regulated workloads](https://github.com/mspnp/aks-baseline-regulated)
- [AKS baseline for microservices](https://github.com/mspnp/aks-fabrikam-dronedelivery)
- [Azure landing zones, enterprise-scale reference implementation using Terraform](https://github.com/Azure/caf-terraform-landingzones-starter/tree/starter/enterprise_scale/construction_sets/aks/online/aks_secure_baseline)

## 進歩的なトピック

このリファレンス実装では、より高度なシナリオをカバーすることは意図的に行っていません。以下のようなトピックは対象外となっています。

- Cluster lifecycle management with regard to SDLC and GitOps
- Workload SDLC integration (including concepts like [Bridge to Kubernetes](https://learn.microsoft.com/visualstudio/containers/bridge-to-kubernetes), advanced deployment techniques, [Draft](https://learn.microsoft.com/azure/aks/draft), etc)
- Container security
- Multiple (related or unrelated) workloads owned by the same team
- Multiple workloads owned by disparate teams (AKS as a shared platform in your organization)
- Cluster-contained state (PVC, etc)
- Windows node pools
- Scale-to-zero node pools and event-based scaling (KEDA)
- [Terraform](https://learn.microsoft.com/azure/developer/terraform/create-k8s-cluster-with-tf-and-aks)
- [dapr](https://github.com/dapr/dapr)

- クラスターのライフサイクル管理に関する SDLC と GitOps
- ワークロード SDLC 統合（[Bridge to Kubernetes](https://learn.microsoft.com/visualstudio/containers/bridge-to-kubernetes) などの概念、高度なデプロイメント テクニック、[Draft](https://learn.microsoft.com/azure/aks/draft) など）
- コンテナー セキュリティ
- 複数の（関連しているかどうか）ワークロードを所有する同じチーム
- 複数ワークロードを所有する異なるチーム（組織内で共有プラットフォームとしての AKS）
- クラスター内に含まれる状態（PVC など）
- Windows ノード プール
- スケール ゼロ ノード プールとイベント ベースのスケーリング（KEDA）
- [Terraform](https://learn.microsoft.com/azure/developer/terraform/create-k8s-cluster-with-tf-and-aks)
- [dapr](https://github.com/dapr/dapr)

このスペースを維持するために、これらのようなトピックに関するリファレンス実装ガイダンスを構築しています。さらにガイダンスが提供されると、このベースライン AKS 実装が開始点として使用されます。このベースラインを使用して構築されたパターンを提案したい場合は、[お問い合わせください](./CONTRIBUTING.md)。

## 最後に

Kubernetes は、とても柔軟なプラットフォームであり、インフラストラクチャおよびアプリケーション オペレーターには、ビジネスとテクノロジの目標を達成するための多くの選択肢を提供します。旅の途中で、Azure プラットフォーム機能、OSS ソリューション、サポート チャネル、規制遵守、および運用プロセスに依存するタイミングを検討する必要があります。**このリファレンス実装を、自分のチーム内でアーキテクチャの会話を始める場所として使用することをお勧めします。特定の要件に適応し、最終的にお客様を喜ばせるソリューションを提供します。**

## 関連ドキュメント

- [Azure Kubernetes Service Documentation](https://learn.microsoft.com/azure/aks/)
- [Microsoft Azure Well-Architected Framework](https://learn.microsoft.com/azure/architecture/framework/)
- [Microservices architecture on AKS](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices)

## コントリビュート・貢献

[コントリビューターガイド](./CONTRIBUTING.md)をご覧ください。

このプロジェクトは、[Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)を採用しています。詳細については、[Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/)を参照するか、追加の質問やコメントがある場合は、<opencode@microsoft.com>にご連絡ください。

With :heart: from Microsoft Patterns & Practices, [Azure Architecture Center](https://aka.ms/architecture).

