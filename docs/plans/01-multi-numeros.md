# Plano técnico: vários números por conta no wacrm

Nov 5, 2026 · @Sérgio Bismark

## Resumo e decisões de projeto

Cada conta passa a ter vários números de WhatsApp (conexões) e cada conversa pertence a exatamente um número. Hoje o código inteiro assume "uma conta = uma linha em `whatsapp_config`": são 25 arquivos de `src` e 11 migrations que tocam essa tabela.

| Decisão | Escolha | Motivo |
| --- | --- | --- |
| Tabela das conexões | Manter `whatsapp_config`, remover o UNIQUE(account\_id), adicionar colunas | 25 arquivos e 11 migrations já usam o nome; renomear dobra o diff sem ganho |
| Conversa | Uma por contato por número: unique (account\_id, contact\_id, connection\_id) | O cliente fala com cada número separadamente, como no próprio WhatsApp |
| Templates | Escopo por WABA (`waba_id`), não por número | Na Meta o template pertence ao WABA; números do mesmo WABA compartilham |
| Respostas automáticas (fluxos, automações, IA) | Saem sempre pelo número da conversa | Nunca responder por um número que o cliente não contatou |
| Envio sem conversa (broadcast, API v1) | `connection_id` explícito, com o número padrão como fallback | Chamadas antigas continuam funcionando |
| IA | Provedor, modelo e chave seguem por conta; o contexto (papel, tom e jeito de agir) é por número e o conhecimento da empresa é um só | Um número pode ser de vendas e outro de suporte, e a IA se comporta conforme o papel de cada um, sem pedir uma chave por número |
| Canal | Só a API oficial da Meta agora; coluna `provider` recebe o Zernio no plano de provedores e Instagram, e o GOWA fica para depois | Mantém o escopo pequeno sem fechar a porta |

Decidido: cada número tem o próprio contexto para a IA (papel, tom e jeito de agir), enquanto o conhecimento da empresa é um só, compartilhado por todos os números. Pode haver vários números ativos ao mesmo tempo; o admin da conta escolhe um padrão, usado para contatos novos e para a primeira conversa iniciada pelo sistema. Quem já conversa com um número continua nele, e quem escreve para um número é atendido por ele.

## Como o código amarra um número por conta hoje

O acoplamento aparece em três formas: consultas `.eq('account_id', ...).single()` em `whatsapp_config`, índices únicos que não conhecem o número, e telas que supõem uma configuração só.

| Camada | Onde | O que assume hoje |
| --- | --- | --- |
| Banco | `017_account_sharing.sql` | `UNIQUE(account_id)` em `whatsapp_config`; o próprio comentário diz "one WhatsApp number per account" e sugere remover o unique e criar um número principal |
| Banco | `036_conversation_contact_dedup.sql` | Índice `idx_conversations_account_contact` em (account\_id, contact\_id): uma conversa por contato na conta |
| Banco | `057_message_templates_account_unique.sql` | Índice `message_templates_account_name_language_key`: um template por nome e idioma na conta |
| Webhook de entrada | `api/whatsapp/webhook/route.ts` | Acha a conta por `metadata.phone_number_id` (linhas 285 a 299 para status, 349 a 399 para mensagens) e descarta se achar mais de uma linha |
| Envio manual | `lib/whatsapp/send-message.ts` | Busca a config pela conta (linha 263), sem olhar a conversa |
| Fluxos e automações | `lib/flows/meta-send.ts`, `lib/automations/meta-send.ts` | `loadAccountMetaCredentials(db, accountId)` usa `.single()` e devolve um número só |
| Broadcast | `broadcast-core.ts`, `broadcast-resume.ts` | Lê a config da conta; a tabela `broadcasts` não guarda de qual número saiu |
| Templates | `template-webhook.ts`, `templates/sync` | Resolve a conta pelo `waba_id`; hoje se mais de uma linha casar, trata como erro |
| IA | `lib/ai/auto-reply.ts` | Credenciais pela conta; `ai_configs` é UNIQUE(account\_id) |
| API v1 | `lib/api/v1/contacts.ts` (linha 78) | Config única da conta |
| Interface | `inbox/page.tsx` (linha 211), `settings/whatsapp-config.tsx`, `settings-overview.tsx` | Um formulário, um status; o inbox lê uma config |

