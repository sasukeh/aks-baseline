# ワークロードの前提条件

AKS クラスターは [ブートストラップ](./07-bootstrap-validation.md) され、[AKS ベースライン リファレンス実装](./) のインフラストラクチャ フォーカスを終えました。次の手順に従って、Ingress コントローラーが Application Gateway に接続するために Web アプリケーションに提供する TLS 証明書をインポートします。
## 手順

## Import the wildcard certificate for the AKS ingress controller to Azure Key Vault

> :book: Contoso Bicycle procured a CA certificate, a standard one, to be used with the AKS ingress controller. This one is not EV, as it will not be user facing.

## AKS イングレスコントローラー のワイルドカード証明書を Azure Key Vault にインポートする

1. Azure Key Vault の詳細を取得し、現在のユーザーに証明書のインポート権限とネットワークアクセスを付与します。

   > :book: Appチームは、AKS ingressコントローラーのワイルドカード証明書を`*.aks-ingress.contoso.com`として使用することを決定します。彼らはAzure Key Vaultを使用して、この証明書のライフサイクルを管理します。

   ```bash
   export KEYVAULT_NAME_AKS_BASELINE=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   echo KEYVAULT_NAME_AKS_BASELINE: $KEYVAULT_NAME_AKS_BASELINE
   TEMP_ROLEASSIGNMENT_TO_UPLOAD_CERT=$(az role assignment create --role a4417e6f-fecd-4de8-b567-7b0420556985 --assignee-principal-type user --assignee-object-id $(az ad signed-in-user show --query 'id' -o tsv) --scope $(az keyvault show --name $KEYVAULT_NAME_AKS_BASELINE --query 'id' -o tsv) --query 'id' -o tsv)
   echo TEMP_ROLEASSIGNMENT_TO_UPLOAD_CERT: $TEMP_ROLEASSIGNMENT_TO_UPLOAD_CERT

   # もしプロキシやその他の出口が一貫したIPを提供しない場合は、このトラフィックを許可するためにAzure Key Vaultファイアウォールを手動で調整する必要があります。
   CURRENT_IP_ADDRESS=$(curl -s -4 https://ifconfig.io)
   echo CURRENT_IP_ADDRESS: $CURRENT_IP_ADDRESS
   az keyvault network-rule add -n $KEYVAULT_NAME_AKS_BASELINE --ip-address ${CURRENT_IP_ADDRESS}
   ```

2. AKS イングレスコントローラーのワイルドカード証明書 `*.aks-ingress.contoso.com` をインポートします。

   :warning: 既に [適切な証明書](https://learn.microsoft.com/azure/key-vault/certificates/certificate-scenarios#formats-of-import-we-support) へのアクセスがある場合、または組織から取得できる場合は、このステップでそれを使用することを検討してください。詳細については、[Azure Key Vault を使用した証明書のインポートチュートリアル](https://learn.microsoft.com/azure/key-vault/certificates/tutorial-import-certificate#import-a-certificate-to-key-vault) を参照してください。

   :warning: 実際のデプロイには、このスクリプトで作成された証明書を使用しないでください。自己署名証明書の使用は、説明のために簡単にするためにのみ提供されます。_開発目的であっても_ 、クラスターには、TLS 証明書の取得とライフタイム管理の組織の要件を使用してください。

   ```bash
   cat traefik-ingress-internal-aks-ingress-tls.crt traefik-ingress-internal-aks-ingress-tls.key > traefik-ingress-internal-aks-ingress-tls.pem
   az keyvault certificate import -f traefik-ingress-internal-aks-ingress-tls.pem -n traefik-ingress-internal-aks-ingress-tls --vault-name $KEYVAULT_NAME_AKS_BASELINE
   ```

3. Azure Key Vault からの証明書インポート権限とネットワークアクセスを現在のユーザーから削除します。

   > :book: このワークスルーで証明書をアップロードするために、Azure Key Vault RBAC割り当ては一時的に許可されていました。実際のデプロイでは、[Azure Key Vault データプレーンの Azure RBAC](https://learn.microsoft.com/azure/key-vault/general/secure-your-key-vault#data-plane-and-access-policies) を使用して、ARMテンプレートを介してこれらの任意のRBACポリシーを管理し、ネットワークアクセスを許可されたトラフィックのみがKey Vaultにアクセスするようにします。

   ```bash
   az keyvault network-rule remove -n $KEYVAULT_NAME_AKS_BASELINE --ip-address "${CURRENT_IP_ADDRESS}/32"
   az role assignment delete --ids $TEMP_ROLEASSIGNMENT_TO_UPLOAD_CERT
   ```


## Azure Policy の適用を確認する

> :book: アプリチームは、他の Azure リソースと同様に AKS クラスターに Azure Policy を適用したいと考えています。彼らのポッドは、[Azure Policy アドオン for AKS](https://docs.microsoft.com/ja-jp/azure/governance/policy/concepts/policy-for-kubernetes) を使用してカバーされます。これらの監査のいくつかは、ポッドの仕様が組織のセキュリティベストプラクティスに準拠していることを確認するために、特定の Kubernetes API リクエスト操作を拒否することで終わる可能性があります。さらに、[Azure Policy によって生成されたデータ](https://docs.microsoft.com/ja-jp/azure/governance/policy/how-to/get-compliance-data) は、アプリチームが AKS クラスターの現在のコンプライアンス状態を評価するプロセスを支援するために使用されます。アプリチームは、リソース グループ レベルで [Azure Policy for Kubernetes built-in restricted initiative](https://docs.microsoft.com/ja-jp/azure/governance/policy/samples/policy-for-kubernetes) と、リソース要求を実行するようにポッドを定義し、信頼できるコンテナレジストリを定義し、ルートファイルシステムへのアクセスが読み取り専用であることを強制し、内部ロードバランサーの使用を強制し、https-only Kubernetes Ingress オブジェクトを強制する [built-in individual Azure policies](https://learn.microsoft.com/azure/aks/policy-samples#microsoftcontainerservice) の 5 つを適用するように割り当てます。
> 
> さらに、内部ガバナンスでは、チームが、パブリックエンドポイントが所有しているドメインサフィックスで終わる完全修飾ドメイン名を介して公開されていることを確認するように求められています。この要件を、クラスターの Ingress コントローラーによって公開されるすべてのエンドポイントに適用するために、[Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/) を使用してカスタムポリシーを定義し、[Azure Policy を介してデプロイする](https://docs.microsoft.com/ja-jp/azure/governance/policy/concepts/rego-for-aks#deploy-a-custom-policy-definition) 機能を活用します。


1. AKS クラスターへのポリシーの適用を確認する

   ```bash
   kubectl get constrainttemplate
   ```

   次のような出力が返されるはずです

   ```output
   NAME                                                 AGE
   k8sazurev1blockdefault                               21m
   k8sazurev1blockendpointeditdefaultrole               21m
   … more …            
   k8sazurev3noprivilegeescalation                      21m
   k8sazurev3readonlyrootfilesystem                     21m
   k8scustomingresstlshostshavedefineddomainsuffix      21m
   ```

### 進捗を保存する

```bash
# saveenv.shスクリプトをいつでも実行して、上記で作成された環境変数を aks_baseline.env に保存します。
./saveenv.sh

# ターミナルセッションがリセットされた場合は、ファイルをソース化して環境変数を再読み込みできます。
# source aks_baseline.env
```

### 次のステップ

:arrow_forward: [AKS Ingress コントローラーの Azure Key Vault インテグレーションを構成する](./09-secret-management-and-ingress-controller.md)
