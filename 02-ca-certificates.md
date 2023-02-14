
# クライアント用 と AKS イングレスコントローラーの TLS証明書を生成する

[前提条件](./01-prerequisites.md) を満たしたら、次の手順に従って、Azure Application Gateway がクライアントに提供する Web アプリケーションの TLS 証明書を作成します。既に適切な証明書にアクセスできるか、組織から取得できる場合は、それらを使用することを検討してください。証明書の生成手順をスキップします。以下は、説明のために自己署名証明書を使用する方法を説明します。

## 手順

1. デプロイメントの残りの部分で使用されるドメインの変数を設定します。

   ```bash
   export DOMAIN_NAME_AKS_BASELINE="contoso.com"
   ```

2. クライアント用の自己署名証明書を生成します

   > :book: ウェブサイトのための CA 証明書を取得する必要があります。このサイトはユーザーに公開されるため、EV 証明書を購入します。Azure Application Gateway の前に配置されます。また、AKS Ingress Controller に使用する別の証明書も取得します。この証明書は、ユーザーに公開されないため、EV 証明書ではありません。

   :worning: このスクリプトで作成された証明書を実際のデプロイメントに使用しないでください。自己署名証明書の使用は、説明のために簡単にするためにのみ提供されています。クラスターの場合は、組織の要件に従って TLS 証明書の取得とライフタイム管理を行います。_開発目的であっても_。

   Azure Application Gateway がクライアントに提供するドメインの証明書を作成します。

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=bicycle.${DOMAIN_NAME_AKS_BASELINE}/O=Contoso Bicycle" -addext "subjectAltName = DNS:bicycle.${DOMAIN_NAME_AKS_BASELINE}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"
   openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:
   ```

3. クライアント用の証明書を Base64 エンコードします。

   :bulb: 組織から証明書を使用した場合でも、上記の手順で生成した証明書を使用した場合でも、正しく Key Vault に格納するためには、証明書（`.pfx`）を Base64 エンコードする必要があります。

   ```bash
   export APP_GATEWAY_LISTENER_CERTIFICATE_AKS_BASELINE=$(cat appgw.pfx | base64 | tr -d '\n')
   echo APP_GATEWAY_LISTENER_CERTIFICATE_AKS_BASELINE: $APP_GATEWAY_LISTENER_CERTIFICATE_AKS_BASELINE
   ```

4. AKS イングレスコントローラーのためにワイルドカード証明書を生成します。

   > :book: Contoso Bicycle は、AKS イングレスコントローラーで使用する別の TLS 証明書、標準証明書を取得します。これは、ユーザーに公開されないため、EV 証明書ではありません。最後に、アプリケーションチームは、AKS イングレスコントローラーのために `*.aks-ingress.contoso.com` のワイルドカード証明書を使用することを決定します。

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out traefik-ingress-internal-aks-ingress-tls.crt -keyout traefik-ingress-internal-aks-ingress-tls.key -subj "/CN=*.aks-ingress.${DOMAIN_NAME_AKS_BASELINE}/O=Contoso AKS Ingress"
   ```

5. AKS イングレスコントローラーの証明書を Base64 でエンコードします

   :bulb: 組織から証明書を使用した場合でも、上記の手順で生成した証明書を使用した場合でも、正しく Key Vault に格納するためには、公開証明書（`.crt` または `.cer`）を Base64 エンコードする必要があります。

   ```bash 
   export AKS_INGRESS_CONTROLLER_CERTIFICATE_PFX_BASE64_AKS_BASELINE=$(cat traefik-ingress-internal-aks-ingress-tls.crt traefik-ingress-internal-aks-ingress-tls.key | base64 | tr -d '\n')
   echo AKS_INGRESS_CONTROLLER_CERTIFICATE_PFX_BASE64_AKS_BASELINE: $AKS_INGRESS_CONTROLLER_CERTIFICATE_PFX_BASE64_AKS_BASELINE
   ```

### Save your work in-progress

### 作業の途中経過を保存する

```bash
# aks_baseline.env への環境変数を保存するために、上記の saveenv.sh スクリプトをいつでも実行できます。
./saveenv.sh

# ターミナルセッションがリセットされた場合は、ファイルをソース化して環境変数を再読み込みできます。
```

### 次のステップ

:arrow_forward: [Azure Active Directory 統合のための準備](./03-aad.md)