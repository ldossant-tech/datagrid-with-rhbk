# Integração entre Red Hat Data Grid 8.5 e Red Hat Build of Keycloak

**Data Grid** 8.5.14

**Red Hat build of Keycloak Operator** 26.6.4

# Objetivo

Este documento descreve o processo completo para integrar o Red Hat Data Grid 8.5 com o Red Hat Build of Keycloak (RHBK), utilizando OpenID Connect (OIDC) para autenticação do Console e OAuth2 Token Introspection para validação dos Access Tokens utilizados nas chamadas REST.

Ao final deste procedimento será possível:

- Autenticar usuários no Console do Data Grid através do RHBK;
- Controlar permissões utilizando Realm Roles e Groups;
- Centralizar a gestão de identidades no Keycloak;
- Eliminar usuários locais do Data Grid.

---

# Arquitetura

```
                         Login

                   +-------------+
                   |   Browser   |
                   +------+------+
                          |
                          | Authorization Code Flow
                          |
                          ▼
              +------------------------+
              | Red Hat Build of       |
              | Keycloak (RHBK)        |
              +-----------+------------+
                          |
                    Access Token
                          |
                          ▼
              +------------------------+
              | Data Grid Console      |
              +-----------+------------+
                          |
            Authorization: Bearer Token
                          |
                          ▼
              +------------------------+
              | Data Grid Server       |
              | OAuth2 Introspection   |
              +-----------+------------+
                          |
                          ▼
              +------------------------+
              | Red Hat Build of       |
              | Keycloak               |
              +------------------------+
```

---

# Fluxo de autenticação

A integração utiliza dois clients distintos.

## datagrid-console

Responsável pela autenticação do usuário utilizando Authorization Code Flow.

É utilizado exclusivamente pelo Console Web.

## datagrid-server

Responsável por validar os Access Tokens através do endpoint OAuth2 Token Introspection.

Esse client nunca realiza login de usuários.

---

# Pré-requisitos

- OpenShift instalado
- Data Grid Operator
- Red Hat Build of Keycloak Operator
- PostgreSQL
- Namespace "datagrid”

---

# Instalação

## Instalar os Operators

- Data Grid Operator
- Red Hat Build of Keycloak Operator

---

## Criar a instância do Data Grid

```bash
oc apply -f infinispan.yaml
```

---

## Criar o banco PostgreSQL

```bash
oc apply -f postgresql.yaml
```

---

## Criar Secret do banco

```bash
oc apply -f keycloak-db-secret.yaml 
```

---

## Criar a instância do Keycloak

```bash
oc apply -f keycloak.yaml
```

---

# Configuração do Keycloak

## Criar Realm

```
Realm:
datagrid
```

---

# Configuração do client datagrid-console

O client **`datagrid-console`** representa a interface web do Red Hat Data Grid e é responsável por autenticar os usuários no Red Hat Build of Keycloak (RHBK).

Quando um usuário acessa o Console, ele é redirecionado ao RHBK para realizar o login utilizando o protocolo OpenID Connect (OIDC). Após a autenticação, o RHBK emite um Access Token, que será utilizado pelo Console para realizar chamadas aos endpoints REST do Data Grid.

Por esse motivo, esse client é configurado como **público (Public Client)**, pois executa diretamente no navegador e não deve armazenar um **Client Secret**.

**Responsabilidades:**

- Iniciar o fluxo de autenticação (Authorization Code Flow);
- Autenticar o usuário no RHBK;
- Receber o Access Token;
- Enviar o Access Token nas chamadas REST ao Data Grid.

```
Client ID:
datagrid-console

Client Type:
OpenID Connect
```

Configuração:

| Configuração | Valor |
| --- | --- |
| Client Authentication | Off |
| Authorization | Off |
| Standard Flow | On |
| Direct Access Grants | Off |
| Implicit Flow | Off |

URLs

```
Root URL

https://<DATAGRID-CONSOLE>

Home URL

https://<DATAGRID-CONSOLE>

Valid Redirect URIs

https://<DATAGRID-CONSOLE>/*

Valid Post Logout Redirect URIs

https://<DATAGRID-CONSOLE>/*

Web Origins

https://<DATAGRID-CONSOLE>
```

---

# Configuração do client datagrid-server

O client **`datagrid-server`** representa o próprio servidor do Red Hat Data Grid perante o RHBK.