Os usos em `047` (checagem de conta vazia) e `059` (flag `whatsappConnected` do dashboard) são consultas `EXISTS` e funcionam com várias linhas. Faltam `019`, `028`, `029`, `030` e `039`, na lista de verificação da fase 0.

## Modelo de dados novo

A conexão é a própria linha de `whatsapp_config`; conversas, broadcasts e templates passam a apontar para ela ou para o WABA dela.

```sql
-- whatsapp_config vira "conexões": várias linhas por conta
ALTER TABLE whatsapp_config DROP CONSTRAINT IF EXISTS whatsapp_config_account_id_key;
ALTER TABLE whatsapp_config
  ADD COLUMN label                TEXT NOT NULL DEFAULT 'Principal',
  ADD COLUMN display_phone_number TEXT,
  ADD COLUMN provider             TEXT NOT NULL DEFAULT 'meta',
  ADD COLUMN is_default           BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN archived_at          TIMESTAMPTZ;

-- exatamente um número padrão ativo por conta (escolhido pelo admin),
-- usado para contatos novos e conversas iniciadas pelo sistema
CREATE UNIQUE INDEX whatsapp_config_one_default
  ON whatsapp_config(account_id) WHERE is_default AND archived_at IS NULL;

-- conversas: uma por contato por número
ALTER TABLE conversations
  ADD COLUMN connection_id UUID REFERENCES whatsapp_config(id) ON DELETE RESTRICT;
DROP INDEX IF EXISTS idx_conversations_account_contact;
CREATE UNIQUE INDEX idx_conversations_account_contact_connection
  ON conversations(account_id, contact_id, connection_id);

-- broadcasts: de qual número saiu
ALTER TABLE broadcasts
  ADD COLUMN connection_id UUID REFERENCES whatsapp_config(id) ON DELETE RESTRICT;

-- templates: escopo por WABA
ALTER TABLE message_templates ADD COLUMN waba_id TEXT;
DROP INDEX IF EXISTS message_templates_account_name_language_key;
CREATE UNIQUE INDEX message_templates_account_waba_name_language_key
  ON message_templates(account_id, COALESCE(waba_id, ''), name, language);

-- IA: contexto (papel e jeito de agir) por número
-- provedor, modelo, chave, prompt da empresa e base de conhecimento
-- continuam em ai_configs e ai_knowledge_*, por conta
CREATE TABLE ai_connection_profiles (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id    UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  connection_id UUID NOT NULL UNIQUE REFERENCES whatsapp_config(id) ON DELETE CASCADE,
  role_label    TEXT,            -- ex.: Vendas, Suporte
  system_prompt TEXT,            -- contexto deste número: como falar e agir
  auto_reply_enabled BOOLEAN NOT NULL DEFAULT true,
  auto_reply_max_per_conversation INTEGER
    CHECK (auto_reply_max_per_conversation BETWEEN 1 AND 20),  -- nulo = usa o da conta
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- RLS no mesmo molde de ai_configs: leitura is_account_member, escrita admin
```

`messages` não muda: herda o número pela conversa. `ON DELETE RESTRICT` impede apagar um número com histórico; o fluxo normal é arquivar (`archived_at`), que para de receber e enviar mas preserva as conversas.

O `phone_number_id` continua único no sistema inteiro (a `013` já cuida disso, a confirmar no SQL na fase 0). É essa unicidade que permite ao webhook achar a conta e o número com uma única consulta.

## Migrations

Quatro migrations novas, em ordem, para que cada uma possa ser aplicada e revertida sozinha. A numeração continua depois da `059`.

