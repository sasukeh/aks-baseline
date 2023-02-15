# クリーンアップ

デプロイされた [AKS ベースライン クラスター](./) を探索した後、作成された Azure リソースを削除して、意図しないコストが蓄積されないようにする必要があります。 このリファレンス実装の一部として作成されたすべてのリソースを削除するには、次の手順に従います。

## 手順

1. Azure リソースに含まれるすべてのリソースを削除するために、リソース グループを削除します。

   > このリファレンス実装で、関連したすべての Azure リソースを削除するには、作成された 3 つのリソース グループを削除する必要があります。

   :warning: 正しいサブスクリプションを使用していることを確認し、これらのグループに存在するリソースが削除するものであることを確認してください。

   ```bash
   az group delete -n rg-bu0001a0008
   az group delete -n rg-enterprise-networking-spokes
   az group delete -n rg-enterprise-networking-hubs
   ```

2. Azure Key Vault を削除する

   > このリファレンス実装では、Key Vault でソフト削除が有効になっているため、名前の衝突が発生しないように、削除を実行します。

   ```bash
   az keyvault purge -n $KEYVAULT_NAME_AKS_BASELINE
   ```

3. Azure AD または Azure RBAC の権限に一時的な変更が加えられている場合は、それらも削除します。

4. [Remove the Azure Policy assignments](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyMenuBlade/Compliance) scoped to the cluster's resource group. To identify those created by this implementation, look for ones that are prefixed with `[your-cluster-name] `.

4. [Azure Policy の割り当てを削除します](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyMenuBlade/Compliance)。 この実装によって作成されたものを特定するには、`[your-cluster-name]` で始まるものを探します。

## 自動化

プロセスを自動化する前に、ここで提示されたようなより生の形式でプロセスを体験することが重要です。 その経験により、さまざまなステップ、内部およびチーム間の依存関係、および途中の失敗点を理解できます。 ただし、このウォークスルーで提供される手順は、自動化を意識して設計されていません。 これは、組織でよく遭遇する一般的な職務分担の観点を提示しますが、組織とは一致しない場合があります。


今、関与するコンポーネントを理解し、チームと組織の間の共有された責任を特定したので、最終的なインフラストラクチャとクラスターのブートストラッピングの周りに繰り返し使用可能なデプロイメント プロセスを構築することをお勧めします。 [AKS ベースライン自動化ガイダンス](https://github.com/Azure/aks-baseline-automation#aks-baseline-automation) は、GitHub Actions と Infrastructure as Code を組み合わせて、この自動化を容易にする方法を学ぶためのものです。 そのガイダンスは、ここで歩んだ同じアーキテクチャの基礎に基づいています。

> Note: [AKS ベースライン自動化ガイダンス](https://github.com/Azure/aks-baseline-automation#aks-baseline-automation) 実装は、このリポジトリと同期を維持しようと努めていますが、さまざまな決定にわずかに偏りが生じる可能性があり、新しい機能を導入するか、このリポジトリで使用されている機能がまだない可能性があります。 それらは、設計上機能的に整合していますが、必ずしも同一ではありません。 そのリポジトリを使用して、自動化の潜在的な可能性を探索し、このリポジトリはコア アーキテクチャ ガイダンスのために使用します。

### 次のステップ

:arrow_up_small: [概要へ戻る](./README.md#overview)