Diferentemente do `datagrid-console`, esse client **não realiza login de usuários**. Sua função é permitir que o Data Grid valide os Access Tokens recebidos do Console através do endpoint **OAuth2 Token Introspection** do RHBK.

Para isso, o Data Grid autentica-se no RHBK utilizando um **Client ID** e um **Client Secret**, motivo pelo qual esse client deve ser configurado como **Confidential Client**.

Essa abordagem evita que o Client Secret seja exposto ao navegador, mantendo a validação dos tokens restrita à comunicação segura entre servidores.

**Responsabilidades:**

- Autenticar o Data Grid no endpoint de introspecção do RHBK;
- Validar os Access Tokens recebidos do Console;
- Permitir que o Data Grid obtenha as informações do usuário autenticado antes de autorizar o acesso aos recursos.

```
Client ID

datagrid-server

Client Type

OpenID Connect
```

Configuração

| Configuração | Valor |
| --- | --- |
| Client Authentication | On |
| Authorization | Off |
| Standard Flow | Off |
| Direct Access Grants | Off |
| Implicit Flow | Off |
| Service Accounts | Off |

Depois acesse

```
Clients

→ datagrid-server

→ Credentials
```

Copie o Client Secret.

---

# Criar Secret do Data Grid

```bash
oc create secret generic datagrid-rhbk-secret \
  --from-literal=client-secret='<CLIENT_SECRET>'
```

---

# Configuração do Data Grid

Criar o ConfigMap contendo o Token Realm.

(Adicionar aqui o XML completo)

Aplicar:

```bash
oc apply -f datagrid-rhbk-config.yaml
```

---

# Atualizar a instância do Data Grid

Adicionar:

```yaml
configMapName: datagrid-rhbk-config
```

Adicionar:

```yaml
container:
  env:
  - name: DATAGRID_RHBK_CLIENT_SECRET
    valueFrom:
      secretKeyRef:
        name: datagrid-rhbk-secret
        key: client-secret
```

Aplicar:

```bash
oc apply -f infinispan.yaml
```

---
# Configuração de Groups no `datagrid-console`

O objetivo desta configuração é fazer com que o **Red Hat Build of Keycloak (RHBK)** inclua os grupos associados ao usuário no **Access Token** emitido para o client `datagrid-console`.

O resultado esperado no token é:

```json
"groups": [
  "admin"
]
```

## 1. Acessar o Realm

No console administrativo do RHBK, acesse o realm utilizado pela integração com o Data Grid:

```text
datagrid
```

## 2. Acessar o client `datagrid-console`

No menu lateral, acesse:

```text
Clients
└── datagrid-console
```

## 3. Acessar o Client Scope dedicado

Dentro do client `datagrid-console`, acesse:

```text
Client scopes
```

Selecione o Client Scope dedicado:

```text
datagrid-console-dedicated
```

Esse Client Scope é criado automaticamente pelo RHBK especificamente para o client `datagrid-console`.

## 4. Criar o Group Membership Mapper

Dentro do `datagrid-console-dedicated`, acesse:

```text
Mappers
└── Configure a new mapper
```

Selecione o tipo de mapper:

```text
Group Membership
```

## 5. Configurar o Mapper

Configure o mapper com os seguintes valores:

| Propriedade             | Valor    |
| ----------------------- | -------- |
| **Name**                | `groups` |
| **Token Claim Name**    | `groups` |
| **Full group path**     | `Off`    |
| **Add to ID token**     | `On`     |
| **Add to access token** | `On`     |
| **Add to userinfo**     | `On`     |

Caso a versão do RHBK disponibilize a opção:

```text
Add to token introspection
```

habilite-a:

```text
Add to token introspection: On
```

## 6. Configurar `Full group path`

A opção **Full group path** deve permanecer desabilitada:

```text
Full group path: Off
```

Dessa forma, caso o usuário pertença ao grupo `admin`, o Access Token apresentará:

```json
"groups": [
  "admin"
]
```

Em vez de:

```json
"groups": [
  "/admin"
]
```

Isso é importante porque o nome do grupo recebido no token deverá corresponder ao nome da **role existente no Data Grid**.

Exemplo:

```text
RHBK Group
   admin
     │
     ▼
Access Token
"groups": ["admin"]
     │
     ▼
Data Grid
Role: admin
```