| Migration | O que faz | Segurança |
| --- | --- | --- |
| `060_whatsapp_connections.sql` | Remove `whatsapp_config_account_id_key`; adiciona `label`, `display_phone_number`, `provider`, `is_default`, `archived_at`; marca a linha existente de cada conta como `is_default = true` | Nenhuma linha é apagada; o comportamento não muda porque cada conta ainda tem 1 conexão |
| `061_conversations_by_connection.sql` | Adiciona `conversations.connection_id` e `broadcasts.connection_id`; preenche com a conexão padrão da conta; só então torna NOT NULL e troca o índice único | Backfill antes do NOT NULL, na mesma transação; o novo índice é mais frouxo que o antigo, então não pode falhar por duplicados |
| `062_templates_by_waba.sql` | Adiciona `message_templates.waba_id`, preenche a partir da conexão padrão, troca o índice único | Conta sem `waba_id` (campo é nullable desde a `001`) fica com `waba_id` nulo e continua única pelo COALESCE |
| `063_ai_per_connection.sql` | Cria `ai_connection_profiles` e um perfil vazio para cada conexão existente | Tabela nova, sem tocar em `ai_configs` nem no conhecimento; a IA responde igual a hoje até alguém preencher um perfil; reverte sozinha |

Backfill de conversas:

```sql
UPDATE conversations c
SET connection_id = w.id
FROM whatsapp_config w
WHERE w.account_id = c.account_id AND w.is_default AND c.connection_id IS NULL;
```

Conta que nunca configurou WhatsApp não tem conversas, então o NOT NULL passa. Se houver conversas de conta sem nenhuma linha em `whatsapp_config` (importação, por exemplo), elas precisam de uma conexão placeholder arquivada; checar antes de aplicar:

```sql
SELECT count(*) FROM conversations c
WHERE NOT EXISTS (SELECT 1 FROM whatsapp_config w WHERE w.account_id = c.account_id);
```

Reverter a `061` só é possível enquanto nenhuma conta tiver dois números com conversa do mesmo contato; depois disso o índice antigo não recria. Por isso a tela de adicionar o segundo número só entra na fase 4, depois de o resto estar estável em produção.

## Webhook de entrada

A Meta continua enviando tudo para a mesma URL, `/api/whatsapp/webhook`. A mudança é que o `phone_number_id` do payload passa a identificar uma conexão, e a conexão carrega a conta.

1. A assinatura HMAC continua sendo verificada primeiro, com `verifyMetaWebhookSignature`, antes de qualquer consulta ao banco.
2. Uma única consulta `whatsapp_config WHERE phone_number_id = $1 AND archived_at IS NULL` devolve `id`, `account_id` e `user_id`. Ela substitui os dois pontos atuais (status em torno da linha 285, mensagens em torno da 349).
3. Se vier mais de uma linha, o código atual loga erro e descarta; mantemos isso, porque o índice único da `013` torna o caso um sinal de dado corrompido.
4. `resolveConversationByPhone` ganha o parâmetro `connectionId` e passa a procurar e criar a conversa por (account\_id, contact\_id, connection\_id). O contato continua único por conta: o mesmo cliente que escreve para dois números é um contato com duas conversas.
5. Fluxos e automações disparam como hoje, com `connectionId` no contexto do gatilho; a resposta sai pelo número da conversa.
6. A IA carrega o perfil do número (`ai_connection_profiles`) e só responde se o `auto_reply_enabled` do perfil e o da conta forem verdadeiros.

Eventos de status (entregue, lido) já chegam com o `phone_number_id` e são casados pelo `message_id`, então mudam pouco: só a consulta de conta passa a vir da conexão.

Eventos de template são outro caso. `template-webhook.ts` acha a conta pelo `waba_id`, e dois números do mesmo WABA geram duas linhas com a mesma conta. A regra de hoje (mais de uma linha, não cria stub) precisa virar: deduplicar por `account_id` e só tratar como ambíguo se houver contas diferentes.

## Envio, templates e broadcasts

A regra central: quem envia sempre recebe uma conexão. Para isso nasce um único ponto de resolução, em `src/lib/whatsapp/connection.ts`, e as consultas diretas a `whatsapp_config` desaparecem dos outros arquivos.

```ts
// Devolve a conexão que deve enviar, ou lança erro claro.
resolveConnection(db, accountId, { connectionId?, conversationId? })
// 1. connectionId explícito (validado: pertence à conta e não está arquivada)
// 2. senão, a conexão da conversa
// 3. senão, a conexão padrão da conta
// Devolve { id, phoneNumberId, wabaId, accessToken (decifrado), label }
```

