# Padrões de Engenharia

Documento de referência para todos os projetos. Copie para a raiz do repositório ou referencie no `CLAUDE.md`.

Itens marcados com **[obrigatório]** valem em qualquer projeto. Os demais se aplicam conforme o contexto. Quando um padrão for violado de propósito, registre o motivo em um ADR (`docs/adr/NNNN-titulo.md`).

---

## 1. Arquitetura

- **[obrigatório]** Separe as responsabilidades em camadas: entrada (rotas/controllers), aplicação (casos de uso), domínio (regras de negócio), infraestrutura (banco, filas, APIs externas).
- **[obrigatório]** Regra de negócio não mora em controller, componente de UI, trigger escondido ou middleware de framework.
- As dependências apontam para dentro. O domínio não importa infraestrutura. Banco, gateways e e-mail entram por interface (adapter), o que facilita teste e troca de fornecedor.
- **[obrigatório] Estilo padrão: monolito modular.** Um único deploy, dividido em módulos por contexto de negócio (ex.: `pagamentos`, `pedidos`, `usuarios`), não por camada técnica. Microsserviços só com justificativa registrada em ADR (ver critérios abaixo).
  - **[obrigatório] Cada módulo expõe uma interface pública** (arquivo/pacote `api` ou `index`). Todo o resto é interno e não pode ser importado de fora.
  - **[obrigatório] Fronteiras impostas por ferramenta no CI**, não por convenção: `import-linter` (Python), `dependency-cruiser` ou Nx boundaries (JS/TS), ArchUnit (Java) ou equivalente. Violação de fronteira quebra o build.
  - **[obrigatório] Cada módulo é dono das suas tabelas** (schema ou prefixo próprio). Nada de `JOIN` ou `FOREIGN KEY` entre tabelas de módulos diferentes. Para ler dado de outro módulo, chame a interface pública dele. Se a consulta cruzada for realmente necessária (relatório), use uma view/read model explícito, mantido por um módulo só.
  - Comunicação entre módulos por chamada direta à interface pública ou por eventos internos. Eventos que mudam estado em outro módulo devem ser idempotentes (seção 5).
  - Processos separados do mesmo código para web, workers e jobs agendados, com entrypoints distintos. Um job pesado não pode derrubar a API.
  - **Quando extrair um módulo para serviço próprio** (todos os critérios devem ser medidos, não supostos): escala exigida muito diferente do resto, ciclo de deploy que bloqueia outros times, requisito de isolamento de falha ou de segurança (ex.: escopo PCI), ou time dedicado grande. "Está ficando grande" e "é mais moderno" não são critérios.
  - Um módulo bem isolado (interface pública, tabelas próprias, sem imports internos de fora) é o pré-requisito para uma extração futura barata. Se o módulo não passa nesse teste, extrair vai custar caro.
- Módulos têm fronteira explícita. Um módulo não lê tabelas internas de outro; usa a interface pública dele.
- **[obrigatório]** Configuração vem de variáveis de ambiente validadas na inicialização. Se faltar valor obrigatório, a aplicação não sobe.
- Não crie abstração antes do segundo caso de uso real. Abstração prematura custa tanto quanto duplicação.
- Decisões que afetam a estrutura (banco, fila, autenticação, provedor de pagamento) são registradas em ADR curto.

## 2. Qualidade de código

- **[obrigatório]** Lint, formatter e checagem de tipos rodam no CI e bloqueiam merge.
- **[obrigatório]** Erros não são engolidos. Nada de `catch {}` vazio. Erros são tipados ou carregam contexto suficiente para diagnóstico.
- Testes:
  - Unitários para regras de negócio.
  - Integração para repositórios e endpoints (com banco real, não mock de banco).
  - Poucos testes de ponta a ponta, só para fluxos críticos (login, checkout, pagamento).
  - Cobertura é sinal, não meta. Priorize caminhos de falha, idempotência e autorização.
- Funções pequenas. Evite parâmetro booleano que muda o comportamento da função; prefira duas funções com nomes claros.
- Comentário explica o **porquê**, não o **o quê**. Código morto sai, não fica comentado.
- Migrations são versionadas. Migration já aplicada não é editada; cria-se outra.
- Dependências com lockfile commitado e atualização automatizada (Dependabot ou Renovate).
- Nomes em um único idioma por projeto.

