# Cenário 1 — autenticar e autorizar a Console do Data Grid com RHBK

Este roteiro substitui o login local da Console do Red Hat Data Grid por login OIDC no Red Hat build of Keycloak (RHBK). Ao final:

- o usuário abre a Console do Data Grid e é redirecionado ao RHBK;
- `dg-admin` consegue administrar o Data Grid;
- `dg-observer` consegue consultar caches e métricas, mas não alterá-los;
- remover um papel no RHBK e emitir uma nova sessão altera o acesso no Data Grid.

O roteiro foi preparado para o ambiente verificado em 21/07/2026: OpenShift 4.20, Data Grid Operator 8.6.4, Data Grid Server 8.6.1-2, namespace `datagrid` e cluster `infinispan`.

> **Escopo:** configuração de laboratório. O PostgreSQL abaixo tem uma réplica e credenciais simples. Para produção, use banco gerenciado/suportado, backup, TLS ponta a ponta, secrets gerenciados, alta disponibilidade e políticas de rede.

## 1. Pré-requisitos e variáveis

É necessário ser administrador do cluster para instalar o Operator do RHBK. Faça login e defina as variáveis na mesma sessão do terminal:

```bash
export DG_NS=datagrid
export DG_NAME=infinispan
export KC_NS=rhbk
export KC_NAME=rhbk
export KC_REALM=datagrid
export APPS_DOMAIN="$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')"
export KC_HOST="${KC_NAME}-${KC_NS}.${APPS_DOMAIN}"
export DG_HOST="$(oc get route infinispan-external -n "${DG_NS}" -o jsonpath='{.spec.host}')"

printf 'RHBK:      https://%s\nData Grid: https://%s\n' "${KC_HOST}" "${DG_HOST}"
```

Não coloque senhas nem client secrets em arquivos versionados.

## 2. Instalar o RHBK Operator 26.6

O catálogo deste cluster oferece o canal `stable-v26.6`. Crie o namespace e a assinatura:

```bash
oc new-project "${KC_NS}"

oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: rhbk
  namespace: rhbk
spec:
  targetNamespaces:
    - rhbk
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhbk-operator
  namespace: rhbk
spec:
  channel: stable-v26.6
  installPlanApproval: Automatic
  name: rhbk-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

oc wait --for=jsonpath='{.status.phase}'=Succeeded \
  csv -l operators.coreos.com/rhbk-operator.rhbk -n "${KC_NS}" --timeout=10m
oc get csv -n "${KC_NS}"
```

Se o Operator já estiver instalado em outro namespace e observando o cluster inteiro, não crie uma segunda assinatura; apenas confirme que o CRD existe:

```bash
oc get crd keycloaks.k8s.keycloak.org
```

## 3. Subir PostgreSQL para o laboratório

Gere senhas e crie o Secret sem imprimi-las:

```bash
export KC_DB_PASSWORD="$(openssl rand -base64 30 | tr -d '\n')"

oc create secret generic keycloak-db-secret -n "${KC_NS}" \
  --from-literal=username=keycloak \
  --from-literal=password="${KC_DB_PASSWORD}" \
  --dry-run=client -o yaml | oc apply -f -

unset KC_DB_PASSWORD
```

Crie o PostgreSQL. A imagem abaixo é adequada ao lab; confirme uma imagem corporativa homologada antes de usar em produção.

```bash
oc apply -n "${KC_NS}" -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: keycloak-postgres
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak-postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak-postgres
  template:
    metadata:
      labels:
        app: keycloak-postgres
    spec:
      containers:
        - name: postgres
          image: registry.redhat.io/rhel9/postgresql-16:latest
          env:
            - name: POSTGRESQL_DATABASE
              value: keycloak
            - name: POSTGRESQL_USER
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-secret
                  key: username
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: keycloak-db-secret
                  key: password
          ports:
            - name: postgres
              containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
          readinessProbe:
            exec:
              command: [/bin/sh, -c, pg_isready -U "$POSTGRESQL_USER" -d "$POSTGRESQL_DATABASE"]
            initialDelaySeconds: 10
            periodSeconds: 10
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: keycloak-postgres
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak-postgres
spec:
  selector:
    app: keycloak-postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: postgres
EOF

oc rollout status deployment/keycloak-postgres -n "${KC_NS}" --timeout=5m
```

## 4. Criar a instância do RHBK

Neste lab, o RHBK recebe HTTP somente dentro do cluster. Uma Route `edge` fornece HTTPS ao navegador. Isso também permite que o Data Grid use a URL interna para introspecção sem precisar importar a CA da Route.