| Ponto | Mudança |
| --- | --- |
| `send-message.ts` (linha 263) | Usa `conversation.connection_id`; o parâmetro vem da própria conversa, o chamador não escolhe |
| `loadAccountMetaCredentials` (flows e automações) | Vira `loadConnectionCredentials(db, accountId, connectionId)`; as funções `engineSendText` e `engineSendTemplate` já recebem `conversationId`, de onde sai a conexão |
| `broadcast-core.ts`, `broadcast-resume.ts` | Lêem `broadcasts.connection_id`; o resume continua o envio pelo mesmo número mesmo que o padrão mude no meio |
| `api/whatsapp/broadcast` | Recebe `connectionId` no corpo; ausente usa a padrão |
| `react`, `media/[mediaId]` | Resolvem a conexão pela conversa ou mensagem; mídia baixada precisa do token do número certo |
| `templates/submit`, `templates/[id]`, `templates/sync` | Operam por WABA: o submit grava `waba_id`; o sync percorre cada WABA distinto da conta e deduplica |
| `health` | Passa a verificar cada conexão e devolve uma lista |

Template no envio: ao mandar um template numa conversa, a busca filtra `waba_id = conexão.waba_id`. Se o template não existir no WABA daquele número, o erro diz isso ("template X não existe no WABA do número Y"), em vez de falhar na Meta com código genérico.

Broadcast: o filtro de audiência continua sobre contatos, mas contatos que já têm conversa com outro número da conta recebem pelo número escolhido no broadcast, abrindo uma conversa nova se for preciso. Isso é o comportamento esperado; vale avisar na tela de criação.

## Fluxos, automações e IA

Os três motores respondem pelo número da conversa que os disparou. Nenhum deles escolhe número por conta própria, o que mantém o código de envio simples.

- **Fluxos** (`lib/flows`): o contexto de execução ganha `connectionId`. A idempotência “uma execução ativa por contato”, que a `017` definiu como (account\_id, contact\_id), continua valendo por conta: um contato não roda dois fluxos ao mesmo tempo, mesmo em números diferentes.
- **Automações** (`lib/automations`): o gatilho “mensagem recebida” ganha um filtro opcional por número (padrão: todos). Sem o filtro, uma automação escrita para o número de vendas dispararia também no de suporte.
- **IA** (`lib/ai/auto-reply.ts`): provedor, modelo, chave, prompt da empresa (`ai_configs`, UNIQUE por conta) e base de conhecimento continuam por conta. O que passa a ser por número é o contexto: o papel do número e o jeito de agir e responder, mais o limite de respostas e o liga e desliga, em `ai_connection_profiles`.

Como o prompt é montado a cada resposta: primeiro o `system_prompt` de `ai_configs`, que é o contexto geral da empresa e vale para todos os números; depois o `system_prompt` do perfil do número, que define o papel e o jeito de agir (vendas qualifica e conduz até agendar; suporte resolve com paciência e nunca empurra venda). A base de conhecimento continua uma só por conta e é a mesma para todos os números: o que muda de um número para outro é como a IA fala e age, não o que ela sabe. Número sem perfil preenchido usa só o prompt da conta, igual a hoje.

O contador `ai_reply_count` e a função `claim_ai_reply_slot` já são por conversa, e a conversa agora é por número, então o limite de respostas automáticas passa a valer por número sem mudar essas peças. A única mudança em `lib/ai` é carregar o perfil pelo `connection_id` da conversa e anexá-lo ao prompt; `knowledge.ts` e `query.ts` não mudam.

O ponto de integração com o que você escolheu do DeskcommCRM: o `stage_classifier` (organizar leads) e as checagens antes de enviar rodam sobre a conversa, então herdam o número sem trabalho extra. A única regra nova para as checagens é validar que o template existe no WABA do número antes do envio, como na seção anterior.

## Interface

Três telas mudam: configurações, inbox e criação de broadcast. As demais não precisam saber de números.

