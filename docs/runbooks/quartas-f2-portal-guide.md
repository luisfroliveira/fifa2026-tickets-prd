# Quartas de Final (F2) — Gateway YARP + identidade dois-mundos

Você coloca **identidade** na frente do FIFA 2026 Tickets: sobe o **gateway YARP** (a porta da frente), faz ele validar **dois mundos de login** com a mesma mecânica (issuer-agnóstico) e, no fim, **migra** usuários antigos (senha bcrypt) para o CIAM **sem apagar nada** — provando que as duas identidades convivem na mesma linha do banco.

| | **Cliente (comprador)** | **Admin (operador)** |
|---|---|---|
| Produto | **Entra External ID** (CIAM) | **Entra ID** (workforce) |
| URL de login | **`<seu-tenant>.ciamlogin.com`** | **`login.microsoftonline.com`** |
| Como entra | cadastro self-service (Google / email+OTP) | conta corporativa que já existe |

> **Story:** [2.11](../stories/2.11.story.md) · **Decisões:** [ADE-007 v1.1](../architecture/ade-007-identity-external-id.md) · [ADE-004 gateway issuer-agnóstico](../architecture/ade-004-yarp-gateway.md) · **Workflow:** [`lab-quartas-de-final.yml`](../../.github/workflows/lab-quartas-de-final.yml)

---

## Preencha os SEUS valores

Este lab reusa os recursos das **Oitavas (F1)** e cria os **novos** das Quartas. Cada aluno tem o próprio fork, subscription e nomes. Anote os **seus** valores aqui antes de começar — todas as fases referenciam estes placeholders.

> 💡 **Sufixo:** escolha um sufixo curto e único (ex.: suas iniciais + número) e use-o em todos os recursos novos. **ACR** só aceita **letras e números** (sem hífen).

| Recurso | Convenção sugerida | Seu valor |
|---|---|---|
| Subscription | `<sua-subscription>` | ____________________ |
| Subscription ID | (GUID) | ____________________ |
| Região | `<sua-regiao>` | ____________________ |
| Resource Group | `<seu-rg>` (reuse o das Oitavas) | ____________________ |
| SQL Server | `<seu-sql-server>` (DB `FIFA2026Tickets`) | ____________________ |
| Function App F1 | `<seu-func>` → `https://<seu-func>.azurewebsites.net` | ____________________ |
| Frontend Web App | `<seu-frontend>` | ____________________ |
| Backend v1 Web App | `<seu-backend>` (intocado — comparação didática) | ____________________ |
| Container Registry (ACR) | `cr<sufixo>` → `cr<sufixo>.azurecr.io` (só letras/números) | ____________________ |
| Container Apps Environment | `cae-<sufixo>` | ____________________ |
| Container App (gateway) | `ca-gateway-<sufixo>` | ____________________ |
| FQDN do gateway | `<gateway-fqdn>` (gerado pelo Azure) | ____________________ |
| Tenant CIAM (nome) | `<seu-tenant>` | ____________________ |
| Domínio inicial CIAM | `<seu-tenant>.onmicrosoft.com` | ____________________ |
| Login do cliente CIAM | `<seu-tenant>.ciamlogin.com` | ____________________ |
| Tenant ID CIAM = `Jwt__CiamTenantId` | `<CiamTenantId>` | ____________________ |
| App Reg SPA (cliente) = `Jwt__CiamClientId` | `student-<iniciais>-v2` | ____________________ |
| App Reg admin (workforce) = `Jwt__AdminClientId` | `student-<iniciais>-admin` | ____________________ |
| Tenant ID workforce = `Jwt__AdminTenantId` | `<AdminTenantId>` | ____________________ |

> 💡 Você terá **DOIS** tenants e **DOIS** client IDs: o **CIAM** (cliente, `*.ciamlogin.com`) e o **workforce** (admin). Anote os dois separadamente — é fácil confundir.

---

## Conceitos (5 min)

1. **Dois mundos, dois produtos, duas URLs de login** (tabela do topo). Analogia: o cliente compra na **bilheteria pública** (qualquer um se cadastra) = External ID; o funcionário entra pela **portaria de serviço** com o **crachá** da empresa = workforce.
2. **O gateway é "issuer-agnóstico":** valida qualquer token **por discovery** (busca a chave pública do emissor numa URL `.well-known`). Aceitar um novo mundo de identidade é **configuração** (trocar a authority), não código novo.
3. **A migração é ADITIVA:** ao "migrar" um usuário antigo pro CIAM, a senha bcrypt **fica intacta**. Você só **adiciona** um ponteiro (`users.entra_oid`). O mesmo humano passa a ter duas credenciais independentes — provar isso é o clímax do lab.

