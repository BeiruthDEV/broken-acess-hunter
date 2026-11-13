# SECURITY PLAYBOOK

**Auditoria, hardening e arquitetura segura para qualquer aplicação web/SaaS, em qualquer linguagem.**

Este documento é uma referência de segurança aplicável a qualquer projeto que tenha um cliente (web, mobile, CLI, integração) e um servidor. Use como base para auditoria, refatoração e checklist pré-deploy.

---

## ÍNDICE

- [Parte 1 — Regras Invioláveis](#parte-1--regras-invioláveis)
- [Parte 2 — Modelo de Confiança](#parte-2--modelo-de-confiança)
- [Parte 3 — Princípios de Arquitetura Segura](#parte-3--princípios-de-arquitetura-segura)
- [Parte 4 — Threat Model](#parte-4--threat-model)
- [Parte 5 — Domínios Técnicos](#parte-5--domínios-técnicos)
  - [5.1 Autenticação](#51-autenticação)
  - [5.2 Autorização](#52-autorização)
  - [5.3 Isolamento entre usuários e tenants](#53-isolamento-entre-usuários-e-tenants)
  - [5.4 Validação de entrada](#54-validação-de-entrada)
  - [5.5 Sanitização de saída](#55-sanitização-de-saída)
  - [5.6 Sessão, cookies e CSRF](#56-sessão-cookies-e-csrf)
  - [5.7 Gestão de segredos](#57-gestão-de-segredos)
  - [5.8 Criptografia](#58-criptografia)
  - [5.9 Upload de arquivos](#59-upload-de-arquivos)
  - [5.10 Pagamentos e cobrança](#510-pagamentos-e-cobrança)
  - [5.11 Limites de uso e planos](#511-limites-de-uso-e-planos)
  - [5.12 Rate limiting e proteção contra abuso](#512-rate-limiting-e-proteção-contra-abuso)
  - [5.13 Logging, auditoria e observabilidade](#513-logging-auditoria-e-observabilidade)
  - [5.14 Tratamento de erros](#514-tratamento-de-erros)
  - [5.15 Configuração de servidor e CORS](#515-configuração-de-servidor-e-cors)
  - [5.16 Dependências e supply chain](#516-dependências-e-supply-chain)
  - [5.17 Negação de serviço](#517-negação-de-serviço)
  - [5.18 SSRF, XXE e ataques server-side](#518-ssrf-xxe-e-ataques-server-side)
  - [5.19 Race conditions e idempotência](#519-race-conditions-e-idempotência)
  - [5.20 Webhooks recebidos](#520-webhooks-recebidos)
  - [5.21 Conformidade legal e proteção de dados](#521-conformidade-legal-e-proteção-de-dados)
- [Parte 6 — Anti-padrões](#parte-6--anti-padrões)
- [Parte 7 — Testes Adversariais Obrigatórios](#parte-7--testes-adversariais-obrigatórios)
- [Parte 8 — Checklist Pré-Deploy](#parte-8--checklist-pré-deploy)
- [Parte 9 — Critérios de Aceite](#parte-9--critérios-de-aceite)
- [Apêndice A — Glossário de Severidade](#apêndice-a--glossário-de-severidade)
- [Apêndice B — Referências Externas](#apêndice-b--referências-externas)

---

# PARTE 1 — REGRAS INVIOLÁVEIS

Estas regras não são sugestões. Não são boas práticas. Não são "depende". São fundamentos. Se a aplicação viola qualquer uma delas, ela é insegura por definição, independentemente de quanto código exista por cima.

## REGRA ZERO — O FRONTEND NUNCA É FONTE DE VERDADE

**O frontend é hostil até prova em contrário.** Toda informação que sai do frontend pode estar alterada. DevTools, proxies, scripts injetados, extensões maliciosas, Postman, Burp, requests forjados — tudo isso é trivial. Trate o frontend exatamente como você trataria um usuário desconhecido enviando requests pela primeira vez.

### O frontend PODE
- Renderizar UI.
- Exibir estados retornados pelo servidor.
- Enviar inputs do usuário para o servidor.
- Chamar APIs autenticadas.
- Mostrar ou esconder componentes por conveniência visual.
- Manter cache temporário para fluidez de UX.
- Validar dados *antes* de enviar (como conveniência, nunca como segurança).

### O frontend NUNCA PODE DECIDIR
- Se o usuário está autenticado de verdade.
- Quem o usuário é.
- Qual o papel (role) do usuário.
- Qual o plano do usuário.
- Quais permissões o usuário tem.
- Qual `userId` será usado nas queries.
- Quais dados o usuário pode acessar.
- Se uma rota privada pode ser acessada.
- Se uma feature paga está liberada.
- Qual preço será cobrado.
- Qual valor (`amount`) será processado.
- Quais limites de uso o usuário tem.
- Se um upload deve ser aceito.
- Se uma transação pertence ao usuário.
- Se o usuário pode exportar dados.
- Se o usuário é admin.

**Cada item dessa lista, sem exceção, é decisão do servidor.**

## REGRA UM — DENY BY DEFAULT

Toda rota é privada até ser explicitamente marcada como pública. Não o contrário.

- Sem token válido → `401 Unauthorized`.
- Com token válido mas sem permissão → `403 Forbidden` (ou `404` quando revelar existência é em si um vazamento).
- Endpoint novo criado sem proteção explícita → bug de segurança, não feature.

## REGRA DOIS — O SERVIDOR DERIVA, NÃO RECEBE

O servidor **deriva** identidade, plano, role e propriedade do estado que ele controla (banco de dados, sessão server-side, token assinado por ele).

O servidor **nunca recebe** essas informações do cliente como fato.

```
ERRADO:  GET /api/transactions?userId=123
CERTO:   GET /api/transactions
         Authorization: Bearer <token>
         → servidor extrai user_id do token verificado
```

```
ERRADO:  POST /api/checkout { "amount": 0.01, "plan": "premium" }
CERTO:   POST /api/checkout { "planKey": "premium_monthly" }
         → servidor resolve preço internamente
```

```
ERRADO:  if (user.role === "admin") { ... }   ← user.role veio do client
CERTO:   if (server_verified_role === "admin") { ... }
```

## REGRA TRÊS — OWNERSHIP É OBRIGATÓRIO

Toda entidade que pertence a alguém tem campo de propriedade no banco. Toda query que lê ou modifica essa entidade filtra por essa propriedade. **Sem exceção sem justificativa documentada e revisada.**

```
ERRADO:  SELECT * FROM transactions WHERE id = ?
CERTO:   SELECT * FROM transactions WHERE id = ? AND user_id = ?
```

## REGRA QUATRO — NENHUM SEGREDO VAI PARA O CLIENTE

- Nenhuma chave de API.
- Nenhum token de provedor externo.
- Nenhuma senha de banco de dados.
- Nenhum segredo de assinatura.
- Nenhum endpoint interno que dependa de obscuridade.

Se está no bundle JavaScript, está público. Se está em URL, está em log. Se está em log que vai pro frontend, está público.

## REGRA CINCO — ERROS NÃO VAZAM ARQUITETURA

Em produção:
- Sem stack trace.
- Sem nome de tabela.
- Sem path interno.
- Sem versão de framework.
- Sem mensagem que distinga "usuário não existe" de "senha errada".
- Sem mensagem que distinga "recurso não existe" de "recurso de outro usuário".

## REGRA SEIS — DEFESA EM PROFUNDIDADE

Toda decisão crítica é verificada em mais de uma camada. Se o middleware falha, o controller pega. Se o controller falha, a query pega. Se a query falha, a constraint do banco pega.

Camada única de defesa é defesa única de falha.

## REGRA SETE — SEM TEATRO DE SEGURANÇA

- Header cosmético que não muda comportamento real → remover.
- Validação no cliente sem validação correspondente no servidor → reforçar no servidor.
- "Criptografia" que só ofusca → remover ou trocar por criptografia real.
- ID "complicado" como única proteção → adicionar autorização real.
- Botão escondido como "controle de acesso" → adicionar autorização real.

Segurança cosmética cria falsa confiança, que é pior que insegurança honesta.

---

# PARTE 2 — MODELO DE CONFIANÇA

Antes de qualquer decisão de segurança, defina explicitamente: **o que é confiável e o que não é, neste sistema, agora.**

## Zonas de confiança

| Zona | Exemplos | Nível de confiança |
|------|----------|-------------------|
| Núcleo confiável | Código do servidor rodando em ambiente controlado, banco de dados primário, gerenciador de segredos | Alto — mas ainda assim, defesa em profundidade |
| Periferia confiável | Workers internos, filas, cache server-side, serviços internos com mTLS | Médio — autenticação mútua, comunicação cifrada |
| Provedores externos verificados | Stripe (com webhook assinado), Auth0, AWS S3 (com IAM) | Médio — só após verificação criptográfica de cada interação |
| Clientes próprios | Navegador rodando o frontend oficial, app mobile oficial | **Nenhum.** Não confie. |
| Clientes terceiros | Integrações, APIs públicas, scrapers | Nenhum |
| Inputs externos | Webhooks não verificados, emails recebidos, uploads | Nenhum |

## Princípio operacional

> **Confie no que você verificou. Verifique cada vez. Verificações expiram.**

Token válido há 5 minutos não é prova de que está válido agora. Permissão de ontem não é permissão de hoje. Assinatura aceita uma vez não autoriza segunda chamada.

---

# PARTE 3 — PRINCÍPIOS DE ARQUITETURA SEGURA

## 3.1 Servidor é a fonte de verdade

```
Frontend       = camada visual
Backend        = fonte de verdade
Banco          = persistência confiável
Provedores ext = validação assíncrona com verificação criptográfica
```

## 3.2 Princípio do menor privilégio

- Usuário tem apenas as permissões necessárias para sua função.
- Serviço tem apenas o acesso necessário aos recursos que usa.
- Conexão de banco da aplicação não usa role de superuser.
- Chave de API tem escopo restrito.
- Token de sessão expira.

## 3.3 Falha segura

Quando algo dá errado, o sistema falha **negando**, não permitindo.

- Erro ao verificar permissão → nega acesso, não permite.
- Erro ao validar token → desloga, não mantém sessão.
- Erro ao consultar plano → assume free, não assume premium.
- Erro ao verificar webhook → rejeita, não processa.

## 3.4 Separação de privilégio

Operações críticas exigem mais de um fator ou mais de uma autorização.

- Mudança de email → exige senha + confirmação por email.
- Exclusão de conta → exige confirmação explícita + atraso.
- Acesso admin → exige MFA.
- Operação financeira de alto valor → exige reautenticação.

## 3.5 Mínima superfície de exposição

- Endpoint que não precisa existir → não existe.
- Campo de banco que não precisa ser retornado → não é retornado.
- Permissão que não precisa ser concedida → não é concedida.
- Porta que não precisa estar aberta → não está aberta.

## 3.6 Auditabilidade

Toda ação que afeta segurança deixa rastro:
- Quem fez.
- O que fez.
- Quando fez.
- De onde fez.
- Resultado.

Sem isso, incidente é especulação.

---

# PARTE 4 — THREAT MODEL

Antes de auditar, modele as ameaças do sistema específico. Use STRIDE como guia, mas **instancie cada categoria com cenário concreto deste projeto**.

## STRIDE aplicado

| Categoria | Pergunta |
|-----------|----------|
| **S**poofing | Como alguém pode se passar por outro usuário, serviço ou origem? |
| **T**ampering | Como alguém pode alterar dados em trânsito, em repouso ou no cliente? |
| **R**epudiation | Como uma ação crítica pode ser executada sem registro confiável? |
| **I**nformation Disclosure | Como dados sensíveis podem vazar (resposta de API, log, erro, cache, URL)? |
| **D**enial of Service | Como o sistema pode ser derrubado ou tornado lento por um atacante? |
| **E**levation of Privilege | Como um usuário comum pode virar admin? Como um usuário free vira premium sem pagar? |

## Hipótese de adversário (defina antes de auditar)

| Atacante | Capacidades assumidas |
|----------|----------------------|
| Usuário autenticado curioso | DevTools, modificação de requests, tentativa de IDOR |
| Ex-funcionário | Conhecimento da stack, possível acesso prévio a tokens |
| Concorrente | Recursos para pentesting prolongado, conhecimento do mercado |
| Bot oportunista | Scanners automatizados, listas de exploits conhecidos |
| Atacante direcionado | Recursos significativos, paciência, possível engenharia social |
| Insider malicioso | Acesso legítimo, conhecimento interno |

**Defina a barra:** contra quais desses o sistema precisa resistir? Um SaaS financeiro precisa resistir a todos. Um app interno de notas pode aceitar resistir só ao bot oportunista.

## Ativos críticos (identifique no seu sistema)

Para cada um, mapeie ameaças concretas:
- Credenciais de usuários.
- Dados financeiros / PII.
- Tokens de sessão.
- Segredos de integração (chaves de API, webhook secrets).
- Capacidade de cobrar / movimentar dinheiro.
- Capacidade de enviar comunicações em massa.
- Capacidade de executar código no servidor.

---

# PARTE 5 — DOMÍNIOS TÉCNICOS

## 5.1 Autenticação

### Requisitos

- **Identidade verificada server-side em toda requisição privada.**
- Senhas armazenadas com algoritmo adequado a senhas: **argon2id** (preferencial), **scrypt** ou **bcrypt** com custo apropriado ao hardware atual. **Nunca** SHA-1/SHA-256/MD5 puros, **nunca** PBKDF2 com poucas iterações.
- Sessões/tokens emitidos pelo servidor, com expiração, invalidáveis, transportados sobre canal seguro.
- Reset de senha: token de uso único, expiração curta (15–60 minutos), invalidação de todas as sessões anteriores ao trocar.
- MFA disponível para todos os usuários, **obrigatório** para admins.
- Proteção contra credential stuffing: rate limit por conta e por IP, lockout temporário, alerta ao usuário.
- Sem enumeração de usuários: respostas idênticas para "usuário não existe" e "senha errada"; signup não revela se email já está cadastrado (envia email confirmando ou informando que conta já existe).

### Tokens (JWT, opaque tokens, cookies de sessão)

- Se JWT: assinado, com expiração curta, refresh separado, **mecanismo de revogação** (lista de revogados, kid rotativo, ou sessões server-side).
- Se opaque token: validação server-side a cada uso, expiração, revogação por logout.
- Se cookie de sessão: `HttpOnly`, `Secure`, `SameSite` apropriado (`Lax` ou `Strict`), nome não óbvio, domínio restrito.
- **Nunca** colocar dados sensíveis no payload de JWT. JWT não é cofre, é envelope assinado público.

### Fluxos sensíveis

- Login: rate limit, log de tentativas, alerta em padrão suspeito.
- Logout: invalida sessão server-side, não só limpa cookie.
- Troca de email: confirmação no email novo + email antigo + senha.
- Troca de senha: senha atual + invalidação de outras sessões.
- Recuperação de senha: token único, expiração, invalidação após uso.

### Anti-padrões

```
❌ Senha em SHA-256 sem salt
❌ JWT sem expiração
❌ JWT com `role` no payload usado sem revalidação
❌ "Lembrar de mim" implementado com cookie permanente sem rotação
❌ Mensagem de erro "senha incorreta para usuário X" (confirma existência)
❌ Token em URL (`?token=abc`) — vai pra log, histórico, referrer
❌ Refresh token armazenado em localStorage acessível a XSS
```

---

## 5.2 Autorização

### Requisitos

- Toda rota privada nega por padrão.
- Decisão de autorização baseada em **estado verificado server-side**, nunca em claim recebido do cliente.
- Modelo de permissão explícito e centralizado: RBAC (role-based), ABAC (attribute-based) ou ReBAC (relationship-based). Não permissões espalhadas em ifs pelo código.
- Verificação aplicada **próxima ao acesso ao dado**, não só no controller. Defesa em profundidade.
- Negação retorna `404` quando revelar existência do recurso é em si um vazamento; `403` quando a existência já é conhecida pelo solicitante.

### Padrão de implementação

```pseudo
// Camada de rota
@require_auth
@require_role("admin")
def admin_endpoint(current_user):
    return service.list_all_users(requester=current_user)

// Camada de serviço (defesa em profundidade)
def list_all_users(requester):
    if requester.role != "admin":
        raise AuthorizationError()
    return db.fetch_users()

// Camada de banco (defesa em profundidade adicional)
// Row-level security policy: admin pode ler todos; user só lê a si mesmo
```

### Anti-padrões

```
❌ if (req.body.role === "admin") { ... }
❌ Verificação apenas em middleware de frontend
❌ Função pública que confia que o chamador já validou
❌ Endpoint admin sem prefixo distinto e sem proteção no roteador
❌ Permissão concedida e nunca revogada
```

---

## 5.3 Isolamento entre usuários e tenants

### Requisitos

- Toda entidade que pertence a alguém tem campo de propriedade (`user_id`, `tenant_id`, ou ambos).
- **Toda** query que lê ou modifica essa entidade filtra por essa propriedade.
- Considere **Row-Level Security (RLS)** no banco como defesa em profundidade quando suportado (PostgreSQL, Oracle).
- IDs expostos publicamente são **opacos e não-adivinháveis** (UUID v4, ULID, nanoid) — IDs sequenciais facilitam enumeração.
- Operações em lote validam ownership de cada item, não só do primeiro.

### Padrão de implementação

```pseudo
def get_resource(resource_id, current_user):
    resource = db.query(Resource).filter(
        Resource.id == resource_id,
        Resource.user_id == current_user.id  // OBRIGATÓRIO
    ).first()
    
    if not resource:
        raise NotFound()  // 404, não 403 — não revela existência
    
    return resource
```

### Multi-tenancy

Se há tenants (empresas, organizações):
- Toda entidade tem `tenant_id`.
- Usuário pertence a um ou mais tenants com role por tenant.
- Toda query filtra por tenant **atual da sessão**, não por tenant passado no request.
- Mudança de tenant ativo é operação explícita, registrada, com revalidação de permissão.

### Anti-padrões

```
❌ SELECT * FROM transactions WHERE id = ?
❌ GET /api/users/123/profile  (sem verificar que 123 == current_user)
❌ Operação em lote que valida só o primeiro item
❌ ID sequencial em URL pública
❌ Tenant_id vindo do client e usado sem verificação
```

---

## 5.4 Validação de entrada

### Requisitos

- **Schema explícito por endpoint** (allowlist de campos, tipos, formatos, faixas).
- **Rejeitar** campos não esperados, não apenas ignorar — previne mass assignment.
- Limites de tamanho: strings, arrays, objetos aninhados, profundidade de JSON, tamanho total do payload.
- Validação de tipo, faixa numérica, formato (email, UUID, data).
- Validação de **relacionamento**: o `category_id` enviado pertence ao usuário atual? O `account_id` referenciado existe e é dele?
- Validação acontece **antes** de qualquer lógica de negócio, não no meio.

### Mass assignment

```
❌ User.update(request.body)  // request.body pode conter {"role": "admin"}
✅ allowed_fields = ["name", "email"]
   User.update({k: v for k, v in request.body.items() if k in allowed_fields})
```

Melhor ainda: use schemas que **rejeitam** campos extras com erro explícito.

### Tipos comuns que precisam de validação cuidadosa

- Números monetários: usar tipo decimal exato, não float; validar faixa; verificar moeda.
- Datas: validar formato, faixa, timezone.
- Emails: validar com biblioteca, normalizar (lowercase), considerar comprimento.
- URLs: validar esquema (apenas http/https, nunca file/javascript/data), validar host.
- HTML/texto rico: sanitizar com biblioteca específica (não regex), allowlist de tags.
- Identificadores: validar formato (UUID, ULID); rejeitar strings arbitrárias.

### Anti-padrões

```
❌ Regex próprio para email
❌ Aceitar JSON arbitrariamente aninhado
❌ Confiar em validação do cliente
❌ Validar formato mas não relacionamento ("category existe?" sem checar "é do usuário?")
❌ Sanitizar HTML com regex
❌ Aceitar amount como float
```

---

## 5.5 Sanitização de saída

### Requisitos

- **Encoding contextual:** cada contexto de saída exige encoding próprio.
  - HTML body → HTML encoding.
  - Atributo HTML → atributo encoding.
  - JavaScript → JS encoding (idealmente, não interpolar dados em JS; usar `data-*` e ler do DOM).
  - URL → URL encoding.
  - CSS → CSS encoding (raro, mas existe).
- **Responses JSON** não contêm:
  - Hashes de senha.
  - Tokens de qualquer tipo.
  - Segredos de provedores.
  - Campos internos (`internal_notes`, `admin_flags`).
  - Dados de outros usuários (mesmo agregados, sem permissão).
- **Defina explicitamente o DTO** retornado para o cliente. Não serialize o modelo do banco direto.

### Padrão de implementação

```pseudo
// ERRADO: serializa tudo do banco
return jsonify(user)

// CERTO: DTO explícito
return jsonify({
    "id": user.id,
    "name": user.name,
    "email": user.email,
    "plan": user.plan,        // só pra UI; servidor revalida em rotas privadas
    "role": user.role         // idem
})
```

### Anti-padrões

```
❌ return jsonify(user.__dict__)
❌ Retornar password_hash mesmo "para debug"
❌ Mensagem de erro com SQL na resposta
❌ Stack trace em produção
❌ Header com versão do framework
```

---

## 5.6 Sessão, cookies e CSRF

### Cookies de sessão

- `HttpOnly` — impede acesso via JS, mitiga XSS.
- `Secure` — só transmite em HTTPS.
- `SameSite=Lax` (padrão razoável) ou `SameSite=Strict` (mais seguro, pode quebrar fluxos OAuth).
- `Domain` restrito ao necessário.
- `Path` restrito ao necessário.
- Nome do cookie não revelador (evite `PHPSESSID`, `JSESSIONID`).
- Expiração explícita.

### CSRF

Aplicável a fluxos baseados em cookie. APIs com Bearer token em header não são vulneráveis a CSRF clássico (mas estão vulneráveis a XSS roubando o token — escolha conscientemente).

Mitigações (combinadas):
- `SameSite=Lax` ou `Strict` no cookie de sessão.
- Token CSRF sincronizador para ações state-changing.
- Verificar header `Origin` ou `Referer` em ações sensíveis.
- Nunca aceitar state-changing operations via `GET`.

### Logout

- Invalida sessão server-side, não só limpa cookie.
- Considera "logout de todos os dispositivos" como opção.
- Após troca de senha, invalida todas as outras sessões.

### Anti-padrões

```
❌ Sessão sem expiração
❌ Cookie sem HttpOnly em sessão
❌ CSRF token igual para todos os usuários
❌ Aceitar DELETE via GET
❌ Logout que só limpa cookie do cliente
```

---

## 5.7 Gestão de segredos

### Requisitos

- **Nenhum segredo no código-fonte.** Nenhum. Inclui chaves de teste, chaves "temporárias", chaves "que só funcionam em dev".
- **Nenhum segredo em logs.**
- **Nenhum segredo em URLs.** URLs vão pra logs de servidor, logs de proxy, histórico de browser, referrer headers.
- Segredos injetados em runtime via:
  - Gerenciador de segredos (AWS Secrets Manager, GCP Secret Manager, Vault, Doppler).
  - Variáveis de ambiente vindas do orquestrador (Kubernetes secrets, Docker secrets).
  - Nunca arquivo `.env` em produção sem proteção de filesystem.
- `.env` no `.gitignore`. Sempre. Verifique o histórico do git — se já foi commitado, **considere comprometido e rotacione**.
- Segredos diferentes por ambiente (dev, staging, prod).
- Rotação documentada e factível. Idealmente automatizada.
- Princípio do menor privilégio: cada serviço recebe apenas os segredos que usa.

### Detecção

- Pre-commit hook que escaneia por padrões de segredo (`git-secrets`, `gitleaks`, `trufflehog`).
- CI que falha se detectar segredo em diff.
- Scan periódico do histórico do repositório.

### Em caso de exposição

Trate como **incidente de segurança**:
1. Rotacione imediatamente.
2. Revogue o segredo antigo.
3. Audite logs do provedor para uso suspeito.
4. Não confie em "remover do git resolve" — o segredo já foi exposto.

### Anti-padrões

```
❌ STRIPE_KEY = "sk_live_..." no código
❌ console.log(API_KEY)
❌ URL como https://api.example.com/?api_key=...
❌ Segredo no Dockerfile
❌ Segredo em variável de build do frontend (vai pro bundle)
❌ Mesmo segredo em dev e prod
```

---

## 5.8 Criptografia

### Princípio fundamental

**Não implemente criptografia. Use bibliotecas auditadas.**

Use APIs de alto nível (libsodium, AWS KMS, equivalentes). Quando isso não é possível, use as primitivas certas com cuidado extremo.

### Em trânsito

- TLS 1.2 mínimo, preferencialmente 1.3.
- HSTS em produção (com `max-age` longo, depois de testar bem).
- Certificados válidos, renovação automatizada (Let's Encrypt, cert-manager).
- Sem mixed content.
- Cabeçalho `Strict-Transport-Security` configurado.

### Em repouso

Criptografar:
- Senhas (com hash, não com cripto reversível).
- Dados sensíveis regulados (PII conforme legislação aplicável).
- Tokens armazenados de provedores externos.
- Backups que saem do ambiente controlado.

Não precisa criptografar:
- Dados públicos.
- Dados que o usuário publica intencionalmente.
- Identificadores internos sem valor isolado.

### Primitivas

| Uso | Recomendação | Evitar |
|-----|-------------|--------|
| Hash de senha | argon2id, scrypt, bcrypt | SHA-*, MD5, PBKDF2 com baixa iteração |
| Hash genérico (não-senha) | SHA-256, BLAKE2, SHA-3 | MD5, SHA-1 |
| Criptografia simétrica | AES-GCM, ChaCha20-Poly1305 | AES-ECB, AES-CBC sem MAC, DES |
| Criptografia assimétrica | Ed25519, X25519, RSA-OAEP (2048+) | RSA-PKCS1v15, RSA < 2048 |
| Assinatura | Ed25519, ECDSA P-256 | RSA-PKCS1v15 sem cuidado |
| Random para uso cripto | CSPRNG da plataforma (`os.urandom`, `crypto.randomBytes`, `SecureRandom`) | `Math.random`, `rand()`, `Random()` |
| Comparação de strings sensíveis | Comparação tempo-constante | `==` (vulnerável a timing attacks) |

### Anti-padrões

```
❌ MD5 ou SHA-1 para senha
❌ AES-ECB
❌ Reusar nonce/IV em AES-GCM
❌ Math.random() para token
❌ if (token == expected) — vazamento por timing
❌ Implementar HMAC manualmente
❌ "Encriptar" com base64
❌ Misturar chaves de assinatura com chaves de criptografia
```

---

## 5.9 Upload de arquivos

### Requisitos

- **Tamanho limitado no servidor** (não confie no client; configure também no proxy reverso).
- **Tipo validado por conteúdo** (magic bytes / detecção real), não só extensão ou MIME declarado.
- **Nome gerado pelo servidor** (UUID, hash). Nome original preservado como metadado se necessário, mas nunca usado como path.
- **Path traversal impossível**: validação rigorosa de qualquer parte do nome que toque o filesystem.
- **Armazenamento fora do diretório servido publicamente.**
- Servido via:
  - Endpoint autorizado que faz proxy do arquivo após verificar permissão, **ou**
  - URL pré-assinada com expiração curta (S3-style), gerada após verificação de permissão.
- **Cabeçalhos de resposta seguros** ao servir:
  - `Content-Disposition: attachment` para downloads.
  - `X-Content-Type-Options: nosniff`.
  - `Content-Type` correto e específico.
  - **Nunca** servir uploads de usuários do mesmo domínio do app principal (use subdomínio dedicado ou CDN) — mitiga XSS via SVG/HTML upload.
- **Antivírus / scanning** quando o caso exige (compartilhamento entre usuários, anexos de email).
- Rate limit no endpoint de upload.
- Logs de tentativas recusadas.

### Validação por conteúdo

```pseudo
allowed_types = {"application/pdf", "text/csv", "application/x-ofx"}
detected_type = detect_mime_by_magic_bytes(file_bytes)  // não confie no header MIME
if detected_type not in allowed_types:
    reject()
```

### Casos especiais

- **Imagens:** reprocesse a imagem (decode + re-encode) para remover metadados e payloads embutidos. Limite dimensões.
- **SVG:** sanitize (remove `<script>`, `onclick`, etc.) ou rejeite.
- **PDF:** se o usuário não precisa ler o PDF original, considere converter para imagem ou texto.
- **Arquivos compactados (ZIP, tar):** atenção a zip bombs (limite de descompressão), zip slip (path traversal).
- **Polyglots:** arquivos que são válidos em múltiplos formatos. Mitigação: validação estrita + reprocessamento.

### Anti-padrões

```
❌ Confiar no Content-Type do cliente
❌ Confiar na extensão
❌ Salvar com nome do cliente: open(user_filename, "w")
❌ Servir uploads do mesmo domínio do app
❌ Sem limite de tamanho
❌ Sem limite de quantidade por usuário
❌ Descompactar sem limite de tamanho final
❌ Permitir upload para path arbitrário
```

---

## 5.10 Pagamentos e cobrança

### Princípios

- **Cliente envia apenas identificador simbólico.** Nunca preço, nunca valor, nunca moeda, nunca desconto.
- **Servidor resolve preço** a partir de configuração server-side ou catálogo do provedor.
- **Ativação só após confirmação assíncrona verificada** (webhook assinado), **nunca** após redirect de sucesso.
- **Idempotência** em todas as operações de cobrança.
- **Logs financeiros imutáveis** e auditáveis.

### Fluxo correto (com provedor tipo Stripe)

```
1. Cliente envia: POST /checkout { "planKey": "premium_monthly" }
2. Servidor mapeia internamente:
   PLAN_MAP = {
     "premium_monthly": { "provider_price_id": env.STRIPE_PRICE_PREMIUM_MONTHLY }
   }
3. Servidor cria sessão de checkout com price_id resolvido server-side.
4. Cliente é redirecionado para o provedor.
5. Provedor processa pagamento.
6. Provedor chama webhook do servidor.
7. Servidor:
   - Verifica assinatura do webhook (HMAC com secret do provedor).
   - Verifica idempotência (event_id já processado? ignora).
   - Atualiza assinatura no banco.
8. Usuário ganha acesso.
```

### O que NUNCA aceitar do cliente

```
❌ amount
❌ price
❌ currency
❌ discount
❌ coupon (sem revalidação server-side)
❌ trial_days
❌ premium: true
❌ subscription_status
❌ stripe_customer_id
```

### Estado de assinatura

- Derivado do provedor, persistido no banco do servidor.
- Status canônicos: `active`, `trialing`, `past_due`, `canceled`, `incomplete`, `expired`.
- Premium liberado apenas em `active` ou `trialing`.
- Reconciliação periódica com o provedor (não confie só em webhooks, eles podem falhar).

### Webhooks

- Assinatura verificada com secret do provedor.
- Idempotência por `event_id`.
- Resposta rápida (`200 OK`) seguida de processamento assíncrono se for caro.
- Replay protection: timestamps válidos, eventos antigos rejeitados.

### Anti-padrões

```
❌ Ativar premium ao chegar em /success
❌ Confiar em "premium": true do client
❌ Cobrar valor enviado no body
❌ Webhook sem verificação de assinatura
❌ Processar mesmo event_id duas vezes
❌ Stripe customer ID confiando no client
```

---

## 5.11 Limites de uso e planos

### Requisitos

- **Limites calculados no servidor**, baseados em estado persistido.
- **Eventos de uso registrados no banco**, não em localStorage do cliente.
- Verificação **antes** da operação cara, não depois.
- Resposta clara quando limite é atingido.

### Padrão de implementação

```pseudo
// Tabela usage_events
{
  id, user_id, event_type, created_at, metadata
}

// Verificação
def check_usage_limit(user, event_type):
    plan = get_user_plan_from_db(user.id)  // server-side
    limit = PLAN_LIMITS[plan][event_type]
    
    if limit is None:  // unlimited
        return
    
    used = count_events_in_period(user.id, event_type, period=plan.period)
    if used >= limit:
        raise UsageLimitReached(event_type=event_type)

// Antes da operação
check_usage_limit(current_user, "upload_statement")
process_upload(...)
record_usage_event(current_user, "upload_statement")
```

### Anti-padrões

```
❌ Contador em localStorage
❌ Contador em variável JS do frontend
❌ Verificação só na UI (botão desabilitado)
❌ Verificação após processar a operação cara
❌ Sem registro persistido — recálculo impossível
```

---

## 5.12 Rate limiting e proteção contra abuso

### Onde aplicar

- **Login:** por conta E por IP. Lockout temporário após N falhas.
- **Reset de senha:** por conta E por IP.
- **Signup:** por IP, com captcha após threshold.
- **Endpoints caros:** geração de relatório, exportação, parsing.
- **Upload:** tamanho total por janela.
- **APIs de busca:** previne scraping.
- **Webhooks enviados:** previne abuso amplificado.

### Estratégias

| Estratégia | Quando usar |
|-----------|-------------|
| Token bucket | Picos curtos aceitáveis |
| Sliding window | Limite suave e justo |
| Fixed window | Simples, mas tem bordas |
| Concurrent connections | Operações de longa duração |

### Considerações

- **Identificador correto:** IP isolado é frágil (atrás de NAT, IPv6 facilmente rotacionável). Combine com conta, dispositivo, sessão.
- **Resposta:** `429 Too Many Requests` com header `Retry-After`. Mensagem não vaza limites exatos (evita timing).
- **Não vire DoS contra você mesmo:** lockout deve ter caminho de recuperação (email, suporte), não bloqueio permanente.
- **Considere CAPTCHA** após threshold em endpoints públicos sensíveis.

### Anti-padrões

```
❌ Rate limit apenas no frontend
❌ Limite tão alto que não protege
❌ Limite tão baixo que vira DoS
❌ Apenas por IP em era de IPv6
❌ Sem mecanismo de recuperação para usuário legítimo bloqueado
```

---

## 5.13 Logging, auditoria e observabilidade

### O que logar

- Login (sucesso e falha).
- Logout.
- Troca de senha, email, MFA.
- Mudança de permissão / role.
- Acesso a dados sensíveis (especialmente admin acessando dado de usuário).
- Ações administrativas.
- Falhas de autorização (`403`).
- Falhas de autenticação (`401`).
- Operações financeiras.
- Uploads aceitos e recusados.
- Rate limit atingido.
- Erros 5xx do servidor.

### O que NUNCA logar

- Senhas (mesmo erradas).
- Tokens (sessão, API, refresh, CSRF).
- Números de cartão.
- Dados de CVV.
- Conteúdo de transação financeira sem mascarar.
- PII desnecessária para o propósito do log.
- Request bodies completos de endpoints sensíveis.
- Stack traces em logs de aplicação visíveis a usuários.

### Formato

- **Estruturado** (JSON), não texto livre.
- **Timestamp em UTC**, formato ISO 8601.
- **Correlation ID** (request ID) que conecta log → request → resposta → eventos derivados.
- **Identificadores opacos** quando possível.
- **Severidade explícita** (debug, info, warn, error, critical).

### Retenção e proteção

- Logs de segurança retidos por período compatível com a legislação e necessidades de investigação (90 dias mínimo é prática comum; alguns regimes exigem mais).
- Logs são alvo: protegidos com mesmo cuidado que dados.
- Acesso a logs é auditado.
- Logs imutáveis (write-once) para eventos críticos.

### Detecção e alerta

- Padrões a alertar:
  - Múltiplas falhas de login.
  - Falha de autorização em rajada.
  - Acesso admin fora de horário típico.
  - Pico de erro 5xx.
  - Webhook falhando consistentemente.
  - Upload recusado em rajada.
- Resposta a alerta documentada.

### Anti-padrões

```
❌ console.log(password)
❌ Log em texto plano sem estrutura
❌ Timezone local (debugar incidente vira pesadelo)
❌ Sem correlation ID
❌ Log de body completo de /login
❌ Logs acessíveis sem autorização
❌ Logs nunca expirados (custo + LGPD/GDPR)
```

---

## 5.14 Tratamento de erros

### Em produção

- Resposta genérica ao cliente: `{"error": "internal_server_error", "message": "Tente novamente"}`.
- Detalhe técnico apenas no log server-side, correlacionado por request ID.
- Nunca:
  - Stack trace na resposta.
  - Nome de tabela.
  - Query SQL.
  - Path interno.
  - Versão de framework.
  - Mensagem que distinga "não existe" de "sem permissão" (use 404 para ambos quando apropriado).

### Códigos HTTP

| Código | Quando |
|--------|--------|
| 400 | Input inválido (formato, schema) |
| 401 | Não autenticado |
| 403 | Autenticado mas sem permissão (existência conhecida) |
| 404 | Recurso não existe **OU** existe mas usuário não pode saber |
| 409 | Conflito de estado |
| 422 | Input válido em formato mas inválido em conteúdo |
| 429 | Rate limit |
| 500 | Erro genérico do servidor |
| 503 | Indisponibilidade temporária |

### Modo desenvolvimento

- Stack trace OK em dev.
- Verifique que `DEBUG=False` (ou equivalente) em produção, validado na startup.
- Falhe a inicialização se variável de produção estiver permissiva.

### Anti-padrões

```
❌ throw new Error("User johndoe@example.com not found in table users")
   ↑ confirma existência, vaza schema, vaza email de usuário
❌ Stack trace em response JSON
❌ Mensagem em inglês técnico para usuário final
❌ Diferenciar "usuário não existe" de "senha errada" no login
```

---

## 5.15 Configuração de servidor e CORS

### Cabeçalhos de segurança

| Header | Recomendação |
|--------|-------------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` (depois de testar) |
| `Content-Security-Policy` | Política específica do app; comece restrita |
| `X-Content-Type-Options` | `nosniff` |
| `X-Frame-Options` | `DENY` ou `SAMEORIGIN` (ou use `frame-ancestors` no CSP) |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Restringir features não usadas (geolocation, camera, etc.) |
| `Cache-Control` | `no-store` em respostas autenticadas |

### CORS

- **Origens explícitas em produção.** Sem `*`.
- `Access-Control-Allow-Credentials: true` exige origem específica.
- Pre-flight (`OPTIONS`) configurado corretamente.
- Considere se o endpoint precisa mesmo ser acessível cross-origin.

```
❌ Access-Control-Allow-Origin: *
✅ Access-Control-Allow-Origin: https://app.exemplo.com
```

### Configuração de runtime

- Debug desabilitado em produção, **validado na startup** (falhar se ligado).
- Versões de runtime e framework suportadas (sem EOL).
- Variáveis de ambiente obrigatórias validadas na startup.
- Princípio do menor privilégio para o processo do servidor (usuário não-root).
- Filesystem read-only quando possível.
- Limites de recurso (memória, CPU, file descriptors).

### Anti-padrões

```
❌ DEBUG=True em produção
❌ allow_origins=["*"] com credentials
❌ Servidor rodando como root
❌ Variável crítica com default permissivo
❌ Versão de framework com EOL conhecido
```

---

## 5.16 Dependências e supply chain

### Requisitos

- **Inventário (SBOM)** das dependências diretas e transitivas.
- **Scanner automatizado de CVE** (Dependabot, Snyk, Renovate, OSV, Trivy).
- **Política de atualização** definida (críticas em horas/dias, outras em semanas).
- **Versões pinadas** (lockfile commitado).
- **Auditoria de dependências novas** antes de adicionar:
  - É mantida ativamente?
  - Quantos mantenedores?
  - Quantos downloads / uso?
  - Histórico de vulnerabilidades?
  - Faz mesmo o que diz fazer?
- **Verificação de integridade** (hashes, assinaturas) quando o ecossistema suporta.

### Riscos específicos

- **Typosquatting:** pacote com nome similar a popular (`reqests` em vez de `requests`). Use scanners.
- **Dependency confusion:** pacote interno publicado externamente por atacante. Use registry interno e fixe prefixos.
- **Mantenedor comprometido:** atualização legítima vira maliciosa. Lockfile + revisão de mudanças.
- **Pacote abandonado:** sem patches de segurança. Substituir.

### Build e deploy

- Builds reprodutíveis quando possível.
- CI roda em ambiente isolado.
- Imagens base de Docker oficiais e atualizadas.
- Mínima imagem final (distroless, alpine).
- Scan de imagem antes de push para registry.

### Anti-padrões

```
❌ npm install pacote-aleatório-novo sem checar
❌ Sem lockfile
❌ Atualização automática sem teste
❌ Versão "*" ou "latest" em produção
❌ Imagem base sem atualização há anos
❌ CVE crítica conhecida ignorada por meses
```

---

## 5.17 Negação de serviço

### Vetores comuns

- Payload gigante.
- JSON profundamente aninhado.
- Regex catastrófico (ReDoS).
- Operações N+1 em endpoint público.
- Operações sem timeout em integração externa.
- Decompressão sem limite.
- Geração de relatório enorme em endpoint síncrono.
- Conexões abertas sem limite.

### Mitigações

- **Limites de payload** no proxy reverso e no app.
- **Limites de profundidade e tamanho** de JSON.
- **Timeouts** em toda I/O externa (HTTP, banco, cache, fila).
- **Paginação obrigatória** em listagens. Sem `limit` controlado só pelo cliente; sempre teto server-side.
- **Operações caras isoladas** em workers / filas / jobs assíncronos.
- **Circuit breaker** em integrações externas.
- **Rate limit** em endpoints caros.
- **Validação de regex** ou uso de bibliotecas com proteção contra ReDoS.
- **GraphQL:** limite de profundidade, complexidade, cost analysis.

### Anti-padrões

```
❌ res.json(allRecords)  sem paginação
❌ for x in inputArray: doExpensiveThing()  sem limite no tamanho do array
❌ fetch(externalUrl)  sem timeout
❌ regex /(a+)+$/ aplicado a input do usuário
❌ ZipFile.extractall() sem limite
❌ Sem queue para operações caras
```

---

## 5.18 SSRF, XXE e ataques server-side

### SSRF (Server-Side Request Forgery)

Se o servidor faz requests baseados em input do usuário (importar URL, webhook, OAuth callback, fetch de imagem), atacante pode forçar requests para:
- Metadata endpoint da cloud (`169.254.169.254` em AWS/GCP — gera credenciais).
- Serviços internos sem auth (`localhost:6379` Redis, `localhost:8500` Consul).
- Rede privada (`10.0.0.0/8`, `192.168.0.0/16`).

### Mitigações

- **Allowlist de hosts** quando possível.
- **Bloquear IPs privados, loopback, link-local, multicast** após resolução DNS.
- **Re-resolver DNS** no momento da conexão (atacante pode usar TTL curto para mudar resolução entre validação e uso — DNS rebinding).
- **Desabilitar redirects** ou re-validar destino em cada redirect.
- **Timeouts curtos.**
- **Sem credenciais ambiente** disponíveis ao processo que faz fetch arbitrário.
- **Network policy** que isola o serviço de fazer requests à rede interna não autorizada.

### XXE (XML External Entities)

Se o servidor parseia XML, configure o parser para:
- Desabilitar resolução de entidades externas.
- Desabilitar DTDs.
- Desabilitar acesso a recursos externos.

A maioria dos parsers modernos já tem defaults seguros, mas verifique explicitamente.

### Injeções variadas

| Tipo | Onde acontece | Mitigação |
|------|---------------|-----------|
| SQL injection | Queries construídas por concatenação | Queries parametrizadas, ORM com cuidado |
| NoSQL injection | Queries em MongoDB, etc. | Validação de tipo, queries com placeholder |
| Command injection | `exec`, `system`, shell | Evitar shell; usar APIs com argumentos separados |
| LDAP injection | Queries LDAP | Escape correto, queries parametrizadas |
| Template injection | Server-side templates com input do usuário | Não passe input direto pro template engine |
| Header injection | `\r\n` em headers | Bibliotecas que validam |

### Anti-padrões

```
❌ db.query("SELECT * FROM x WHERE id = " + user_input)
❌ exec(f"convert {user_filename} output.png")
❌ template.render(user_template_string)
❌ urllib.request(user_url)  sem validação
❌ Parser XML com defaults antigos
```

---

## 5.19 Race conditions e idempotência

### Onde aparecem

- Pagamentos: clicar duas vezes → cobrar duas vezes.
- Limites de uso: dois requests simultâneos passam por verificação antes de qualquer registro.
- Saldo: transferência verificada e debitada em momentos diferentes.
- Criação de recurso único (nome de usuário): dois requests simultâneos criam duplicata.

### Mitigações

- **Transações de banco** com isolamento adequado.
- **Locks** (otimistas via version/etag, ou pessimistas via `SELECT FOR UPDATE`).
- **Constraints únicos** no banco (`UNIQUE` index).
- **Idempotency keys** em operações state-changing externas (cliente envia chave, servidor garante que mesma chave = mesmo resultado).
- **Operações atômicas** quando o banco suporta (`INSERT ... ON CONFLICT`, `UPDATE ... WHERE`).

### Padrão para pagamento

```pseudo
def create_charge(user, plan_key, idempotency_key):
    existing = db.find_charge(idempotency_key=idempotency_key)
    if existing:
        return existing  // já processado
    
    with db.transaction():
        charge = db.create_charge(
            user_id=user.id,
            idempotency_key=idempotency_key,
            // ...
        )
        provider.create_charge(amount=resolve_amount(plan_key), idempotency_key=idempotency_key)
        return charge
```

### Anti-padrões

```
❌ Verificar saldo, depois debitar (sem lock)
❌ Verificar limite de uso, depois registrar uso (sem atomicidade)
❌ Botão "pagar" sem idempotency key
❌ Verificar unicidade no app, criar sem constraint no banco
```

---

## 5.20 Webhooks recebidos

### Requisitos

- **Assinatura criptográfica verificada** (HMAC com secret do provedor) em **toda** chamada.
- **Comparação tempo-constante** na verificação.
- **Idempotência** por `event_id`.
- **Replay protection:** timestamps válidos dentro de janela curta; eventos antigos rejeitados.
- **Resposta rápida** (`200 OK`) seguida de processamento assíncrono se for caro.
- **Logs detalhados** mas sem expor secret.
- **Sem efeito colateral** antes da verificação de assinatura.

### Padrão

```pseudo
@route("/webhooks/provider")
def handle_webhook(request):
    payload = request.raw_body  // bytes exatos, antes de parse
    signature = request.headers["X-Signature"]
    timestamp = request.headers["X-Timestamp"]
    
    if not within_acceptable_window(timestamp):
        return 400  // replay attack
    
    expected = hmac_sha256(WEBHOOK_SECRET, timestamp + payload)
    if not constant_time_compare(signature, expected):
        return 401
    
    event = parse_json(payload)
    if already_processed(event.id):
        return 200  // idempotente
    
    enqueue_processing(event)
    return 200
```

### Anti-padrões

```
❌ Webhook sem verificação de assinatura
❌ signature == expected (timing attack)
❌ Processar antes de verificar
❌ Sem proteção contra replay
❌ Mesmo event_id processado N vezes
❌ Processamento síncrono pesado bloqueia o provedor
```

---

## 5.21 Conformidade legal e proteção de dados

### Identificação da legislação aplicável

Depende de **onde os dados são coletados**, **de quem**, e **onde são processados**:

| Lei | Quando se aplica |
|-----|-----------------|
| **LGPD** (Brasil) | Tratamento de dados de pessoas no Brasil, ou por empresa no Brasil |
| **GDPR** (UE) | Tratamento de dados de pessoas na UE |
| **CCPA/CPRA** (Califórnia) | Empresas com presença/receita na CA atendendo residentes |
| **HIPAA** (EUA) | Dados de saúde nos EUA |
| **PCI-DSS** | Qualquer empresa que processa cartões |
| **SOX** | Empresas listadas em bolsa nos EUA (controles financeiros) |

### Princípios comuns

- **Base legal** para cada tratamento (consentimento, contrato, obrigação legal, interesse legítimo).
- **Minimização:** colete apenas o necessário.
- **Limitação de propósito:** use apenas para o que foi declarado.
- **Direitos do titular:**
  - Acesso (exportar dados).
  - Correção.
  - Exclusão / "direito ao esquecimento".
  - Portabilidade.
  - Oposição ao tratamento.
- **Registro de tratamento:** documentação de que dados, por que, por quanto tempo.
- **Notificação de incidente:** prazos específicos por lei (LGPD não fixa em horas; GDPR exige 72h).
- **DPO / Encarregado** quando exigido.
- **Avaliação de impacto** (DPIA / RIPD) em tratamentos de alto risco.

### Implementação técnica

- Endpoint de **exportação de dados** do usuário.
- Endpoint de **exclusão** real (não soft-delete) ou pseudo-anonimização documentada.
- **Consentimento granular** quando exigido, registrado com timestamp.
- **Cookies** com banner conforme jurisdição.
- **Política de privacidade** acessível e atualizada.
- **Contratos com processadores** (DPA) quando dados saem do controle direto.
- **Localização de dados** considerada (transferência internacional pode exigir salvaguardas).

### Anti-padrões

```
❌ "Aceito tudo" como única opção de consentimento
❌ Exclusão que só esconde mas mantém o dado
❌ Compartilhar dados com terceiros sem base legal
❌ Reter dados indefinidamente
❌ Política de privacidade desatualizada
❌ Sem processo de resposta a incidente
```

---

# PARTE 6 — ANTI-PADRÕES

Lista consolidada dos erros mais comuns. Se encontrar qualquer um destes no código, é refatoração obrigatória.

## Autenticação e autorização

```
❌ Decisão de auth baseada em localStorage
❌ Decisão de auth baseada em cookie não assinado
❌ Decisão de auth baseada em campo do request body
❌ if (user.role === "admin") onde user veio do client
❌ Endpoint privado sem decorator/middleware de auth
❌ "Se não tem token, deixa passar como anônimo" em rota privada
❌ Senha em SHA-256 ou MD5
❌ JWT sem expiração
❌ Mesma chave de assinatura em dev e prod
```

## Dados e isolamento

```
❌ Query sem filtro de ownership
❌ ID sequencial em URL pública
❌ Operação em lote que valida só o primeiro item
❌ Serializar model do banco direto na resposta
❌ Retornar password_hash mesmo "filtrado por convenção"
❌ Cliente envia user_id e servidor confia
```

## Pagamentos

```
❌ Cliente envia amount/price/currency
❌ Ativar premium ao chegar em /success
❌ Webhook sem verificação de assinatura
❌ Processar mesmo event_id duas vezes
❌ Confiar em redirect URL com flag de sucesso
```

## Upload

```
❌ Confiar na extensão do arquivo
❌ Confiar no Content-Type do cliente
❌ Salvar com nome do cliente
❌ Servir uploads do mesmo domínio do app
❌ Sem limite de tamanho
```

## Configuração

```
❌ DEBUG=True em produção
❌ allow_origins=["*"] com credentials
❌ Segredo no código
❌ .env commitado
❌ Stack trace na resposta
❌ Versão do framework em header
```

## Logs

```
❌ Log de senha (mesmo errada)
❌ Log de token
❌ Log sem estrutura
❌ Log em timezone local
❌ Log sem retenção definida
```

## Criptografia

```
❌ Implementar primitivas próprias
❌ AES-ECB
❌ Math.random() para uso cripto
❌ Comparação não-tempo-constante de tokens
❌ MD5 para qualquer coisa de segurança
```

---

# PARTE 7 — TESTES ADVERSARIAIS OBRIGATÓRIOS

Antes de considerar a aplicação segura, execute manualmente cada um destes testes contra a aplicação rodando. Documente o resultado de cada um.

## 7.1 Acesso sem autenticação

Para cada rota privada, faça request sem token / sem cookie.
- **Esperado:** `401 Unauthorized`.
- **Erro:** qualquer 2xx ou 5xx.

## 7.2 Acesso autenticado a recurso alheio

Crie dois usuários (A e B). Logado como A, tente acessar recursos de B via:
- ID na URL: `GET /api/resource/{id_de_B}`.
- ID em query: `?resource_id={id_de_B}`.
- ID em body: `{ "resource_id": "{id_de_B}" }`.
- ID em header customizado.
- Cookie de tenant manipulado.

**Esperado:** `404` (ou `403` conforme política).
**Erro:** retornar o recurso de B, mesmo parcialmente.

## 7.3 Escalação de privilégio

Logado como user comum, tente:
- Alterar `role` em qualquer endpoint que aceite update de perfil.
- Alterar `plan` em endpoint similar.
- Acessar endpoints `/admin/*`.
- Alterar `localStorage.role = "admin"` e recarregar.
- Forjar JWT com `role: admin` (assinatura inválida).

**Esperado:** `403` ou request rejeitado. UI pode mudar com localStorage mas APIs retornam negação.

## 7.4 Bypass de pagamento

- Enviar `amount: 0.01` no body do checkout.
- Enviar `plan: "premium_yearly"` quando comprou `monthly`.
- Enviar `currency: "USD"` em vez de `BRL`.
- Aplicar cupom inventado.
- Chegar direto em `/success` sem pagar.
- Cancelar pagamento depois de ativar.

**Esperado:** servidor ignora ou rejeita modificações; ativação só após webhook real.

## 7.5 Bypass de limite

- Atingir limite free.
- Apagar localStorage / cookies.
- Tentar usar novamente.

**Esperado:** servidor ainda bloqueia. Contador é server-side.

## 7.6 Acesso direto a endpoint

- Endpoint premium sem ser premium.
- Endpoint admin sem ser admin.
- Endpoint de outro tenant.

**Esperado:** `401` ou `403`.

## 7.7 Mass assignment

Em qualquer endpoint que atualize entidade do usuário, envie campos extras:
```json
{
  "name": "valor legítimo",
  "user_id": "outro_id",
  "role": "admin",
  "plan": "premium",
  "is_admin": true,
  "stripe_customer_id": "..."
}
```

**Esperado:** campos extras rejeitados ou ignorados.

## 7.8 Upload malicioso

- Arquivo com extensão permitida mas conteúdo de outro tipo.
- Arquivo com nome contendo `../` ou `..\`.
- Arquivo gigante.
- Arquivo zero-byte.
- SVG com `<script>`.
- HTML disfarçado de imagem.
- ZIP bomb.
- Polyglot (JPG válido que também é HTML).

**Esperado:** rejeitado, ou aceito e processado de forma segura.

## 7.9 Injeções

Em todo campo de input:
- SQL: `' OR '1'='1`, `'; DROP TABLE users; --`
- Comando: `; rm -rf /`, `$(cat /etc/passwd)`
- Template: `{{7*7}}`, `${7*7}`
- LDAP: `*)(uid=*`
- Header: `\r\nSet-Cookie: ...`
- XSS: `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>`

**Esperado:** input tratado como dado, não como código.

## 7.10 SSRF

Se o app faz fetch baseado em input:
- `http://169.254.169.254/latest/meta-data/` (AWS metadata).
- `http://localhost:6379/` (Redis interno).
- `http://10.0.0.1/`.
- URL com redirect para o acima.

**Esperado:** bloqueado.

## 7.11 Race condition

- Clicar "pagar" duas vezes muito rápido.
- Dois requests simultâneos atingindo limite na borda.
- Criar recurso "único" em dois requests simultâneos.

**Esperado:** uma única cobrança, um único registro, conflito tratado.

## 7.12 Vazamento em erro

- Forçar erro em cada endpoint (input malformado, conflito, indisponibilidade).
- Verificar resposta.

**Esperado:** mensagem genérica. Sem stack trace, schema, paths, versões.

## 7.13 Vazamento em log

- Inspecionar logs após operação com dados sensíveis.

**Esperado:** sem senhas, tokens, cartões, PII desnecessária.

## 7.14 Headers de segurança

- Inspecionar resposta de página HTML autenticada.

**Esperado:** HSTS, CSP, X-Content-Type-Options, X-Frame-Options ou frame-ancestors, Referrer-Policy.

---

# PARTE 8 — CHECKLIST PRÉ-DEPLOY

Verificar cada item antes de subir para produção. Não suba se algum estiver não-resolvido.

## Autenticação e autorização

- [ ] Toda rota privada exige token verificado.
- [ ] Senhas armazenadas com argon2id, scrypt ou bcrypt (custo adequado).
- [ ] JWTs (se usados) têm expiração e mecanismo de revogação.
- [ ] MFA disponível; obrigatório para admins.
- [ ] Rate limit em login, reset, signup.
- [ ] Sem enumeração de usuários em erros.
- [ ] Reset de senha invalida sessões anteriores.

## Isolamento de dados

- [ ] Toda entidade tem `user_id` (ou `tenant_id`).
- [ ] Toda query filtra por ownership.
- [ ] IDs públicos são opacos (UUID/ULID), não sequenciais.
- [ ] Operações em lote validam cada item.

## Validação e sanitização

- [ ] Schema explícito por endpoint.
- [ ] Mass assignment bloqueado.
- [ ] Limites de tamanho (string, array, payload).
- [ ] Encoding contextual na saída.
- [ ] DTOs explícitos para responses.

## Pagamentos

- [ ] Cliente envia só planKey, nunca preço.
- [ ] Servidor resolve preço internamente.
- [ ] Webhook com verificação de assinatura.
- [ ] Idempotência por event_id.
- [ ] Estado de assinatura no banco, derivado do provedor.
- [ ] Ativação apenas após webhook, nunca por redirect.

## Limites de uso

- [ ] Limites calculados server-side.
- [ ] Eventos persistidos no banco.
- [ ] Verificação antes da operação cara.
- [ ] Plano vem do banco, não do cliente.

## Segredos

- [ ] Nenhum segredo no código.
- [ ] `.env` no `.gitignore`.
- [ ] Segredos diferentes por ambiente.
- [ ] Histórico do git verificado (sem segredos antigos).
- [ ] Pre-commit hook contra segredos.

## Criptografia

- [ ] TLS 1.2+ em todo transporte externo.
- [ ] HSTS ativo.
- [ ] Certificados com renovação automatizada.
- [ ] Bibliotecas de cripto auditadas; nada implementado manualmente.
- [ ] CSPRNG usado para uso criptográfico.
- [ ] Comparação tempo-constante para tokens.

## Upload

- [ ] Tamanho limitado server-side.
- [ ] Tipo validado por conteúdo.
- [ ] Nome gerado pelo servidor.
- [ ] Servido de domínio separado ou via URL assinada.
- [ ] Headers `nosniff` + `Content-Disposition`.

## Configuração

- [ ] `DEBUG=False` em produção, validado na startup.
- [ ] CORS com origens explícitas.
- [ ] Cabeçalhos de segurança configurados.
- [ ] Variáveis obrigatórias validadas na startup.
- [ ] Servidor não roda como root.

## Logs e observabilidade

- [ ] Logs estruturados com timestamp UTC.
- [ ] Correlation ID em todo request.
- [ ] Sem segredos / senhas / tokens em logs.
- [ ] Eventos de segurança logados.
- [ ] Alerta configurado para padrões suspeitos.
- [ ] Retenção definida.

## Erros

- [ ] Mensagens genéricas em produção.
- [ ] Sem stack trace na resposta.
- [ ] Sem schema, paths, versões expostas.
- [ ] Códigos HTTP corretos.

## Dependências

- [ ] Lockfile commitado.
- [ ] Scanner de CVE rodando em CI.
- [ ] Nenhuma CVE crítica conhecida sem mitigação.
- [ ] Imagens base atualizadas.

## DoS

- [ ] Timeouts em toda I/O externa.
- [ ] Paginação obrigatória em listagens.
- [ ] Limites de payload no proxy e no app.
- [ ] Operações caras isoladas em workers.

## Conformidade

- [ ] Legislação aplicável identificada.
- [ ] Política de privacidade publicada e atualizada.
- [ ] Endpoints de acesso, exportação e exclusão de dados.
- [ ] Consentimento granular onde exigido.
- [ ] Processo de resposta a incidente documentado.

## Documentação

- [ ] `SECURITY.md` com modelo de confiança e contatos.
- [ ] Runbook de incidente.
- [ ] Política de divulgação de vulnerabilidade.

---

# PARTE 9 — CRITÉRIOS DE ACEITE

A aplicação é considerada pronta apenas quando, **simultaneamente**:

1. **Frontend não decide nada de segurança.** Identidade, papel, plano, propriedade, preço, limite — todos derivados pelo servidor.
2. **Backend é fonte de verdade** em toda decisão crítica.
3. **Toda rota privada nega por padrão.**
4. **Toda entidade sensível tem dono** e toda query filtra por dono.
5. **Nenhum usuário consegue, nos testes adversariais, acessar dados de outro.**
6. **Nenhum usuário consegue virar admin** mexendo em cliente.
7. **Nenhum usuário consegue ativar plano pago** sem confirmação assíncrona verificada.
8. **Nenhum usuário consegue burlar limite** mexendo em cliente.
9. **Nenhum segredo está exposto** em código, log, resposta de API ou bundle.
10. **Erros em produção não vazam** stack trace, schema, paths ou existência de recursos alheios.
11. **Todos os achados Críticos e Altos** estão corrigidos ou aceitos com justificativa documentada.
12. **Todos os achados Críticos e Altos** têm teste automatizado de regressão.
13. **Testes adversariais foram executados** e os resultados documentados.
14. **Documentação de segurança existe** e descreve o sistema real, não o ideal.
15. **Pendências fora do escopo** estão listadas, não escondidas.

---

# APÊNDICE A — GLOSSÁRIO DE SEVERIDADE

Use rigidamente. Inflar severidade é o oposto de rigor.

| Severidade | Critério |
|------------|----------|
| **Crítica** | Acesso a dados de outros usuários sem autenticação ou com autenticação básica. Escalação para admin. Bypass de pagamento. RCE. Leak de segredos em produção. Vulnerabilidade explorável sem conta. |
| **Alta** | Bypass de feature paga. IDOR limitado a recursos específicos. Leak de PII parcial. Sessão forjável com esforço razoável. CSRF em ação destrutiva. Falta de rate limit em endpoint sensível (login, reset). |
| **Média** | Falta de validação que não leva a impacto direto. Log que vaza informação útil ao atacante mas não crítica. Configuração permissiva mitigada por outra camada. Falta de header de hardening em endpoint sensível. |
| **Baixa** | Hardening ausente sem exploração imediata. Configuração subótima. Boa prática não seguida sem impacto demonstrável. |
| **Informativa** | Desvio de boa prática puramente. Documentação. Estilo. |

---

# APÊNDICE B — REFERÊNCIAS EXTERNAS

Para aprofundamento. Não substitui auditoria; complementa.

- **OWASP Top 10:** atualizado periodicamente, categoriza os riscos mais comuns. <https://owasp.org/Top10/>
- **OWASP Application Security Verification Standard (ASVS):** padrão detalhado de requisitos. <https://owasp.org/ASVS/>
- **OWASP Cheat Sheet Series:** referências técnicas concisas por tópico. <https://cheatsheetseries.owasp.org/>
- **CWE (Common Weakness Enumeration):** taxonomia de fraquezas. <https://cwe.mitre.org/>
- **NIST SP 800-63 (Identity):** padrão de autenticação. <https://pages.nist.gov/800-63-3/>
- **NIST Cybersecurity Framework:** estrutura geral de segurança organizacional.
- **MITRE ATT&CK:** taxonomia de táticas de adversários.
- **Mozilla Web Security Guidelines:** referência prática de configuração web. <https://infosec.mozilla.org/guidelines/web_security>

---

**Versão do documento:** 1.0
**Aplicável a:** qualquer aplicação web/SaaS, qualquer linguagem, qualquer stack
**Princípio único:** *o servidor é a fonte de verdade; o cliente é hostil até prova em contrário.*