| Tela | Hoje | Depois |
| --- | --- | --- |
| Configurações, WhatsApp (`settings/whatsapp-config.tsx`) | Um formulário com um status | Lista de conexões (apelido, número exibido, status, padrão, IA ligada) com botão “Adicionar número”; cada linha abre o formulário atual, que não muda por dentro |
| Visão geral (`settings-overview.tsx`) | Um cartão de WhatsApp | Cartão com contagem “2 de 3 conectados” |
| Inbox (`inbox/page.tsx`, linha 211 lê a config) | Lista única de conversas | Filtro por número no topo (padrão: todos); selo do número em cada conversa quando a conta tem mais de uma conexão |
| Resposta no inbox | Envia pela config da conta | Mostra “responder por: \[número da conversa\]”, sem seletor, porque trocar de número no meio da conversa seria enviar para quem não a iniciou |
| Nova conversa | Escolhe contato | Escolhe contato e número; se já existe conversa com aquele número, abre a existente |
| Broadcast | Sem campo de número | Campo “enviar por”, obrigatório, com o padrão pré-selecionado; lista os templates do WABA do número escolhido |

Cada conexão ganha uma aba “IA” na própria linha da lista: papel (vendas, suporte), contexto do número (como ele fala e age), limite de respostas e liga e desliga. A base de conhecimento continua numa tela só, igual para todos os números. A escolha do número padrão fica na lista, com a explicação “usado para contatos novos”.

Duas regras de interface evitam a confusão mais provável. Com uma única conexão, nada de seletor, filtro ou selo aparece: a conta se comporta como hoje. E a URL de callback do webhook aparece uma vez no topo da lista, porque é a mesma para todos os números; cada conexão guarda o próprio `verify_token`, e o handshake GET do webhook já percorre todas as linhas de `whatsapp_config` procurando o token que bate, então não precisa mudar.

Permissões: adicionar, arquivar e escolher o padrão ficam para admin ou acima, o mesmo nível que hoje edita a configuração. O filtro do inbox é liberado para qualquer agente.

## API v1 com connection\_id

A API pública envia por telefone (`POST /api/v1/messages` recebe `to`, não `conversation_id`), então é onde a escolha de número mais importa para integrações externas, como a captura por landing page. Compatibilidade primeiro: chamada sem o campo novo continua funcionando e usa o número padrão.

| Endpoint | Mudança |
| --- | --- |
| `POST /api/v1/messages` | Campo opcional `connection_id`; passa para `resolveConversationByPhone`, que acha ou cria a conversa daquele número. A resposta inclui `connection_id` |
| `POST /api/v1/broadcasts` | Campo opcional `connection_id`, gravado em `broadcasts.connection_id` |
| `GET /api/v1/conversations` | Filtro `?connection_id=`; cada conversa serializada traz `connection_id` |
| `GET /api/v1/connections` (novo) | Lista conexões da conta (id, label, número exibido, status, padrão), escopo novo `connections:read`. Nunca devolve token |
| Webhooks de saída (`dispatchWebhookEvent`) | O payload dos eventos de mensagem e conversa passa a incluir `connection_id` |

`connection_id` inválido, de outra conta ou arquivado devolve 422 com mensagem clara, nunca cai silenciosamente no padrão.

Um bug latente que a mudança expõe: `resolveAuditUserId` em `lib/api/v1/contacts.ts` busca `whatsapp_config` por conta com `.maybeSingle()`. Com duas linhas, `maybeSingle` devolve erro e o código cai no dono da conta, mudando a autoria das escritas da API sem aviso. A correção é filtrar por `is_default` ou ordenar por `created_at` e pegar a primeira.

O v0.8 não tem servidor MCP no código de `src` (a busca não achou nenhum), então não há o que adaptar ali; se você adicionar um depois, a ferramenta de enviar mensagem deve receber `connection_id` igual ao endpoint.

## Testes e riscos

O projeto já usa vitest, com testes ao lado das rotas (por exemplo `api/v1/messages/route.test.ts`). Os testes novos cobrem as regras que, se quebrarem, mandam mensagem pelo número errado.