> ⚠️ **Os dois enganos que quebram o lab:**
> - **Domínio do cliente:** login do cliente em `login.microsoftonline.com` (em vez de `ciamlogin.com`) → `AADSTS50011`. `ciamlogin.com` ≠ `microsoftonline.com`.
> - **B2C proibido:** **nunca** use `b2clogin.com` (Azure AD B2C **legado**). Este lab usa **exclusivamente Entra External ID** (`ciamlogin.com`). Se o Portal oferecer B2C, ignore.

---

## Ordem das fases

```
Fase 0  Google (Gemini key — adiantamento p/ o último lab)
Fase 1  Tenant CIAM (External ID) + user flow + Google/OTP
Fase 2  App Reg SPA (cliente) no CIAM            ─┐  daqui saem os
Fase 3  App Reg admin (workforce) + App Role     ─┘  4 GUIDs reais Jwt__*
Fase 4  Migrations            (Actions: acao=migrations)
Fase 5  Gateway: provisionar + GUIDs + smoke     (Actions: acao=gateway)
Fase 6  Frontend             (Actions: acao=frontend, exige VITE_CIAM_*)
Fase 7  Login CIAM e2e (cliente)
Fase 8  Login admin e2e
Fase 9  Migração users v1 → CIAM (hands-on, SQL)
```

> **Regra de ouro:** o Portal/Entra/Google criam e configuram **infra e identidade** à mão; o GitHub Actions é o **último passo** e só publica **código** (gateway, frontend) e **schema** (migrations). Ele **não cria recursos Azure**.

---

## Provisionamento da infra (Fase 5 — à mão, antes do Actions)

A infra do gateway (ACR, Container Apps Environment, Container App) é criada **à mão** (Portal ou `az` no Cloud Shell), **antes** de rodar o workflow. O Actions só publica a imagem depois. Crie com os **seus** nomes (placeholders da tabela acima):

```bash
# ACR (Basic, admin-enabled) → cr<sufixo>.azurecr.io  (só letras/números no nome)
az acr create -g <seu-rg> -n cr<sufixo> --sku Basic --admin-enabled true --location <sua-regiao> -o table

# Container Apps Environment
az containerapp env create -g <seu-rg> -n cae-<sufixo> --location <sua-regiao> -o table

# Container App do gateway (ingress externo, targetPort 8080 — errar aqui = 502)
az containerapp create -g <seu-rg> -n ca-gateway-<sufixo> \
  --environment cae-<sufixo> \
  --image mcr.microsoft.com/k8se/quickstart:latest \
  --target-port 8080 --ingress external --min-replicas 0 --max-replicas 1 -o table

# Conectar o ACR (admin creds) p/ puxar a imagem real (NÃO imprima ACR_PASS em logs compartilhados)
ACR_USER=$(az acr credential show -g <seu-rg> -n cr<sufixo> --query username -o tsv)
ACR_PASS=$(az acr credential show -g <seu-rg> -n cr<sufixo> --query "passwords[0].value" -o tsv)
az containerapp registry set -g <seu-rg> -n ca-gateway-<sufixo> \
  --server cr<sufixo>.azurecr.io --username "$ACR_USER" --password "$ACR_PASS" -o table
```

> A imagem do gateway expõe a **porta 8080** (`src/Fifa2026.V2.Gateway/Dockerfile`: `EXPOSE 8080` + `ASPNETCORE_URLS=http://+:8080`); por isso o **targetPort do ingress = 8080**. O Container App sobe primeiro com imagem placeholder; o workflow `acao=gateway` depois troca pela imagem real SHA-tagged. Depois de criado, anote o **FQDN** (`<gateway-fqdn>`) gerado pelo Azure.

---

## Walkthrough

### Fase 0 — Conta Google do lab (Gemini key)

Adiantamento para o **último lab** (chatbot LLM). Como você vai criar a conta Google de qualquer jeito (Fase 1 usa o Google como IdP opcional), já deixe a key pronta.

1. Crie/abra uma conta **Gmail exclusiva do lab** (ex.: `fifa2026.lab.<iniciais>@gmail.com`), em janela anônima.
2. Acesse **https://aistudio.google.com/apikey** logado nessa conta → aceite os termos.
3. **Create API key → Create API key in new project** → copie a chave e guarde como `GEMINI_API_KEY` (nunca no código). Modelo do lab: `gemini-2.5-flash`.