## 3. Segurança

### 3.1 Princípios

- **[obrigatório] O backend é a única autoridade.** Autenticação, autorização, cálculo de preço, regras de estado e validação de negócio acontecem no servidor. O frontend pode esconder botões e validar formulário para dar boa experiência, mas isso não é controle de acesso. Quem decide é sempre o servidor, mesmo que a UI já tenha bloqueado a ação.
- **[obrigatório] Toda requisição checa a permissão sobre o recurso específico.** Não basta estar logado. Quando um usuário pede `/pedidos/123`, o servidor confirma que o pedido 123 pertence a ele. Sem essa checagem, há IDOR (acesso a recurso alheio trocando o ID).
- **[obrigatório] Negar por padrão.** Rota nova nasce privada. Ser pública exige marcação explícita.
- **[obrigatório]** Nunca aceite do cliente valores que o servidor deveria calcular ou decidir: preço, desconto, `role`, `user_id` de dono, status de pagamento, saldo. Leia do banco.
- **Menor privilégio.** Usuário de banco, token de serviço e permissão de cloud só têm o que precisam.
- **[obrigatório] Segredos** ficam fora do código e do repositório (gerenciador de segredos ou variável de ambiente). São rotacionáveis e nunca aparecem em log.

### 3.2 Autenticação e sessão

- **[obrigatório]** Senhas com argon2id ou bcrypt. Nunca MD5 ou SHA puro.
- **[obrigatório]** Cookie de sessão com `HttpOnly; Secure; SameSite=Lax` (ou `Strict` quando o fluxo permitir). `SameSite=None` só com necessidade real de cross-site, e com proteção CSRF.
- **[obrigatório]** Proteção CSRF em requisições que mudam estado e usam cookie. `SameSite` ajuda, mas não substitui token CSRF ou validação de `Origin`.
- Sessão com expiração absoluta e por inatividade. Regenerar o ID de sessão no login e em qualquer elevação de privilégio.
- Logout invalida a sessão no servidor, não só apaga o cookie no cliente.
- Rate limit em login, recuperação de senha e OTP.
- Mensagem de erro de login genérica. Não revelar se o e-mail existe.
- MFA obrigatório para contas administrativas.
- JWT: não guardar em `localStorage`. Validar `alg`, `exp`, `iss` e `aud`. Para aplicação web, sessão opaca no servidor costuma ser mais simples e revogável.

### 3.3 Entrada e saída

- **[obrigatório] Validar tudo na borda com schema** (tipo, formato, tamanho máximo, enum, faixa). Rejeitar com erro claro, sem "corrigir" silenciosamente.
- **[obrigatório] Validar de novo na regra de negócio** (saldo suficiente, estado permitido, prazo válido). A validação de formato não garante regra de negócio.
- **[obrigatório] SQL sempre parametrizado** (ORM ou query com bind). Nunca concatenar input.
- **[obrigatório] XSS:**
  - Escape automático do template engine. Não desligue.
  - Não usar `innerHTML`, `dangerouslySetInnerHTML` ou `v-html` com dado de usuário.
  - Rich text: sanitizar com biblioteca testada (DOMPurify no cliente, equivalente no servidor).
  - Escape depende do contexto: HTML, atributo, JavaScript e URL são escapes diferentes.
  - Content-Security-Policy restritiva (sem `unsafe-inline` quando possível).
- **Mass assignment:** whitelist explícita dos campos aceitos em cada endpoint de escrita.
- **Upload:** validar o tipo real pelo conteúdo (magic bytes), não só pela extensão. Limitar tamanho. Gerar nome no servidor. Armazenar em bucket privado. Não servir com MIME que o navegador executa.
- **SSRF:** ao buscar uma URL informada pelo usuário, usar allowlist de hosts e bloquear IPs privados e de metadata (ex.: 169.254.169.254).
- **Path traversal:** nunca montar caminho de arquivo com input bruto. Resolver e checar que continua dentro do diretório base.
- **Desserialização:** não usar `pickle`, `unserialize` ou equivalente com dado não confiável.
- **Erros:** o cliente recebe mensagem genérica com um `request_id`. Stack trace fica só no log.