| Teste | O que garante |
| --- | --- |
| `resolveConnection` | Ordem: id explícito, conversa, padrão; rejeita id de outra conta e conexão arquivada |
| Webhook com dois números na mesma conta | Mensagem para o número B cria conversa com `connection_id` B, não reaproveita a do A |
| Mesmo contato em dois números | Um contato, duas conversas, contagens de não lidas separadas |
| Resposta de fluxo e de IA | Sai pelo número da conversa, nunca pelo padrão |
| `template-webhook` com dois números no mesmo WABA | Dedupica por conta e não cai no caso “mais de uma linha” |
| Broadcast e resume | O envio retomado usa o número gravado no broadcast, mesmo com padrão trocado |
| `resolveAuditUserId` com duas linhas | Continua devolvendo o dono da conexão padrão |
| Migrations 060 a 062 | Rodam sobre um banco com dados da versão 059 e deixam todas as conversas com `connection_id` |

Testes de IA por número: dois perfis na mesma conta (vendas e suporte) geram prompts diferentes para o mesmo texto de entrada; a busca de conhecimento devolve os mesmos documentos nos dois; número sem perfil cai no prompt da conta. Testes do número padrão: contato novo pela API sem `connection_id` usa o padrão; arquivar o padrão exige escolher outro antes.

Riscos, do mais grave ao mais leve:

1. **Responder pelo número errado.** É o pior defeito possível, porque o cliente vê outra empresa. Mitigação: um único `resolveConnection`, e teste que falha se qualquer arquivo de `src` consultar `whatsapp_config` por conta com `.single()` ou `.maybeSingle()`.
2. **Migration 061 sem volta.** Depois que existir contato com duas conversas, o índice antigo não recria. Mitigação: a interface de segundo número só entra depois de um período em produção com o código novo e um número só.
3. **Limite da Meta.** A Meta limita quantos números um WABA aceita e exige verificação do negócio para mais; o valor muda, então confira na documentação da Meta antes de prometer um número ao cliente. O sistema não impõe limite próprio.
4. **Policies de RLS e funções SECURITY DEFINER** das migrations `019`, `028`, `029`, `030` e `039`, ainda não lidas. Se alguma assume uma linha por conta, falha em silêncio. Entram na verificação da fase 0.
5. **Next.js 16 com mudanças de API.** O `AGENTS.md` do repositório manda ler `node_modules/next/dist/docs/` antes de escrever código. Para rotas, ler o guia de route handlers e o de `after` (usado em `api/v1/messages`) antes da fase 2.
6. **Sem teste ao vivo aqui.** A sandbox não alcança a Meta, então a validação com dois números reais precisa ser feita no seu ambiente, com dois números de teste do mesmo app.

## Ordem de entrega em fases

A ordem faz o sistema continuar idêntico para quem tem um número só até a fase 3; o múltiplo só aparece para o usuário na fase 4.

| Fase | Entrega | Pronto quando |
| --- | --- | --- |
| 0. Preparação | Ler `019`, `028`, `029`, `030`, `039`; ler os guias de route handlers do Next 16; escrever o teste que veta `.single()` em `whatsapp_config` | Lista de verificação fechada e teste vermelho nos arquivos que ainda violam |
| 1. Banco | Migrations `060`, `061`, `062` com backfill | `typecheck` e testes passam; toda conversa tem `connection_id` em um banco com dados reais |
| 2. Resolução | `resolveConnection` e substituição nos 25 arquivos; `resolveConversationByPhone` com `connectionId` | Teste da fase 0 fica verde; comportamento igual ao atual com uma conexão |
| 3. Webhook e envio | Webhook por conexão; templates por WABA; broadcast com `connection_id`; fluxos e automações respondendo pelo número da conversa | Testes de dois números na mesma conta passam; deploy com um número só por um período de observação |
| 4. Interface | Lista de conexões, botão de adicionar, escolha do número padrão pelo admin, filtro e selo no inbox, campo “enviar por” no broadcast | Dois números de teste ativos ao mesmo tempo e conversando no ambiente real |
| 5. IA por número | Migration `063`; prompt em duas camadas (empresa e número); aba “IA” na lista de conexões | Dois números com papéis diferentes respondem de forma diferente à mesma pergunta, com o mesmo conhecimento |
| 6. API e extras | `connection_id` na API v1, `GET /api/v1/connections`, filtro por número nas automações | Contratos da API documentados e testados |

O plano não inclui, de propósito: canal não oficial (GOWA), mover conversa de um número para outro e unificar a visão do mesmo cliente entre números. Cada um é trabalho à parte e nenhum bloqueia este.