✅ **Checkpoint:** conta Gmail do lab criada; Gemini key guardada para depois.

---

### Fase 1 — Tenant CIAM (Entra External ID) + user flow

Tudo aqui é no **Microsoft Entra admin center** ([entra.microsoft.com](https://entra.microsoft.com)) — **não** no `portal.azure.com`.

#### 1.1 Criar o tenant External ID (via Azure Subscription)

> ⚠️ Na prática **só aparece "Use Azure Subscription"** — a "30-day free trial" quase nunca é ofertada. Não espere o trial: siga por **Use Azure Subscription**, que não expira em 30 dias e tem **free tier de 50.000 MAU** (um lab fatura ~zero). Confirme o valor vigente em [aka.ms/ExternalIDPricing](https://aka.ms/ExternalIDPricing).

A conta precisa de privilégio para **criar o tenant** + **Owner/Contributor na subscription**. Pode ser necessário registrar o resource provider:

```bash
az provider register -n Microsoft.AzureActiveDirectory
```

1. Em [entra.microsoft.com](https://entra.microsoft.com): **Entra ID → Overview → Manage tenants → `Create`**.
2. **External → Continue → Use Azure Subscription**.
3. Preencha: **Subscription** `<sua-subscription>`; **Resource Group** (reuse `<seu-rg>` ou crie um dedicado); **Location** (⚠️ **não muda depois**); **Nome + domínio** `<seu-tenant>.onmicrosoft.com`.
4. Criação leva **até 30 min** (acompanhe em **Notifications**).
5. Troque para o tenant novo: engrenagem (**Settings**) → **Directories + subscriptions → Switch**. Em **Tenant overview**, anote **Tenant ID** e **Primary domain** (`<seu-tenant>.ciamlogin.com`).

✅ **Checkpoint:** tenant CIAM criado; domínio `*.ciamlogin.com` confirmado; **Tenant ID** anotado (= `Jwt__CiamTenantId` = `<CiamTenantId>`).

#### 1.2 Vincular a subscription (billing)

Logo após criar, o **Tenant Overview** pode mostrar o aviso *"An Azure subscription is required to continue receiving SLA support"*. Isso só significa que o tenant precisa estar vinculado a uma subscription para billing (não é cobrança — o free tier cobre o lab).

- Vá em **Home → Billing**: se o **subscription ID já aparece**, está OK (criar via "Use Azure Subscription" normalmente já vincula) → **nada a fazer**.
- Se **não** aparece: **Click here to upgrade → Add Subscription** → escolha `<sua-subscription>` + resource group → confirme.

✅ **Checkpoint:** **Home → Billing** mostra um subscription ID vinculado.

#### 1.3 Criar o user flow (sign-up/sign-in)

1. **Entra ID → External Identities → User flows → `New user flow`**.
2. **Name** (ex.: `SignUpSignIn`).
3. **Identity providers:** marque **Email Accounts → Email one-time passcode** (OTP — fallback de zero dependência).
4. **User attributes:** escolha o que coletar (email, display name…) → **Create**.

✅ **Checkpoint:** user flow com Email+OTP funcionando (cobre o lab inteiro sem dependência externa).

#### 1.4 Adicionar o Google como Identity Provider — OPCIONAL

> 🟢 **Pule esta seção se quiser.** O **Email OTP** da Fase 1.3 já cobre o lab inteiro, sem nenhuma dependência externa. Só faça se quiser **login social com Google**; caso contrário, vá direto para a **Fase 2**.

**Pré-requisito:** Client ID + Client secret do OAuth client do Google (crie no **[Apêndice 3](#apêndice-3--google-oauth-opcional)**, que usa o seu Tenant ID da 1.1) e volte com os dois valores.

1. No tenant CIAM: **Entra ID → External Identities → All identity providers → aba Built-in**. Ao lado de **Google**, **Configure**.
2. Preencha **Name** (`Google`), cole **Client ID** e **Client secret** → **Save**.
3. Ative no user flow: **External Identities → User flows →** seu flow **→ Settings → Identity providers →** marque **Google → Save**.

✅ **Checkpoint:** na tela de login do CIAM aparece "Continuar com Google", com email+OTP ainda como caminho principal.

---

### Fase 2 — App Registration SPA (cliente) no tenant CIAM

1. No tenant CIAM: **Entra ID → App registrations → `New registration`**.
2. **Name:** `student-<iniciais>-v2` · **Account types:** single-tenant.
3. **Register** → na **Overview**, anote **Application (client) ID** (= `Jwt__CiamClientId` e `VITE_CIAM_CLIENT_ID`).
4. **Authentication → Add a platform → Single-page application** → **Redirect URI:** `http://localhost:5173` **e** a URL de prod (`https://<seu-frontend>.azurewebsites.net`). **NÃO** crie client secret (SPA é público, usa PKCE). Se escolher "Web" por engano, corrija no **Manifest**.
5. Vincule ao user flow: **External Identities → User flows →** o flow **→ Use → Applications → `Add application`** → `student-<iniciais>-v2` → **Select**. (Há um app `b2c-extensions-app` na lista — **não apague**.)

✅ **Checkpoint:** App Reg SPA no CIAM; **client ID** anotado; redirect SPA + vínculo ao user flow. Confirme `*.ciamlogin.com` no topo do Entra admin center (não criar no workforce por engano).

---

### Fase 3 — App Registration admin (workforce) + App Role `Admin`

> Segundo mundo. ⚠️ Troque para o tenant **workforce** (domínio `*.onmicrosoft.com`, **não** `ciamlogin.com`).

1. **Entra ID → App registrations → `New registration`** → Name `student-<iniciais>-admin` → single-tenant → **Register**.
2. Anote **Application (client) ID** (= `Jwt__AdminClientId`) e **Directory (tenant) ID** (= `Jwt__AdminTenantId` = `<AdminTenantId>`, é o **workforce**, diferente do CIAM).
3. **App roles → `Create app role`**: Display `Admin` · Allowed member types **Users/Groups** · **Value `Admin`** · habilitada → **Apply**. (Decisão do owner: uma única App Role.)
4. Atribua: **Enterprise applications →** `student-<iniciais>-admin` **→ Users and groups → `Add user/group`** → seu admin → role `Admin` → **Assign**.

✅ **Checkpoint:** App Reg admin no workforce; App Role `Admin` criada e **atribuída**. Agora você tem os **4 GUIDs reais** `Jwt__*` (CIAM tenant/client + admin tenant/client).

---

### Fase 4 — Migrations do banco

> Das Oitavas você já tinha `phase-01` e `phase-03`; a migration **NOVA das Quartas** é `phase-04-ciam-link.sql` (cria `users.entra_oid` **vazia**). O **preenchimento** é o hands-on da [Fase 9](#fase-9--migração-users-v1--ciam-hands-on--o-clímax) — de propósito, **não** roda no workflow.

Confirme os Secrets/Vars do [Apêndice 1](#apêndice-1--vars-e-secrets-do-github) e **Actions → "Lab Quartas de Final" → Run workflow → `acao = migrations`** → branch `phase-04-quartas`. O workflow abre acesso público + firewall temporário ao SQL (privado), aplica as 3 migrations e **reverte** o acesso ao final (mesmo em falha). Idempotente (pode repetir).

✅ **Checkpoint:** workflow verde; `users.entra_oid` existe (vazia) + índice `UQ_users_entra_oid`.

---

### Fase 5 — Gateway YARP: GUIDs + smoke

> Pré-requisito: infra provisionada (ver [Provisionamento da infra](#provisionamento-da-infra-fase-5--à-mão-antes-do-actions)) e Fases 1–3 feitas (para ter os 4 GUIDs reais).

#### 5.1 App Settings de identidade

O gateway só sobe com as 4 `Jwt__*` presentes (**fail-closed**). As 6 env vars do Container App:

| App Setting | Valor |
|---|---|
| `Jwt__CiamTenantId` | Tenant ID do CIAM (Fase 1.1) = `<CiamTenantId>` |
| `Jwt__CiamClientId` | Client ID da App Reg SPA (Fase 2) |
| `Jwt__AdminTenantId` | Tenant ID do workforce (Fase 3) = `<AdminTenantId>` |
| `Jwt__AdminClientId` | Client ID da App Reg admin (Fase 3) |
| `FunctionAppF1Url` | `https://<seu-func>.azurewebsites.net` (sem ela `/purchase` dá 502) |
| `Gateway__FrontendOrigin` | `https://<seu-frontend>.azurewebsites.net` (CORS restrito ao front) |

> 🔒 **Duplo underscore:** `Jwt:CiamTenantId` em variável de ambiente vira `Jwt__CiamTenantId`. A connection string do SQL **NÃO** vai no gateway (fica na Function).
> **Discovery que o gateway monta:** authority `https://<seu-tenant>.ciamlogin.com/<CiamTenantId>`, issuer `…/v2.0`; admin `https://login.microsoftonline.com/<AdminTenantId>/v2.0`. Validação fail-closed, `ClockSkew=0`, `ValidIssuer`/`ValidAudiences` explícitos.

> 💡 Se você subiu o Container App só para testar a infra antes de ter os GUIDs reais, pode usar **GUIDs placeholder** (válidos em forma) — o gateway sobe e o `401` sem token já funciona. O fluxo com **token real** só fecha com os 4 GUIDs reais.

#### 5.2 Configurar os 4 GUIDs reais

Com os 4 GUIDs reais em mãos (Fases 1–3):

```bash
az containerapp update -g <seu-rg> -n ca-gateway-<sufixo> --set-env-vars \
  "Jwt__CiamTenantId=<CiamTenantId>" \
  "Jwt__CiamClientId=<CLIENT_ID_SPA_CIAM>" \
  "Jwt__AdminTenantId=<AdminTenantId>" \
  "Jwt__AdminClientId=<CLIENT_ID_ADMIN>" -o table
```

> **Portal:** Container App → **Containers → Edit and deploy → Environment variables** → ajuste os 4 valores → **Save** (gera nova revisão).

#### 5.3 Deploy do código do gateway (Actions `acao=gateway`)

O workflow faz `dotnet build/test` (+ projeto de Tests), build & push da imagem no ACR e `az containerapp update --image` (troca o placeholder pela imagem real) + smoke. Imagem: `cr<sufixo>.azurecr.io/gateway:<sha>`. Re-rode (**`acao = gateway`**) só se trocar o código.

> 📝 O cache de borda (AC-6) usa um `XCacheMiddleware`/`IMemoryCache` (o `OutputCache` nativo não captura respostas proxied pelo YARP). Suíte: **17/17**.

#### 5.4 Smoke test

```bash
FQDN="<gateway-fqdn>"
sleep 20   # cold start: min-replicas=0
curl -fsS "https://${FQDN}/health"
# → {"status":"healthy","service":"gateway-yarp"}   (rota anônima)
curl -s -o /dev/null -w '%{http_code}\n' -X POST "https://${FQDN}/purchase" \
  -H "Content-Type: application/json" -d '{"matchId":1,"category":"VIP","userId":1,"quantity":1}'
# → 401   (fail-closed: sem token o gateway recusa)
```

✅ **Checkpoint:** `/health` = 200; `POST /purchase` sem token = **401**. O `401` já funciona **mesmo com GUIDs placeholder** — prova a infra. O fluxo com **token real** exige a 5.2 + Fases 1–3. As políticas (rate-limit 5/min → 429, cache `X-Cache HIT` em GET, CORS) estão no código — demonstre com token real.

---

### Fase 6 — Frontend: authority CIAM + publicar (Actions `acao=frontend`)

> Depende do tenant CIAM (`VITE_CIAM_*`).

1. Garanta **SCM Basic Auth `On`** no Web App do frontend e capture o `AZURE_FRONTEND_PUBLISH_PROFILE` **depois** disso.
2. Configure `VITE_CIAM_AUTHORITY` e `VITE_CIAM_CLIENT_ID` no fork (as demais Vars já existem — [Apêndice 1](#apêndice-1--vars-e-secrets-do-github)).
3. **Run workflow → `acao = frontend`** (`npm ci` + `vite build` com `VITE_CIAM_*` + deploy no Web App).

> ⚠️ Authority em `login.microsoftonline.com` → `AADSTS50011`. Como `ciamlogin.com` é authority "non-AAD", o MSAL exige `knownAuthorities: ['<seu-tenant>.ciamlogin.com']` (o `authV2.ts` já contempla).

✅ **Checkpoint:** frontend publicado com a authority CIAM embutida.

---

### Fase 7 — Login CIAM + smoke ponta-a-ponta (clímax do cliente)

1. Abra o frontend → **Entrar (v2)** → redireciona para `<seu-tenant>.ciamlogin.com`.
2. **Sign-up self-service:** "Continuar com Google" **ou** email + **OTP**.
3. Faça uma compra (`POST /purchase`) — o SPA envia `Authorization: Bearer <token-CIAM>`.
4. Confirme no SQL:
   ```sql
   SELECT TOP 5 id, user_id, entra_oid, status, created_at
   FROM dbo.purchases WHERE entra_oid IS NOT NULL ORDER BY id DESC;
   ```

> 🔎 **GATE — confirme em runtime:** cole o access token em [jwt.ms](https://jwt.ms) e verifique o **formato exato** do `iss` (termina em `…/v2.0`), o `aud` (= seu client ID) e o claim `oid`. É aqui que você trava o formato do issuer CIAM e o `knownAuthorities`. Nunca cole tokens de produção — em sala é trial descartável.

✅ **Checkpoint (AC-11):** login CIAM → gateway valida → `X-Entra-OID` propagado → `purchases.entra_oid` (origem CIAM) gravado ao lado de registros v1.

> ☕ **PONTO DE PAUSA NATURAL** — se dividir em dois encontros, encerre aqui (cliente CIAM real validado pelo gateway = uma aula completa).

---

### Fase 8 — Admin workforce + segundo emissor

> Pré-condição: Fase 3 feita e os `Jwt__Admin*` reais já no gateway (5.2).

1. Logue como admin (authority `https://login.microsoftonline.com/<AdminTenantId>`). Em [jwt.ms](https://jwt.ms): `iss = login.microsoftonline.com/.../v2.0` e `roles: ["Admin"]`.
2. 🧪 Um **cliente CIAM válido** numa rota admin recebe **403** (autenticado, sem a role) — não 401.

✅ **Checkpoint (AC-13):** login admin via workforce com `roles:["Admin"]`, separado do cliente CIAM. Dois mundos coexistindo, validados pela **mesma** mecânica issuer-agnóstica.

---

### Fase 9 — Migração `users` v1 → CIAM (hands-on — o clímax)

> A coluna `users.entra_oid` já existe (Fase 4) **vazia** — você a **preenche** aqui, de forma aditiva.

**9.1 Listar os alvos**
```sql
SELECT id, name, email, entra_oid FROM dbo.users WHERE entra_oid IS NULL ORDER BY id;
```

**9.2 Sign-up no CIAM com o MESMO email do v1** — para cada conta, faça sign-up self-service no CIAM (Fase 7) com **o email idêntico** ao de `users`.

> 💡 A senha **bcrypt NÃO vai** pro CIAM (o External ID não importa hash). O usuário cria credencial nova; o `users.password` bcrypt fica **intacto** no caminho v1.

**9.3 Capturar o `oid` emitido pelo CIAM** — via app (token em jwt.ms/DevTools) **ou** via Portal (Entra admin center → tenant CIAM → **Users** → o usuário → **Object ID**).

**9.4 Vincular o `oid` ao registro v1 (idempotente)**
```sql
UPDATE dbo.users
SET    entra_oid = @oid       -- oid do 9.3
WHERE  email = @email         -- MESMO email do v1
  AND  entra_oid IS NULL;     -- guard de idempotência
```
Idempotência: `WHERE entra_oid IS NULL` (2ª execução = 0 linhas) + índice UNIQUE filtrado `UQ_users_entra_oid` + sign-up nativo do CIAM (email já existente não duplica).

**9.5 Provar a coexistência (o clímax)**
```sql
SELECT u.id AS user_id_v1, u.email,
       CASE WHEN u.password LIKE '$2%' THEN 'bcrypt-presente' ELSE 'sem-bcrypt' END AS credencial_v1,
       u.entra_oid AS oid_ciam_v2,
       CASE WHEN u.password IS NOT NULL AND u.entra_oid IS NOT NULL
            THEN 'COEXISTE (v1 bcrypt + v2 CIAM)'
            WHEN u.entra_oid IS NULL THEN 'so v1 (nao migrou)'
            ELSE 'estado inesperado' END AS status_migracao
FROM dbo.users u WHERE u.email = @email;
```
Esperado: `status_migracao = COEXISTE (v1 bcrypt + v2 CIAM)`.

✅ **Checkpoint (AC-16):** uma linha de `users` com as **duas identidades** vivas lado a lado. Modernização sem destruição, provada em banco.

> **Rollback (aditivo ⇒ trivial):** desfazer um vínculo = `UPDATE dbo.users SET entra_oid = NULL WHERE email = @email;`. Reverter a migration = `DROP INDEX UQ_users_entra_oid ON dbo.users; ALTER TABLE dbo.users DROP COLUMN entra_oid;`. **Nunca** crie backup table no SQL (regra do projeto).

---

## Apêndice 1 — Vars e Secrets do GitHub

Fork → **Settings → Secrets and variables → Actions**. Os **nomes** abaixo são **fixos** (iguais para todos); os **valores** são os **seus** (placeholders da [tabela do topo](#preencha-os-seus-valores)).

### GitHub Variables

| Nome EXATO | Valor (seu) | Usada em (ação) |
|---|---|---|
| `SQL_SERVER` | `<seu-sql-server>` | migrations |
| `RESOURCE_GROUP` | `<seu-rg>` | migrations |
| `ACR_LOGIN_SERVER` | `cr<sufixo>.azurecr.io` | gateway |
| `PHASE04_CONTAINERAPP_NAME` | `ca-gateway-<sufixo>` | gateway |
| `PHASE04_RESOURCE_GROUP` | `<seu-rg>` | gateway |
| `FRONTEND_APP_NAME` | `<seu-frontend>` | frontend |
| `GATEWAY_V2_URL` | `https://<gateway-fqdn>` | frontend (→ `VITE_GATEWAY_V2_URL`) |
| `BACKEND_URL` | `https://<seu-backend>.azurewebsites.net` | frontend |
| `FUNCTION_V2_URL` | `https://<seu-func>.azurewebsites.net` | frontend (→ `VITE_FUNCTION_V2_URL`) |
| `VITE_CIAM_AUTHORITY` | `https://<seu-tenant>.ciamlogin.com/` | frontend |
| `VITE_CIAM_CLIENT_ID` | Client ID da App Reg SPA (Fase 2) | frontend |

> ⚠️ As vars do gateway têm prefixo **`PHASE04_`** (`PHASE04_CONTAINERAPP_NAME`, `PHASE04_RESOURCE_GROUP`) — é exatamente o que o YAML lê; configure assim ou o workflow não encontra. `SQL_SERVER`/`RESOURCE_GROUP` têm fallback para defaults internos do YAML, mas mantê-las explícitas é mais claro.

### GitHub Secrets

| Nome EXATO | Conteúdo | Usada em (ação) |
|---|---|---|
| `AZURE_CREDENTIALS` | JSON do Service Principal com acesso ao RG | migrations + gateway |
| `SQL_CONNECTION_STRING` | connection string ADO.NET do `FIFA2026Tickets` | migrations |
| `AZURE_FRONTEND_PUBLISH_PROFILE` | publish profile do `<seu-frontend>` | frontend |

> **Montar a `SQL_CONNECTION_STRING` (Cloud Shell PowerShell):**
> ```powershell
> $server = "<seu-sql-server>"; $senha = "<senha-adminsql>"
> "Server=$server.database.windows.net,1433;Database=FIFA2026Tickets;User Id=adminsql;Password=$senha;Encrypt=true;TrustServerCertificate=true"
> ```

> 📌 **Sobras da F1 (Oitavas) — NÃO confunda:** existem no fork mas o workflow das Quartas **NÃO lê**. Deixe-as quietas.
> - Variables inertes: `BACKEND_APP_NAME`, `FUNCTION_APP_NAME`
> - Secrets inertes: `AZURE_BACKEND_PUBLISH_PROFILE`, `FUNCTION_PUBLISH_PROFILE`

---

## Apêndice 2 — Provisionamento da infra (resumo)

Os comandos `az` para criar ACR + Container Apps Environment + Container App estão na seção [Provisionamento da infra](#provisionamento-da-infra-fase-5--à-mão-antes-do-actions), com os placeholders dos **seus** recursos. Lembre: **infra à mão primeiro** (Portal/CLI), **Actions só publica código depois**.

> Para o instrutor: os comandos `az` já executados na demo de referência (com os nomes reais do HML) estão no doc interno `_instrutor-quartas-hml.md` — não distribuído aos alunos.

---

## Apêndice 3 — Google OAuth (opcional)

> 🟢 Só faça se for oferecer login social do Google (o **Email OTP** da Fase 1.3 já cobre o lab). Faça **depois** de ter o **Tenant ID** do CIAM (Fase 1.1) — os redirect URIs dependem dele.

A interface atual chama-se **Google Auth Platform** (rótulos legados *APIs & services → OAuth consent screen* entre parênteses).

1. [console.cloud.google.com](https://console.cloud.google.com) (conta do lab) → **New Project** (ex.: `fifa2026-ciam-lab`) → **Create** → selecione-o.
2. **☰ → Google Auth Platform → Branding** *(legado: APIs & services → OAuth consent screen)*. No wizard **Get started**: App name + User support email (Gmail do lab); **Audience = External**; contato → Save.
3. **Audience:** confirme **Publishing status = Testing** e adicione o Gmail do lab em **Test users**.
4. **Branding → Authorized domains:** adicione **`ciamlogin.com`** e **`microsoftonline.com`**.
5. **Clients → Create client** → **Web application** (ex.: `entra-ciam-callback`) → em **Authorized redirect URIs** cole **os 7** abaixo, **trocando** `<tenant-ID>` pelo seu `<CiamTenantId>` (Fase 1.1) e `<tenant-subdomain>` pelo seu subdomínio (`<seu-tenant>`):

```text
https://login.microsoftonline.com
https://login.microsoftonline.com/te/<tenant-ID>/oauth2/authresp
https://login.microsoftonline.com/te/<tenant-subdomain>.onmicrosoft.com/oauth2/authresp
https://<tenant-ID>.ciamlogin.com/<tenant-ID>/federation/oidc/accounts.google.com
https://<tenant-ID>.ciamlogin.com/<tenant-subdomain>.onmicrosoft.com/federation/oidc/accounts.google.com
https://<tenant-subdomain>.ciamlogin.com/<tenant-ID>/federation/oauth2
https://<tenant-subdomain>.ciamlogin.com/<tenant-subdomain>.onmicrosoft.com/federation/oauth2
```

> Cadastre os 7 ou o login falha com `redirect_uri_mismatch`. Link: https://learn.microsoft.com/en-us/entra/external-id/customers/how-to-google-federation-customers

6. **Create** → copie **Client ID** e **Client secret** (o secret só aparece agora por inteiro) → leve para a **Fase 1.4**.

---

## Apêndice 4 — Troubleshooting

| Sintoma | Causa provável | Mitigação |
|---|---|---|
| **502** em toda chamada | targetPort do ingress ≠ **8080** | ingress targetPort = **8080** ([Provisionamento](#provisionamento-da-infra-fase-5--à-mão-antes-do-actions)) |
| **502** só em `/purchase` | `FunctionAppF1Url` ausente/errada | apontar p/ `https://<seu-func>.azurewebsites.net` |
| Container App `Failed`/CrashLoop | `Jwt__*` ausente/vazia/`"common"` (fail-closed) | 4 `Jwt__*` presentes; placeholder serve p/ subir e fazer o 401 |
| `/purchase` dá **200** sem token | gateway não fail-closed (config errada) | revisar `AddJwtBearer`/`Jwt__*`; deveria ser **401** |
| `/health` não responde no 1º hit | cold start (`min-replicas=0`) | aguardar ~20s e repetir |
| `AADSTS50011` no login do cliente | authority com `microsoftonline.com` | `VITE_CIAM_AUTHORITY` = `<seu-tenant>.ciamlogin.com` (Fase 6) |
| MSAL recusa authority "não confiável" | falta `knownAuthorities` | `knownAuthorities: ['<seu-tenant>.ciamlogin.com']` (já no `authV2.ts`) |
| **401 "Invalid issuer"** (cliente/admin) | `Jwt__*` placeholder/errado | trocar pelos 4 GUIDs reais (Fase 5.2) |
| `roles` ausente no token admin | role não atribuída | Enterprise applications → atribuir `Admin` (Fase 3) |
| Cliente CIAM em rota admin dá 401 (esperava 403) | policy fixando esquema | `AdminOnly` só `RequireRole("Admin")` (já no código) |
| `redirect_uri_mismatch` (Google) | redirect URI ≠ callback do Entra | cadastrar **todos** os 7 URIs (Apêndice 3) |
| Vars do gateway "não encontradas" | esqueceu o prefixo `PHASE04_` | usar `PHASE04_CONTAINERAPP_NAME` / `PHASE04_RESOURCE_GROUP` |
| Migrations falham por firewall | SQL privado; runner sem regra | o workflow abre/reverte acesso temporário (já tratado no YAML) |
| Usuário não migra / `so v1` | UPDATE não rodou / email divergente | re-executar UPDATE idempotente (Fase 9.4) |
| Só "Use Azure Subscription" (sem trial) | trial 30d quase nunca é ofertado | seguir por **Use Azure Subscription** — free 50K MAU, não expira (Fase 1.1) |
| Aviso *"Azure subscription is required… SLA"* | tenant sem subscription vinculada | **Home → Billing**: se houver subscription ID, OK; senão **Add Subscription** (Fase 1.2) |
| "Insufficient privileges" ao criar o tenant | conta sem Owner / RP não registrado | conta **Global Admin + Owner** + `az provider register -n Microsoft.AzureActiveDirectory` |