```bash
cat <<EOF | oc apply -f -
apiVersion: k8s.keycloak.org/v2beta1
kind: Keycloak
metadata:
  name: ${KC_NAME}
  namespace: ${KC_NS}
spec:
  instances: 1
  db:
    vendor: postgres
    host: keycloak-postgres
    database: keycloak
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  http:
    httpEnabled: true
  hostname:
    hostname: https://${KC_HOST}
  proxy:
    headers: xforwarded
  ingress:
    enabled: false
EOF

oc wait --for=condition=Ready keycloak/${KC_NAME} -n "${KC_NS}" --timeout=15m

oc create route edge "${KC_NAME}" -n "${KC_NS}" \
  --service="${KC_NAME}-service" \
  --port=http \
  --hostname="${KC_HOST}" \
  --insecure-policy=Redirect \
  --dry-run=client -o yaml | oc apply -f -

curl -fsS "https://${KC_HOST}/realms/master/.well-known/openid-configuration" >/dev/null
echo "RHBK disponível em https://${KC_HOST}"
```

Caso o campo `spec.proxy.headers` não exista no CRD instalado, remova o bloco `proxy` e acrescente em `spec.additionalOptions`:

```yaml
additionalOptions:
  - name: proxy-headers
    value: xforwarded
```

## 5. Acessar a administração do RHBK

O Operator gera um administrador temporário:

```bash
oc get secret "${KC_NAME}-initial-admin" -n "${KC_NS}" \
  -o jsonpath='{.data.username}' | base64 -d; echo
oc get secret "${KC_NAME}-initial-admin" -n "${KC_NS}" \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Abra `https://${KC_HOST}/admin`, entre com essas credenciais e troque a senha temporária. Para produção, crie uma conta administrativa nominal e habilite MFA.

## 6. Configurar realm, papéis, usuários e clientes

Na Admin Console do RHBK:

1. Clique em **Create realm**, informe `datagrid` e confirme.
2. Em **Realm roles**, crie os papéis `admin` e `observer`. Os nomes devem coincidir exatamente com os papéis do Data Grid.
3. Em **Users**, crie `dg-admin`; em **Credentials**, defina uma senha não temporária; em **Role mapping**, atribua `admin`.
4. Crie `dg-observer` da mesma forma e atribua `observer`.

Crie o cliente público usado pelo navegador:

1. Vá para **Clients > Create client**.
2. Selecione **OpenID Connect** e use Client ID `infinispan-console`.
3. Deixe **Client authentication** desabilitado e **Standard flow** habilitado.
4. Em **Valid redirect URIs**, adicione `https://<DG_HOST>/*`.
5. Em **Valid post logout redirect URIs**, adicione `https://<DG_HOST>/*`.
6. Em **Web origins**, adicione `https://<DG_HOST>`.

Substitua `<DG_HOST>` pelo valor exibido no passo 1, sem barras finais.

Crie o cliente confidencial usado pelo Data Grid para introspecção:

1. Crie outro cliente OIDC com Client ID `infinispan-server`.
2. Habilite **Client authentication**.
3. Desabilite **Standard flow** e **Direct access grants**; o cliente existe somente para autenticar a chamada de introspecção.
4. Salve e copie o valor em **Credentials > Client secret**.

Verifique no client scope `roles` que o mapper de realm roles adiciona `realm_access.roles` ao access token. Essa é a configuração padrão, mas é importante não removê-la.

## 7. Guardar o client secret no OpenShift

Cole o client secret apenas no prompt abaixo:

```bash
read -rsp 'Client secret de infinispan-server: ' ISPN_OIDC_SECRET; echo
oc create secret generic infinispan-oidc-secret -n "${DG_NS}" \
  --from-literal=infinispan_server_secret="${ISPN_OIDC_SECRET}" \
  --dry-run=client -o yaml | oc apply -f -
unset ISPN_OIDC_SECRET
```

O Data Grid Operator transforma esse Secret em um credential store chamado `credentials`.

## 8. Configurar o token realm no Data Grid

Crie o ConfigMap. `authServerUrl` precisa ser a URL pública, pois é enviada ao navegador. `introspectionUrl` usa o Service interno do RHBK.

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: infinispan-oidc-config
  namespace: ${DG_NS}
data:
  infinispan-config.yaml: |-
    infinispan:
      server:
        security:
          securityRealms:
            - name: default
              tokenRealm:
                name: ${KC_REALM}
                authServerUrl: https://${KC_HOST}
                clientId: infinispan-console
                oauth2Introspection:
                  clientId: infinispan-server
                  introspectionUrl: http://${KC_NAME}-service.${KC_NS}.svc:8080/realms/${KC_REALM}/protocol/openid-connect/token/introspect
                  credentialReference:
                    store: credentials
                    alias: infinispan_server_secret
        endpoints:
          socketBinding: default
          securityRealm: default