### 3.4 Headers e transporte

- **[obrigatório]** HTTPS em todo lugar. `Strict-Transport-Security` habilitado.
- **[obrigatório]** Headers: `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `frame-ancestors 'none'` (ou `X-Frame-Options: DENY`), `Permissions-Policy` restritiva.
- **CORS:** lista explícita de origens. Nunca `*` junto com credenciais.
- Remover headers que revelam a stack (`X-Powered-By`, versão do servidor).

### 3.5 Dependências e supply chain

- **[obrigatório]** Scan de dependências no CI (`npm audit`, `pip-audit`, Trivy ou equivalente).
- SAST no CI (Semgrep ou equivalente).
- Imagens de container mínimas, rodando como usuário não-root.
- Antes de adicionar pacote, verificar manutenção, downloads e histórico de segurança.

### 3.6 Segredos

- **[obrigatório]** Gitleaks no pre-commit e no CI.
- Se um segredo vazou, **rotacione**. Apagar o commit não basta, porque o segredo já foi copiado.

## 4. LGPD e dados pessoais

Este bloco é guia de engenharia, não parecer jurídico. Para produto real, valide com jurídico e com o encarregado (DPO).

- **[obrigatório] Inventário de dados:** para cada dado pessoal, registre o que é coletado, para qual finalidade, base legal (art. 7º, ou art. 11º para dado sensível), onde fica e por quanto tempo.
- **Minimização:** não colete o que não usa.
- **Finalidade:** não use dado coletado para uma finalidade nova sem base legal.
- **[obrigatório] Dados de cartão:** nunca armazene número completo. Use tokenização do gateway (escopo PCI DSS).
- **[obrigatório] PII fora dos logs e traces:** CPF, e-mail, telefone, endereço, tokens e senhas são removidos ou mascarados no logger. Mantenha uma lista de campos proibidos aplicada automaticamente.
- **Retenção:** prazo definido por tipo de dado, com job que elimina ou anonimiza ao vencer.
- **Direitos do titular:** existe caminho (endpoint ou processo) para acesso, correção, exclusão e portabilidade.
- **Consentimento:** quando for a base legal, registre aceite, versão do termo e data.
- **Incidentes:** existe processo documentado para comunicar à ANPD e aos titulares dentro do prazo que a regulamentação exige.
- **Operadores:** fornecedores que tratam dados (cloud, e-mail, analytics) têm contrato com cláusulas de proteção de dados.
- **Ambientes não-produtivos** não recebem dado real sem anonimização.
- Acesso a dados sensíveis gera log de auditoria.

## 5. Dinheiro e idempotência

- **[obrigatório] Valores monetários** em inteiro (centavos) ou `NUMERIC`/`Decimal`. Nunca `float`.
- **[obrigatório] Webhooks:**
  - Validar assinatura HMAC com comparação em tempo constante.
  - Checar timestamp para evitar replay.
  - Responder 2xx após persistir o evento, não após processar tudo.
- **[obrigatório] Deduplicação no banco, não em memória.** Crie a tabela `processed_events` com `UNIQUE (provider, event_id)`. A checagem em `if` com cache ou Redis, sem constraint, tem condição de corrida: duas instâncias recebem o mesmo evento ao mesmo tempo e processam os dois.

  Fluxo recomendado, tudo na mesma transação:

  ```
  BEGIN;
    INSERT INTO processed_events (provider, event_id) VALUES (...);
    -- se violar UNIQUE: já processado, ROLLBACK e responde 200
    -- aplica o efeito (ex.: marca pedido como pago, cria lançamento)
  COMMIT;
  ```

  Se o efeito falhar, o rollback também desfaz o registro do evento, e o retry processa de novo.
- **[obrigatório] Valide a transição de estado, não só o ID.** Mesmo com dedupe, um evento atrasado pode chegar fora de ordem. Pagamento já `paid` não volta para `pending`. Use máquina de estados com transições permitidas.
- **[obrigatório] Update condicional ou lock.** Ex.: `UPDATE pedidos SET status='paid' WHERE id=? AND status='pending'` e checar o número de linhas afetadas. Ou `SELECT ... FOR UPDATE` quando houver cálculo.
- **Endpoints de criação** chamados pelo cliente (pagamento, pedido) aceitam header `Idempotency-Key`. A resposta fica guardada por um período, e a repetição devolve a mesma resposta.
- **Ledger imutável:** lançamentos são inseridos, nunca editados. Saldo é derivado ou conferido contra o ledger.
- **Reconciliação:** job periódico compara o que o gateway informa com o que está no banco e alerta divergências.
- **Retries** com backoff exponencial e jitter, limite de tentativas e dead letter queue. Chamada externa que cobra só é repetida com idempotency key do provedor.
- Métrica para eventos duplicados descartados (`webhook_duplicates_total`). Subir de repente indica problema no provedor ou na rede.

## 6. Rate limit

- **[obrigatório]** Aplicar em: login, cadastro, recuperação de senha, OTP, endpoints caros e APIs públicas.
- Chave do limite: IP, conta ou API key, conforme o caso.
- Atrás de proxy, confie só no `X-Forwarded-For` do proxy conhecido. Caso contrário, o cliente forja o IP.
- Algoritmo: token bucket ou sliding window. Com mais de uma instância, o contador fica em store compartilhado (Redis).
- Resposta `429` com `Retry-After`, sem detalhar a regra.
- Limite na borda (gateway, WAF, Nginx) e, quando houver regra de negócio, também na aplicação.
- Webhooks de terceiros não devem ser limitados de forma agressiva. Se o limite rejeitar, o provedor pode perder evento. A idempotência cobre os retries.
- Métrica de `429` por rota.

## 7. Banco de dados e N+1

- **[obrigatório] Constraints no banco:** `NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`. Não delegue tudo à aplicação.
- **[obrigatório] Índice em toda FK** e em colunas de filtro frequente. Rode `EXPLAIN` em query nova.
- **[obrigatório] Paginação em toda listagem**, com limite máximo de tamanho de página. Prefira paginação por cursor.
- **N+1:**
  - Detecte em desenvolvimento e teste, com ferramenta do ORM (logs de SQL, `django-debug-toolbar`, estatísticas do Hibernate, equivalente).
  - Escreva teste que afirme o número máximo de queries em endpoints de listagem.
  - Corrija com eager loading, join ou batch (`WHERE id IN (...)`, DataLoader).
  - Nunca faça query dentro de loop.
- **Transações curtas.** Chamada HTTP externa nunca acontece dentro de transação aberta.
- Timeout de statement e de conexão. Pool dimensionado e medido.
- **Migrations seguras:** evite lock longo em tabela grande. Use padrão expand/contract para renomear ou remover coluna em uso.
- **Backups** com restore testado periodicamente. Backup não restaurado não é backup.

## 8. Observabilidade

Implemente antes de precisar. Diagnosticar incidente sem telemetria custa muito mais do que instrumentar no início.

### 8.1 Escolha da stack

- **OpenTelemetry (OTel)** é a camada de instrumentação. Use SDK e collector OTel no código, porque é padrão e não prende a um fornecedor.
- **Backend** (escolha uma por projeto ou organização):
  - Open source: Prometheus (métricas), Loki (logs), Tempo (traces), Grafana (visualização).
  - SaaS: New Relic ou Datadog. Recebem OTel diretamente.
- Não rode duas stacks completas em paralelo sem motivo. Duplica custo e complexidade.
- Defina retenção e sampling desde o início. SaaS cobra por volume, e stack própria consome disco.

### 8.2 Logs estruturados

- **[obrigatório]** Formato JSON, uma linha por evento.
- Campos mínimos: `timestamp`, `level`, `service`, `env`, `version`, `request_id`, `trace_id`, `event`, `message`, `duration_ms` quando aplicável.
- `event` é um nome fixo e pesquisável (ex.: `payment.approved`, `webhook.duplicate_ignored`), não texto livre.
- Sem PII em claro (ver seção 4).
- Níveis: `error` para falha que exige ação, `warn` para anomalia tolerada, `info` para eventos de negócio, `debug` desligado em produção.

### 8.3 Identificador por request

- **[obrigatório]** Gere um `request_id` (UUID) na entrada da aplicação. Aceite `X-Request-ID` do cliente ou do gateway só se estiver em formato válido.
- Propague o ID em logs, traces, chamadas para outros serviços e filas (no payload ou nos headers da mensagem).
- Devolva o ID no header de resposta e no corpo de erro. É o que o suporte pede quando o usuário reclama.
- Use o `trace_id` do OTel como correlação entre logs e traces.

### 8.4 Métricas

- **[obrigatório] Por rota:** RED (taxa de requisições, taxa de erros, latência em histograma com p50, p95, p99).
- Por recurso: USE (utilização, saturação, erros) para CPU, memória, pool de conexão, fila.
- **Cardinalidade:** nunca use `user_id`, `request_id` ou e-mail como label de métrica. Explode o custo.
- Métricas de negócio: pagamentos aprovados, webhooks duplicados descartados, falhas de reconciliação.

### 8.5 Traces

- Spans em entrada HTTP, query de banco, chamada externa, publicação e consumo de fila.
- Sampling: amostragem de traces de sucesso, manter 100% dos com erro quando possível.

### 8.6 Error tracking

- **[obrigatório]** Ferramenta dedicada (Sentry ou equivalente) com agrupamento de erros, versão (release) e ambiente.
- Scrub de PII antes de enviar.
- Erro que aparece em produção tem dono e está linkado a uma issue.

### 8.7 Alertas e SLO

- **[obrigatório]** Alerte em sintoma (taxa de erro, latência acima do SLO, fila crescendo, webhook falhando), não em causa.
- Poucos alertas, todos acionáveis, cada um com link para runbook.
- SLO por serviço (ex.: 99,9% das requisições abaixo de 500 ms no mês).
- Health checks separados: liveness (processo vivo) e readiness (pronto para tráfego, dependências ok).

### 8.8 Dashboards mínimos por serviço

Tráfego, erros, latência, saturação e status das dependências (banco, cache, fila, APIs externas).

## 9. Entrega

- **[obrigatório]** Pipeline de CI: lint, tipos, testes, SAST, scan de dependências, scan de segredos.
- Deploy com rollback testado. Mudança arriscada entra atrás de feature flag.
- Ambientes separados com configuração própria. Produção não é usada como ambiente de teste.
- Logs de deploy e versão em cada requisição (ligados ao error tracking).

## 10. Checklist de revisão de PR

- [ ] A autorização é checada no backend, sobre o recurso específico?
- [ ] Todo input é validado com schema, e a regra de negócio é validada de novo?
- [ ] Sem SQL concatenado, sem `innerHTML` com dado do usuário?
- [ ] Sem segredo no código, sem PII em log?
- [ ] Erros tratados, sem `catch` vazio, mensagem genérica para o cliente?
- [ ] Query nova tem índice, sem N+1, com paginação?
- [ ] Operação com dinheiro é idempotente, transacional e protegida por constraint?
- [ ] Endpoint novo tem auth, rate limit, `request_id` nos logs e métrica?
- [ ] Testes cobrem sucesso e falha?
- [ ] Dado pessoal novo tem finalidade, base legal e retenção definidos?
- [ ] Migration não bloqueia tabela grande e é reversível (ou tem plano)?

## 11. Setup inicial de projeto novo

- [ ] `.env.example` com todas as variáveis, e validação de config na inicialização
- [ ] Lint, formatter, checagem de tipos e CI bloqueando merge
- [ ] Pre-commit com Gitleaks
- [ ] Headers de segurança e CORS restritivo
- [ ] Logger estruturado com redação de PII
- [ ] Middleware de `request_id`
- [ ] Health checks (liveness e readiness)
- [ ] OpenTelemetry configurado e error tracking ligado
- [ ] Rate limit padrão nas rotas sensíveis
- [ ] Migrations versionadas e backup com restore testado
- [ ] Tabela `processed_events` se houver webhook
- [ ] Inventário de dados pessoais iniciado