## 7. Salvar a configuração

Após preencher todos os campos, clique em **Save** para salvar o mapper.

A partir desse momento, novos Access Tokens emitidos pelo `datagrid-console` poderão incluir o claim `groups` com os grupos associados ao usuário.

---

# Configuração do Audience Mapper

O **Audience** é uma informação adicionada ao Access Token que indica **quais aplicações estão autorizadas a consumir esse token**.

Nesta integração, o Access Token é emitido para o **`datagrid-console`**, porém também precisa ser aceito pelo **`datagrid-server`**, que é responsável por validar o token através do endpoint de OAuth2 Token Introspection.

Por isso, é necessário criar um **Audience Mapper**, adicionando o **`datagrid-server`** como audiência do token.

Sem esse mapper, o login ocorre normalmente, porém todas as chamadas REST retornam:

```
401 Unauthorized
```

Criar o mapper:

```
Clients

→ datagrid-console

→ Client Scopes

→ datagrid-console-dedicated

→ Configure a new mapper

→ Audience
```

Configuração:

| Campo | Valor |
| --- | --- |
| Name | datagrid-server-audience |
| Included Client Audience | datagrid-server |
| Add to Access Token | On |
| Add to Lightweight Token | On |
| Add to Token Introspection | On |

---

# Configuração das Roles

Criar Realm Roles

```
admin

application
```

---

# Configuração dos Groups

Criar:

```
admin

application
```

Realizar o Role Mapping correspondente.

---

# Criar Usuários

Criar:

```
admin

application

observer
```

Associar cada usuário ao Group correspondente.

---

# Validação

Após a autenticação:

✔ Console acessível

✔ Nenhuma aba retorna HTTP 401

✔ Cache Manager carregado

✔ Estatísticas disponíveis

✔ Health disponível

---

# Troubleshooting

## Login entra em loop

Verificar:

- Redirect URI
- Web Origins
- auth-server-url
- CSP

---

## 401 Unauthorized em todas as abas

Verificar:

- Audience Mapper
- Client Secret
- OAuth2 Introspection
- Token Realm
- URL de Introspection

---

## Variável não substituída

Erro

```
Property env.DATAGRID_RHBK_CLIENT_SECRET could not be replaced
```

Verificar:

```
spec.container.env
```

na instância do Infinispan.

---

# Referências

## Referências Oficiais

- [Red Hat Data Grid 8.5 — Data Grid Operator Guide](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html-single/data_grid_operator_guide/index)  
  Referência para configuração do recurso `Infinispan`, autenticação, Secrets, autorização, roles e permissões no Data Grid Operator.

- [Red Hat Data Grid 8.5 — Token Realms](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_server_guide/security-realms)  
  Referência para configuração de `token-realm`, autenticação baseada em tokens e `oauth2-introspection`.

- [Red Hat Build of Keycloak 26.6 — Server Administration Guide](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html-single/server_administration_guide/index)  
  Referência para configuração de Users, Groups, Roles, Client Scopes, Protocol Mappers e demais recursos administrativos do RHBK.

- [Red Hat Build of Keycloak 26.6 — OIDC Clients, Audience e Token Introspection](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.6/html/server_administration_guide/assembly-managing-clients_server_administration_guide)  
  Referência para configuração de clients OIDC, claim `aud`, Audience Mapper e validação de audience durante OAuth 2.0 Token Introspection.

- [Red Hat Data Grid 8.5 — Security Authorization with Role-Based Access Control](https://docs.redhat.com/en/documentation/red_hat_data_grid/8.5/html/data_grid_security_guide/security-authorization)  
  Referência para RBAC no Data Grid, incluindo roles padrão (`admin`, `deployer`, `application`, `observer` e `monitor`) e suas respectivas permissões.

> **Aviso:** Este repositório é uma recomendação técnica para demonstrar um laboratório de integração entre o Red Hat Data Grid e o Red Hat Build of Keycloak (RHBK), utilizando autenticação OIDC, OAuth 2.0 Token Introspection e controle de acesso baseado em roles. Ele não substitui a documentação oficial da Red Hat. Para implantações em ambiente produtivo, siga sempre as documentações oficiais, a matriz de suporte vigente, as políticas internas de segurança e as recomendações aplicáveis às versões do Red Hat Data Grid, RHBK, OpenShift e Operators instalados.