EOF
```

Associe o ConfigMap, o credential store e habilite RBAC no CR existente. Esse comando provoca atualização/reinício dos pods:

```bash
oc patch infinispan "${DG_NAME}" -n "${DG_NS}" --type=merge -p "
spec:
  configMapName: infinispan-oidc-config
  security:
    endpointAuthentication: true
    credentialStoreSecretName: infinispan-oidc-secret
    authorization:
      enabled: true
"

oc wait --for=condition=WellFormed infinispan/${DG_NAME} -n "${DG_NS}" --timeout=15m
oc get pods -n "${DG_NS}" -l infinispan_cr="${DG_NAME}" -w
```

Encerre o `-w` com `Ctrl+C` quando os dois pods estiverem `1/1 Running`.

## 9. Validar antes de abrir o navegador

Confirme que a Console anuncia OIDC:

```bash
curl -ks "https://${DG_HOST}/rest/v2/login?action=config" | jq .
```

O retorno esperado contém valores equivalentes a:

```json
{
  "mode": "OIDC",
  "clientId": "infinispan-console",
  "realm": "datagrid",
  "ready": "true"
}
```

Se `ready` não for `true`, examine:

```bash
oc logs -n "${DG_NS}" infinispan-0 --since=10m
oc logs -n "${KC_NS}" -l app=keycloak --since=10m
```

## 10. Demonstrar as permissões na Console

1. Abra `https://${DG_HOST}/console` em uma janela anônima.
2. Confirme o redirecionamento para o realm `datagrid` no RHBK.
3. Entre como `dg-admin`: caches, configuração e operações administrativas devem estar disponíveis.
4. Saia completamente da Console e do RHBK, ou use outra janela anônima.
5. Entre como `dg-observer`: leitura e monitoramento devem funcionar; criação, alteração e remoção devem ser negadas.
6. No RHBK, remova o papel `observer`, encerre a sessão do usuário em **Users > dg-observer > Sessions** e faça login novamente. O acesso passa a ser negado.

Papéis são carregados no token. Alterar o mapeamento no RHBK não modifica um access token já emitido; é necessário obter uma nova sessão/token ou revogar a sessão existente.

## 11. Diagnóstico rápido

### Loop de redirecionamento ou URI inválida

Confira se a URI está exatamente como a Route atual:

```bash
oc get route infinispan-external -n "${DG_NS}" -o jsonpath='https://{.spec.host}/console{"\n"}'
```

Revise **Valid redirect URIs** e **Web origins** do cliente `infinispan-console`.

### Login funciona, mas tudo retorna Unauthorized

Confirme que o usuário recebeu `admin` ou `observer` como **realm role**, não apenas como client role. Saia e entre novamente para obter um token novo.

### Introspecção falha

Teste a descoberta a partir de um pod do Data Grid:

```bash
oc exec -n "${DG_NS}" infinispan-0 -- \
  curl -fsS "http://${KC_NAME}-service.${KC_NS}.svc:8080/realms/${KC_REALM}/.well-known/openid-configuration"
```

Confira o nome/porta do Service com `oc get svc -n "${KC_NS}"` e confira se o secret atual do cliente é o mesmo armazenado em `infinispan-oidc-secret`.

### Data Grid não volta a `WellFormed`

```bash
oc describe infinispan "${DG_NAME}" -n "${DG_NS}"
oc logs -n "${DG_NS}" infinispan-0 --since=15m
oc get configmap infinispan-oidc-config -n "${DG_NS}" -o yaml
```

## 12. Critérios de sucesso

- `/rest/v2/login?action=config` informa `OIDC` e `ready=true`.
- A Console redireciona para o RHBK e retorna ao Data Grid após o login.
- `dg-admin` administra o cluster.
- `dg-observer` consulta, mas não altera recursos.
- Um usuário sem papel recebe acesso negado.
- Nenhum client secret está armazenado no ConfigMap ou no repositório.

## Referências oficiais

- [Red Hat Data Grid 8.6 — Token realms e mecanismos OIDC](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.6/html-single/data_grid_security_guide/)
- [Red Hat Data Grid 8.6 — Configuração customizada com o Operator](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.6/html-single/data_grid_operator_guide/index)
- [RHBK 26.6 — Operator Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html-single/operator_guide/index)
- [RHBK 26.6 — Server Administration Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html-single/server_administration_guide/index)

