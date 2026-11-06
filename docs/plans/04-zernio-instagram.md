# Plano técnico: Zernio e Instagram no wacrm, com o GOWA para depois

Nov 6, 2026 · @Sérgio Bismark

## Resumo e decisões

O wacrm passa a falar com os provedores por uma interface única e ganha o Instagram como segundo canal: o direct cai na mesma caixa de entrada do WhatsApp e o comentário pode virar direct automaticamente. A prioridade é o Zernio, o caminho oficial para WhatsApp e Instagram, somado à conexão direta com a Meta que já existe. O GOWA (WhatsApp não oficial) foi adiado por decisão de 06/11/2026: o desenho dele fica guardado neste documento, e a interface já nasce pronta para recebê-lo.

| Decisão | Escolha | Motivo |
| --- | --- | --- |
| Provedores | `meta` (já existe e continua) e `zernio` (novo, oficial). O `gowa` fica para depois. | O plano de vários números já reservou a coluna `provider` na conexão |
| Instagram | Só pelo Zernio | Evita montar e manter, só para o Instagram, a aprovação de app e as permissões na Meta |
| Comentário vira direct | Criado na tela de automações do wacrm e executado pelo comment-to-DM do Zernio | O Zernio já cuida da resposta privada, do prazo, de um direct por comentário e da trava de seguir |
| Direct, IA e fluxos | O mesmo motor do WhatsApp, em todos os canais | Mensagem do Instagram entra, responde e automatiza como qualquer outra |
| Chave do Zernio | Uma `ZERNIO_API_KEY` global e um perfil do Zernio por conexão | Mesmo molde da chave global de IA; evita o conflito de "um WhatsApp por perfil" |
| GOWA | Adiado, sem data. Quando voltar: desligado por padrão, liberado por empresa, com freios de envio | Pode custar o número do cliente |

**O Zernio é global e invisível para o cliente.** Existe uma conta única do Zernio, a sua, com a `ZERNIO_API_KEY` guardada só no servidor. O cliente nunca cria conta no Zernio nem vê o nome dele: ele clica em Conectar dentro do wacrm, autoriza o Instagram ou o WhatsApp na tela da própria Meta e volta para o sistema. O wacrm cria um perfil do Zernio por conexão, por baixo dos panos. A cobrança do Zernio cai na sua conta e entra no painel de custos e margem; o que a Meta cobra pelos templates do WhatsApp continua indo direto para o WABA do cliente.

Como entendi o que não ficou explícito, para você corrigir:

- **"API oficial pelo Zernio":** o Zernio é mais uma forma de conectar o WhatsApp oficial. A conexão direta com a Meta continua existindo e nada é removido, e você decidiu mantê-la.
- **"Puxar os chats do Instagram":** ao conectar, o sistema importa as conversas dos últimos 30 dias e passa a receber as novas por webhook. Mensagem importada não dispara IA, fluxo nem automação.
- **"Automações estilo ManyChat":** comentário com palavra-chave vira direct, resposta a story, e IA e fluxos nos directs. Disparo em massa para seguidores fica fora, porque a Meta não oferece: o beta de lista de DM foi pausado em 2024, segundo [a Spur](https://www.spurnow.com/en/blogs/instagram-dm-automation-rules).

## O que o Zernio entrega de verdade, conferido na documentação

O Zernio cobre o que o plano precisa (WhatsApp oficial, direct e comentário do Instagram, webhooks e automação de comentário), mas alguns endpoints de envio não apareceram nas páginas públicas que consegui abrir. Esses ficam marcados como "confirmar na fase 0", com uma conta de teste, antes de qualquer código depender deles. Tudo abaixo foi lido nas páginas do próprio Zernio em 06/11/2026.

| Capacidade | O que a documentação diz | Situação |
| --- | --- | --- |
| Conectar WhatsApp | `GET /v1/connect/whatsapp?profileId&redirect_url` devolve a URL de autorização e volta com o `accountId`. O número pode ser próprio (Embedded Signup ou token de System User da Meta) ou alugado pelo Zernio | Documentado |
| Um número por perfil | O número fica ativo em um perfil só. Conflitos voltam como `one_whatsapp_per_profile`, `whatsapp_number_pinned_to_profile` e `whatsapp_number_already_connected`; sem forma de pagamento no time volta 402 | Documentado |
| Enviar no WhatsApp | `POST /v1/inbox/conversations/{id}/messages` (`accountId`, `message`) e, para abrir conversa com template, `POST /v1/inbox/conversations`. Tipos: texto, template, imagem, vídeo, documento, áudio, interativo, localização e contato. Fora das 24 horas só template | Documentado; corpo exato de cada tipo: confirmar na fase 0 |
| Templates | O status chega no webhook `whatsapp.template.status_updated`. Os endpoints para criar e editar template não apareceram | Confirmar na fase 0 |
| Disparo em massa | `POST /v1/broadcasts`, com entrega acompanhada por `message.delivered` e `message.read` | Documentado |
| Direct do Instagram | A caixa de entrada unificada e os webhooks `message.*` e `conversation.started` cobrem o direct. Os endpoints de listar e enviar direct do Instagram não apareceram | Confirmar na fase 0 |
| Comentário vira direct | `POST /v1/comment-automations` (detalhado na seção de automações) e `GET /v1/comment-automations/{id}/logs` | Documentado |
| Webhooks | Rota única no wacrm, assinatura HMAC-SHA256 no cabeçalho `X-Zernio-Signature` sobre o corpo cru e `X-Zernio-Event-Id` para não processar duas vezes | Documentado |
| Limites do Instagram | O Zernio enfileira e repete quando a Meta limita o ritmo, e avisa por `account.rate_limited` | Documentado |

Eventos que o wacrm vai consumir: `message.received`, `sent`, `delivered`, `read`, `failed`, `edited`, `deleted`; `comment.received`; `conversation.started`; `reaction.received`; `referral.received`; `account.connected`, `disconnected`, `permissions_changed`; `whatsapp.template.status_updated`; `whatsapp.account.quality_updated`; `automation.action.applied` e `failed`. O `comment.received` não é enviado para o X, o que não afeta o plano.

Restrições que valem para o produto:

- **Instagram:** só conta Business ou Creator. Conta pessoal não conecta, e o cliente precisa saber disso antes de tentar.
- **WhatsApp em coexistência** (número usado ao mesmo tempo no app do celular): o ritmo cai para 20 mensagens por segundo e grupos e chamadas ficam desligados. Pelo fluxo `onboarding=api` a conexão pula a coexistência.
- **Número novo no WhatsApp:** começa em 250 contatos únicos por dia, regra da Meta, e o limite sobe com a qualidade.
- **Webhooks de entrada:** a página de caixa de entrada indica que o add-on de inbox é exigido para receber mensagens. Confirmar na conta antes de contar com ele no custo.

### Preço e o que entra no painel de margem

O Zernio cobra por conta conectada (uma conta de WhatsApp ou de Instagram conta uma) e por mensagem enviada, em faixas graduais. A página não diz a moeda de forma explícita, e eu assumi dólar; confirmar na conta.

| Item | Valor |
| --- | --- |
| Contas conectadas, por mês | 1 e 2 grátis; 3 a 10 a US$ 6; 11 a 100 a US$ 3; a partir da 101 a US$ 1 |
| Mensagens enviadas | 10.000 por mês grátis por time, depois US$ 1 a cada 10.000 |
| Mensagens e comentários recebidos | Grátis |
| Template do WhatsApp | A Meta cobra direto do WABA do cliente, sem acréscimo do Zernio |
| Mensagem de serviço do WhatsApp | A página diz grátis até 30/09/2026 e depois cobrada na tarifa de utilidade da Meta, com 1.000 grátis por número por mês. Como a data já passou, confirmar na conta quem cobra e como aparece |
| Número alugado pelo Zernio | A partir de cerca de US$ 3 por mês |

Como a chave do Zernio é uma só para o sistema todo, a faixa de contas é do seu time e não de cada empresa cliente. Conta de exemplo, assumindo faixa gradual por time: com 20 contas conectadas no total, as 2 primeiras são grátis, 8 contas (da 3 à 10) custam US$ 48 e 10 contas (da 11 à 20) custam US$ 30, ou seja US$ 78 por mês e cerca de US$ 3,90 por conta. O painel de custos e margem (plano de licenças) passa a contar as contas Zernio por empresa e aplicar a faixa sobre o total.

Fontes (lidas em 06/11/2026): [WhatsApp no Zernio](https://docs.zernio.com/platforms/whatsapp), [preços de mensagens](https://docs.zernio.com/pricing/messages), [tabela de preços](https://zernio.com/pricing.md), [caixa de entrada](https://zernio.com/social-media-inbox), [comentário para direct](https://zernio.com/comment-to-dm) e [adaptador chat-sdk do Zernio](https://chat-sdk.dev/adapters/vendor-official/zernio).

## Camada de provedores no wacrm

Todo código que hoje fala com a Meta passa a falar com uma interface de canal, e o Zernio entra como adaptador novo, e o GOWA poderá entrar depois pelo mesmo caminho, sem tocar em caixa de entrada, IA, fluxos ou automações. A volta funciona do mesmo jeito: cada webhook vira eventos normalizados que o motor atual já sabe processar. Hoje essa camada não existe: há pelo menos quatro caminhos de envio duplicados, e cerca de 20 arquivos em `src/app/api/whatsapp`, 13 em `src/lib/whatsapp` e 6 em `src/components/inbox` tocam a Meta direto.

&#91;embedded content: camada de provedores · 1 interface, 3 adaptadores\]

Cada envio sai pela interface e cada webhook volta por ela, então caixa de entrada, IA, fluxos e automações não mudam quando entra um provedor novo. O Instagram passa pelo adaptador do Zernio, e o direct e o comentário chegam como eventos da mesma interface. O adaptador tracejado, do GOWA, fica para depois.

### Primeiro passo: unificar sem mudar comportamento

Antes de qualquer provedor novo, os quatro envios viram um só e o webhook da Meta passa a produzir eventos normalizados, com testes de caracterização escritos antes e nenhuma mudança de comportamento. Os quatro caminhos de hoje:

- `src/lib/whatsapp/send-message.ts`: o núcleo, usado por `/api/whatsapp/send` e `/api/v1/messages`.
- `src/lib/automations/meta-send.ts`: envio das automações.
- `src/lib/flows/meta-send.ts`: envio dos fluxos, que a IA em `src/lib/ai/auto-reply.ts` também usa.
- `src/lib/whatsapp/broadcast-core.ts` e `/api/whatsapp/broadcast`: disparo em massa.

A ordem importa para o plano de vários números. O `resolveConnection` dele escolheria a conexão em cada um desses quatro caminhos separadamente; unificando antes, os dois planos mexem em um ponto só.

### A interface `ChannelProvider`

| Grupo | Métodos |
| --- | --- |
| Capacidades | `capabilities(conn)` diz o que a conexão aceita (matriz abaixo) |
| Envio | `sendText`, `sendMedia`, `sendTemplate`, `sendInteractive`, `sendReaction`, `sendTyping`, `markRead` |
| Mídia | `fetchMedia(ref)` baixa a mídia recebida para o espelhamento que o wacrm já faz |
| Entrada | `parseWebhook(req)` valida a assinatura e devolve `NormalizedEvent[]` |
| Ciclo de vida | `connect`, `status`, `disconnect` |
| Opcionais | `listTemplates`, `submitTemplate` (só WhatsApp oficial) |

Os tipos de `NormalizedEvent` são `message_in`, `message_echo` (mensagem enviada pelo celular ou por outro app), `status` (enviada, entregue, lida, falhou), `reaction`, `comment`, `connection_status` e `template_status`. O motor nunca pergunta "qual provedor?", pergunta "essa conexão pode?", por exemplo `caps.templates`. É isso que impede um `if (provider === 'gowa')` espalhado pelo código.

### Conexões: migração 071

A tabela `whatsapp_config` continua sendo a tabela de conexões, como o plano de vários números já definiu (com `provider`, `label`, `display_phone_number`, `is_default` e `archived_at`). Esta migração acrescenta o canal, o id externo e relaxa os dois campos que só a Meta usa. Hoje `phone_number_id` e `access_token` são `NOT NULL` (`001_initial_schema.sql`); o `CHECK` por provedor garante que uma conexão da Meta continue exigindo os dois.

```sql
-- 071_connections_providers.sql
ALTER TABLE whatsapp_config
  ADD COLUMN IF NOT EXISTS channel TEXT NOT NULL DEFAULT 'whatsapp'
    CHECK (channel IN ('whatsapp', 'instagram')),
  ADD COLUMN IF NOT EXISTS external_account_id TEXT,  -- accountId do Zernio ou device_id do GOWA
  ADD COLUMN IF NOT EXISTS zernio_profile_id TEXT,    -- um perfil do Zernio por conexão
  ADD COLUMN IF NOT EXISTS handle TEXT,               -- @usuario no Instagram
  ADD COLUMN IF NOT EXISTS provider_status TEXT NOT NULL DEFAULT 'connected'
    CHECK (provider_status IN ('pending', 'connected', 'disconnected', 'paused', 'error'));

ALTER TABLE whatsapp_config
  ALTER COLUMN phone_number_id DROP NOT NULL,
  ALTER COLUMN access_token DROP NOT NULL;

ALTER TABLE whatsapp_config
  ADD CONSTRAINT whatsapp_config_provider_check
    CHECK (provider IN ('meta', 'zernio', 'gowa')),
  ADD CONSTRAINT whatsapp_config_provider_fields_check CHECK (
    provider_status = 'pending'
    OR (provider = 'meta' AND phone_number_id IS NOT NULL AND access_token IS NOT NULL)
    OR (provider IN ('zernio', 'gowa') AND external_account_id IS NOT NULL)
  );

CREATE UNIQUE INDEX IF NOT EXISTS whatsapp_config_provider_external_uidx
  ON whatsapp_config (provider, external_account_id)
  WHERE external_account_id IS NOT NULL;
```

Três decisões dentro dessa migração:

- **Um perfil do Zernio por conexão:** o Zernio só deixa um número de WhatsApp ativo por perfil (`one_whatsapp_per_profile`). Criar um perfil para cada conexão evita o conflito quando uma empresa conecta mais de um número.
- **Sem segredo novo por conexão:** a `ZERNIO_API_KEY` é uma variável de ambiente global, no mesmo molde da chave global de IA, e o acesso ao GOWA seguirá o mesmo caminho quando ele voltar. Se a fase 0 mostrar que algum provedor exige token por conexão, ele usa o mesmo `encrypt()` e `decrypt()` do `access_token`.
- **`provider_status = 'pending'`:** a linha nasce antes de a autorização do Zernio voltar, e ganha o `external_account_id` no retorno. Se o plano 1 já tiver criado um `CHECK` em `provider`, este o substitui.

O valor `gowa` já fica permitido no `CHECK` de `provider`, para não exigir outra migração quando ele voltar; nenhum código o usa até lá.

### Identidade do contato: migração 072

Instagram não tem telefone, e o GOWA, quando voltar, às vezes entrega só um identificador interno (`@lid`) no lugar do número. `contacts.phone` continua `NOT NULL`, seguindo a migração 040, que já guarda contatos conhecidos só pelo BSUID com `phone = ''` e um índice único parcial. A identidade fica numa tabela à parte, assim um mesmo contato pode ter WhatsApp e Instagram.

```sql
-- 072_contact_identities.sql
CREATE TABLE IF NOT EXISTS contact_identities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  contact_id UUID NOT NULL REFERENCES contacts(id) ON DELETE CASCADE,
  channel TEXT NOT NULL CHECK (channel IN ('whatsapp', 'instagram')),
  provider TEXT NOT NULL,
  external_id TEXT NOT NULL,  -- IGSID no Instagram; @lid ou número no GOWA
  handle TEXT,                -- @usuario no Instagram
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (account_id, channel, external_id)
);

CREATE INDEX IF NOT EXISTS contact_identities_contact_idx
  ON contact_identities (contact_id);

ALTER TABLE contact_identities ENABLE ROW LEVEL SECURITY;

CREATE POLICY contact_identities_select ON contact_identities FOR SELECT
  USING (is_account_member(account_id));
CREATE POLICY contact_identities_write ON contact_identities FOR ALL
  USING (is_account_member(account_id, 'agent'))
  WITH CHECK (is_account_member(account_id, 'agent'));
```

O papel `'agent'` na política de escrita é o mesmo que a tabela `contacts` usa desde a migração 017. Para unir duas identidades do mesmo cliente (o mesmo número que primeiro chegou como `@lid` e depois como telefone), a regra de mesclagem da migração 022 (deduplicação de telefone) é estendida para mover as identidades junto com as conversas.

### Eventos recebidos e mensagens: migração 073

Os provedores reenviam webhooks, então cada evento recebido é registrado uma vez e ignorado nas repetições. No Zernio a chave é o cabeçalho `X-Zernio-Event-Id`; no GOWA, quando ele voltar, o id da mensagem junto com o tipo do evento (confirmar na fase 0 se o payload traz um id de evento próprio). A mesma migração abre espaço na linha do tempo para um comentário do Instagram aparecer ao lado do direct, como uma nota de evento. A lista atual de `messages.content_type` vem da migração 010 (`text`, `image`, `document`, `audio`, `video`, `location`, `template`, `interactive`), e ganha `event`.

```sql
-- 073_provider_events.sql
CREATE TABLE IF NOT EXISTS provider_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  provider TEXT NOT NULL,
  event_id TEXT NOT NULL,
  connection_id UUID REFERENCES whatsapp_config(id) ON DELETE SET NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (provider, event_id)
);

-- Sem política: só o webhook, com a chave de serviço, lê e escreve.
ALTER TABLE provider_events ENABLE ROW LEVEL SECURITY;

ALTER TABLE messages
  DROP CONSTRAINT IF EXISTS messages_content_type_check;

ALTER TABLE messages
  ADD CONSTRAINT messages_content_type_check
  CHECK (content_type IN (
    'text', 'image', 'document', 'audio', 'video',
    'location', 'template', 'interactive', 'event'
  ));
```

Um job diário apaga os registros de `provider_events` com mais de 14 dias (prazo a ajustar), porque a repetição de um webhook acontece em minutos ou horas, não em semanas. O id do provedor da mensagem continua em `messages.message_id`, que já serve de chave para status e reação.

### O que cada conexão consegue fazer

A matriz abaixo é o que `capabilities(conn)` devolve. A interface do wacrm lê essa matriz para esconder o que a conexão não faz (botão de template em conversa de Instagram, por exemplo), e o construtor de fluxos avisa quando um fluxo usa um recurso que a conexão escolhida não tem. Onde está "Confirmar", a documentação que consegui abrir não responde e a fase 0 decide.

| Recurso | Meta direto | Zernio WhatsApp | Zernio Instagram | GOWA |
| --- | --- | --- | --- | --- |
| Texto | Sim | Sim | Sim | Sim |
| Mídia | Sim | Sim | Sim; tipos de arquivo aceitos: confirmar | Sim; a recebida precisa ser baixada do GOWA e espelhada |
| Template aprovado | Sim | Sim | Não | Não |
| Regra de janela | 24 horas; fora dela só template | 24 horas; fora dela só template | 24 horas; 7 dias com a tag Human Agent, só para atendente humano | Sem regra da Meta, mas com risco de bloqueio do número |
| Botões | Sim | Sim (interativo) | Sim | Não confiável: vira menu numerado em texto |
| Lista | Sim | Sim (interativo) | Não | Não: vira menu numerado em texto |
| Reação | Sim | Confirmar | Sim | Receber sim; enviar: confirmar |
| Indicador de digitando | Sim | Confirmar | Sim | Confirmar |

A coluna do GOWA é referência para quando ele voltar; nada dela é construído agora.

A regra da tag Human Agent vem de [uma página da Spur](https://www.spurnow.com/en/blogs/instagram-dm-automation-rules), que é terceiro e não a Meta: sete dias para atendente humano, e robô não pode usar a tag. A consequência prática é que a IA e os fluxos respeitam as 24 horas no Instagram, e só uma pessoa respondendo pela caixa de entrada usa a extensão de sete dias, caso a fase 0 confirme que o Zernio expõe a tag.

### Janela e formatação no remetente único

Duas regras novas entram no envio unificado, para valerem de uma vez para a caixa de entrada, a IA, os fluxos, as automações, o disparo e a API:

- **Janela de resposta:** no que li do código, as 24 horas são aplicadas no compositor da caixa de entrada (`message-composer.tsx`), usando `conversations.last_inbound_at`. O remetente unificado passa a validar também no servidor, pela regra de janela da conexão. Confirmar na fase 1 se o servidor já valida hoje.
- **Formatação por canal:** o texto do WhatsApp usa `*negrito*` e `_itálico_`, e no Instagram esses sinais aparecem literais. O caminho de envio da IA hoje não tem nenhuma limpeza de formatação, então uma função `formatForChannel(texto, canal)` entra no remetente único e tira os sinais para o Instagram.

## GOWA (adiado): WhatsApp não oficial, isolado e com freios

**Adiado por decisão de 06/11/2026.** O GOWA fica para outro momento e o Zernio é a prioridade. Esta seção guarda o desenho e as pendências para quando ele voltar. Nada dela entra nas fases 0 a 6, e as variáveis `GOWA_BASE_URL`, `GOWA_BASIC_AUTH` e `GOWA_WEBHOOK_SECRET` só são criadas nesse momento.

O GOWA permite conectar um número de WhatsApp sem passar pela API oficial, ao custo de um risco real de bloqueio do número, e por isso entra desligado por padrão, isolado da rede pública e com limites de envio. Ele é o [go-whatsapp-web-multidevice](https://github.com/aldinokemal/go-whatsapp-web-multidevice), um servidor que você hospeda e que fala o protocolo do WhatsApp Web (multi-dispositivo), com API REST e webhooks. O WhatsApp pode tratar um cliente não oficial como violação dos termos, e a decisão de bloquear é dele; os freios abaixo reduzem o risco e não o eliminam.

### O que a documentação confirma

| Ponto | O que sabemos | Situação |
| --- | --- | --- |
| Dispositivos | CRUD em `/devices` (desde a v8). O cabeçalho `X-Device-Id` ou o parâmetro `device_id` escolhe o dispositivo, e cada um tem webhook em `/devices/:device_id/webhook` | Documentado |
| Autenticação | O wacrm chama o GOWA com Basic auth (`GOWA_BASE_URL`, `GOWA_BASIC_AUTH`) | Documentado |
| Webhook | Payload com `event`, `device_id` e `payload`, assinado com HMAC-SHA256 em `X-Hub-Signature-256`, usando `WHATSAPP_WEBHOOK_SECRET`. O segredo é um por processo do GOWA | Documentado |
| Eventos | `message`, `message.ack`, `message.reaction`, `group.*`, `call.offer`, `presence`, `device.*`. Sinais de risco: `device.temporary_ban` e `device.reachout_timelock` | Documentado |
| Armazenamento | SQLite por padrão; Postgres pela variável `DB_URI`. A documentação cita a versão 9.7.0 | Documentado |
| Envio | Só `/send/image` apareceu nas páginas que abri. Texto, documento, áudio, vídeo e reação não foram vistos | Confirmar no openapi, fase 0 |
| Pareamento | O caminho do QR code (e do código de pareamento, se existir) não foi visto | Confirmar no openapi, fase 0 |

Fontes (lidas em 06/11/2026): [repositório do GOWA](https://github.com/aldinokemal/go-whatsapp-web-multidevice) e [visão geral da API](https://www.mintlify.com/aldinokemal/go-whatsapp-web-multidevice/api/overview). As páginas do openapi e do formato de webhook no GitHub não abriram nesta pesquisa, e a fase 0 as lê direto.

### Como o wacrm conversa com o GOWA

- **Rede:** o GOWA fica em rede privada, acessível só pelo wacrm. A interface web e o WebSocket dele nunca ficam expostos, porque o hub de WebSocket não isola um dispositivo do outro: quem conecta ali vê os eventos de todos.
- **Um dispositivo por conexão:** o `external_account_id` da conexão guarda o `device_id`. O wacrm cria o dispositivo pelo `/devices` e configura o webhook dele.
- **Pareamento:** a tela "Conectar WhatsApp não oficial" pede o QR ao GOWA e repassa a imagem ao navegador pelo backend do wacrm. Quando o evento `device.*` confirma o login, a conexão vira `connected`.
- **Webhook de entrada:** `POST /api/webhooks/gowa` valida a assinatura sobre o corpo cru, acha a conexão pelo `device_id`, registra o evento em `provider_events`, traduz para `NormalizedEvent` e responde 200 sempre, como o plano de licenças já definiu para o webhook.
- **Mídia recebida:** o GOWA guarda a mídia nele mesmo, então o adaptador baixa o arquivo e passa pelo espelhamento de mídia recebida que o wacrm já tem, para não depender do disco do GOWA.
- **Onde roda:** precisa de um host com processo ligado o tempo todo e disco persistente, não roda em função serverless. Usar Postgres no `DB_URI` (um banco separado do Supabase do wacrm) com backup; com o SQLite padrão, perder o disco desparelha todos os números e todo cliente precisa escanear o QR de novo.

### Freios contra bloqueio

| Risco | Freio no wacrm |
| --- | --- |
| Envio em rajada | Fila por conexão com intervalo aleatório (jitter) entre mensagens e teto por hora, reaproveitando o jitter das mensagens picadas do plano de créditos |
| Número recém-conectado | Aquecimento: o teto diário começa baixo e sobe por etapas; os valores são configuráveis pelo superadmin e calibrados na fase 7 com uso real |
| Mensagem fria | Disparo em massa pelo GOWA só para contatos que já têm conversa (`last_inbound_at` preenchido); nada de lista fria |
| Sinal de risco da conta | `device.temporary_ban` e `device.reachout_timelock` pausam a conexão na hora (`provider_status = 'paused'`) e avisam o admin da empresa e você |
| Expectativa do cliente | Rótulo "não oficial" na conexão e um aviso com aceite registrado na hora de conectar |
| Quem pode usar | Interruptor `allow_unofficial_providers` por empresa, desligado por padrão, no painel do superadmin junto dos limites da licença (decisão sua, abaixo) |

### Sem template e sem botão confiável

O GOWA não tem template aprovado e não entrega botão ou lista de forma confiável, então o remetente único transforma `sendInteractive` em um menu numerado em texto ("1. Orçamento", "2. Suporte"). Para a resposta continuar funcionando como clique, o wacrm guarda as opções na mensagem enviada e, se o cliente responder com o número logo em seguida, converte para o mesmo `interactive_reply_id` que a Meta devolveria. O construtor de fluxos mostra um aviso quando um fluxo com botão ou template está ligado a uma conexão GOWA. Grupos (`group.*`) e chamadas (`call.offer`) ficam fora desta versão e os eventos são ignorados.

## Instagram na caixa de entrada

O Instagram entra como mais uma conexão (canal `instagram`, provedor `zernio`), e o direct aparece na mesma lista de conversas do WhatsApp, com selo do canal, filtro por canal e a mesma ficha de contato. A IA, os fluxos e as automações de texto funcionam sem código novo, porque o direct chega como `message_in` e sai pelo remetente único; o que muda é a identidade do contato, a janela de resposta e o histórico.

### Como cada peça do wacrm trata o Instagram

| Peça | No Instagram |
| --- | --- |
| Conectar | Conta Business ou Creator, autorizada pelo fluxo do Zernio. Ao voltar, o `accountId` vai para `external_account_id` e o @ para `handle`. A tela avisa antes que conta pessoal não conecta. O endpoint exato de conexão do Instagram não apareceu na documentação: confirmar na fase 0 |
| Lista de conversas | Selo do canal e filtro por canal. A conexão de cada conversa já vem do plano de vários números (`conversations.connection_id`) |
| Contato | Quem escreve pela primeira vez vira contato com `phone = ''` e uma linha em `contact_identities` (canal `instagram`, `external_id` = id do Instagram, `handle`). A ficha mostra o @ e os canais do contato, e uma ação manual "unir com contato do WhatsApp" move as identidades e as conversas para um só contato |
| Compositor | Sem botão de template. Mostra quanto falta para a janela de 24 horas fechar e, fechada, bloqueia o envio com o motivo. Texto sem sinais de formatação do WhatsApp |
| IA | A mesma IA do WhatsApp, com perfil próprio por conexão (`ai_connection_profiles`, do plano de vários números), respeitando a janela e o repasse para humano (`ai_handoff_summary`) |
| Fluxos | Os gatilhos de hoje (`keyword`, `first_inbound_message`, `manual`) funcionam no direct. Botão funciona; lista vira menu numerado em texto, porque o Instagram não tem lista |
| Automações | `keyword_match`, `interactive_reply`, `tag_added`, `first_inbound_message` e `new_message_received` funcionam no direct. Os gatilhos novos de comentário e story ficam na próxima seção |
| Comentários | Entram como nota de evento (`content_type = 'event'`) na linha do tempo do contato, só quando o autor já tem conversa ou quando uma automação dispara. Comentário solto de quem nunca falou com você fica apenas no registro da automação, para a caixa não encher |
| Reações, edições e exclusões | Chegam pelos eventos `reaction.received`, `message.edited` e `message.deleted` |

### Janela de resposta

No Instagram vale a janela de 24 horas contada da última mensagem do cliente, e a resposta a um story também cai nessa regra. O wacrm já guarda `conversations.last_inbound_at`, que passa a alimentar a contagem da conexão. Existe uma extensão de sete dias com a tag Human Agent, mas só para atendente humano; robô não pode usar. Por isso a IA e os fluxos param nas 24 horas, e a extensão só vale para uma pessoa respondendo pela caixa de entrada, e só se a fase 0 confirmar que o Zernio permite enviar com essa tag. As regras de 24 horas e da tag Human Agent vêm de [uma página da Spur](https://www.spurnow.com/en/blogs/instagram-dm-automation-rules) (terceiro, lida em 06/11/2026), e vale conferir na documentação da Meta antes de lançar.

### Importar o histórico ao conectar

Para "puxar os chats do Instagram" para o sistema, a conexão nova dispara um job que importa as conversas dos últimos 30 dias (prazo a decidir), com cursor para retomar de onde parou e idempotência por `messages.message_id`, então rodar duas vezes não duplica nada. Três regras para não causar estrago:

- **Sem disparo:** cada evento carrega uma origem (`live` ou `history`), e IA, fluxos e automações ignoram `history`. Sem isso, conectar uma conta com 500 conversas antigas faria o robô responder a todas.
- **Datas reais:** `created_at` e `last_inbound_at` usam a data da mensagem original, não a do momento da importação.
- **Sem notificação:** mensagem importada não gera notificação nem não lida.

O endpoint de listar conversas e mensagens antigas do Instagram não apareceu na documentação que abri. Se o Zernio não o expuser, o histórico começa do dia da conexão e a tela diz isso.

### Teto de envio por conta

O guia de mercado fala em cerca de 200 mensagens por hora por conta do Instagram para envio automatizado (mesma página da Spur, terceiro). O Zernio já enfileira e repete quando a Meta limita, mas o wacrm também aplica um teto por hora na conexão, configurável, no mesmo remetente único que já controla o jitter das mensagens picadas. Assim um fluxo com muitos contatos não depende só da fila do Zernio.

## Automações estilo ManyChat

A tela de automações do wacrm continua única para WhatsApp e Instagram: o que acontece dentro de uma conversa (palavra-chave no direct, IA, fluxos) roda no motor atual, e o que nasce de um comentário ou de uma resposta a story é compilado para as automações de comentário do Zernio, que cuidam da resposta pública, do direct automático, do atraso e da trava de seguir. O wacrm guarda a automação, sincroniza com o Zernio e registra o resultado; quem usa não precisa saber que o Zernio existe.

### O que o ManyChat faz e onde cada coisa roda

| O que o cliente quer | Gatilho ou campo | Onde roda |
| --- | --- | --- |
| Comentou a palavra no post X, recebe direct | `comment_keyword` com post definido | Zernio (`POST /v1/comment-automations`) |
| Comentou a palavra em qualquer post da conta | `comment_keyword` sem post | Zernio |
| Responder o comentário em público | `commentReply`, com variações | Zernio |
| Respondeu a um story | `story_reply` | Zernio |
| "Siga para receber" | `followGate` (mensagem e botão) | Zernio |
| Só para seguidor, ou só para quem tem N seguidores | `audience` (`followerStatus`, `minFollowerCount`, `whenUnknown`) | Zernio |
| Palavra com erro de digitação, ou palavra exata | `matchMode` (`contains`, `word`, `exact`), `excludeKeywords`, `typoTolerance` (só no modo `word`) | Zernio |
| Atraso antes de responder | `dmDelaySeconds` e `commentReplyDelaySeconds` (o do comentário vai de 0 a 86.400 segundos) | Zernio |
| Botão de link no direct | `buttons` do tipo `url` | Zernio |
| Palavra-chave dentro do direct | `keyword_match`, que já existe | Motor do wacrm |
| IA respondendo o direct | IA atual, com perfil por conexão | Motor do wacrm |
| Marcar tag, criar negócio, avisar a equipe depois do direct | Etapas da automação atual | Motor do wacrm |

Todos os campos do Zernio acima estão na documentação de comment-to-DM lida em 06/11/2026 ([página do produto](https://zernio.com/comment-to-dm) e [guia](https://zernio.com/blog/comment-to-dm-automation)). O corpo exato de cada requisição fica para a fase 0.

### Como as duas metades se encontram

1. O cliente cria a automação no wacrm, escolhendo a conexão do Instagram, o post (por link ou id), as palavras, a resposta pública, a mensagem do direct, o atraso, o filtro de audiência e a trava de seguir. Os campos ficam em `trigger_config`, como as automações de hoje.
2. Ao ativar, o wacrm monta a requisição com o `profileId` e o `accountId` da conexão e cria a automação no Zernio. O id que volta fica em `automations.external_ref`. Pausar e apagar no wacrm repetem a ação no Zernio.
3. Quando alguém comenta, o Zernio faz a resposta pública e o direct. O wacrm recebe `automation.action.applied` (ou `automation.action.failed`) e grava uma linha em `automation_logs` (`success`, `partial` quando está esperando a pessoa seguir, `failed` com o motivo).
4. O direct enviado chega como `message.sent` e entra na linha do tempo do contato como mensagem de saída da automação, para o atendente ver o que foi dito. Se o id da mensagem já existe, o evento é ignorado.
5. Se a automação tem etapas depois do direct (marcar tag, criar negócio, avisar alguém), elas rodam no motor do wacrm quando o `applied` chega. Se a pessoa responde o direct, a resposta entra como `message_in` e a IA, o fluxo ou uma automação de palavra-chave assumem normalmente.

O campo `alsoMatchInDms` do Zernio fica sempre desligado. Palavra-chave dentro do direct é do motor do wacrm, e ligar nos dois lados faria a pessoa receber duas respostas.

### Migração 074

`automations.trigger_type` é texto livre, sem `CHECK` (migração 006), então os gatilhos `comment_keyword` e `story_reply` entram sem mexer em constraint. A migração acrescenta a conexão, a referência externa, o estado da sincronização e a regra de um post por automação ativa:

```sql
-- 074_instagram_automations.sql
ALTER TABLE automations
  ADD COLUMN IF NOT EXISTS connection_id UUID
    REFERENCES whatsapp_config(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS external_ref TEXT,           -- id da automação no Zernio
  ADD COLUMN IF NOT EXISTS external_sync_status TEXT NOT NULL DEFAULT 'none'
    CHECK (external_sync_status IN ('none', 'synced', 'pending', 'error')),
  ADD COLUMN IF NOT EXISTS external_sync_error TEXT;

CREATE UNIQUE INDEX IF NOT EXISTS automations_external_ref_uidx
  ON automations (external_ref) WHERE external_ref IS NOT NULL;

-- O Zernio aceita uma automação ativa por post; o wacrm barra antes de chegar lá.
CREATE UNIQUE INDEX IF NOT EXISTS automations_one_active_per_post_uidx
  ON automations (connection_id, (trigger_config->>'post_id'))
  WHERE is_active
    AND trigger_type = 'comment_keyword'
    AND trigger_config->>'post_id' IS NOT NULL;
```

### Regras que a tela precisa respeitar

- **Um post, uma automação ativa.** As automações da conta inteira podem rodar em paralelo, e a do post tem prioridade. A tela avisa quando uma segunda automação ativa aponta para o mesmo post, e o índice acima garante.
- **Um direct por pessoa por automação.** O Zernio não repete para quem já recebeu. A tela avisa isso, para ninguém achar que quebrou ao comentar duas vezes.
- **Uma resposta privada por comentário, em até sete dias.** É regra da Meta para o direct que nasce de comentário, e o atraso máximo de 24 horas do comentário cabe folgado (o limite do atraso do direct fica para a fase 0). Conferir na documentação da Meta antes de lançar, porque a página da Spur não a cita.
- **Botão só de link.** Os botões do direct automático são do tipo `url`; botão de resposta rápida não está na lista do Zernio.
- **Cada comentarista vira contato no Zernio também.** O dado fica nos dois sistemas, e a política de privacidade do produto precisa dizer isso.
- **Falha visível.** Se o Zernio recusar a criação (post inválido, conta desconectada), a automação fica com `external_sync_status = 'error'`, o motivo aparece na tela e há um botão para tentar de novo.
- **Os logs que o Zernio guarda ajudam a ajustar a regra.** `GET /v1/comment-automations/{id}/logs` também traz os comentários que não bateram (`misses`), e a tela pode mostrar "3 pessoas comentaram ‘quero’ e não receberam, olhe a palavra-chave".

### Em aberto para a fase 0

Os endpoints para editar e apagar uma automação de comentário não apareceram (só criar e ler os logs). Também falta confirmar se o `automation.action.applied` traz o identificador da pessoa no Instagram, que o wacrm usa para achar ou criar o contato, e se o Zernio tem um endpoint simples de resposta privada a um comentário. Se tiver, o wacrm pode executar tudo sozinho no futuro, mas hoje a automação pronta do Zernio já traz atraso, trava de seguir e filtro de audiência que seriam trabalho a mais.

## Impacto nos outros planos

O principal ajuste é que a conexão deixa de ser "um número da Meta" e passa a ser "um número ou uma conta, de qualquer provedor e canal", o que toca os três planos anteriores em pontos pequenos e precisos.

| Plano | O que muda |
| --- | --- |
| Vários números (migrações 060 a 063) | A coluna `provider` já existe e ganha os valores `zernio` e `gowa`, mais a coluna `channel`. A linha do plano que diz "só a API oficial da Meta agora; coluna provider deixa o GOWA para depois" continua valendo para o GOWA, e o Zernio passa a entrar por este plano. O `resolveConnection` é aplicado uma vez só, no remetente unificado (por isso a fase 1 vem antes). `ai_connection_profiles` vale também para cada conexão de Instagram |
| Créditos, picadas e armazenamento (064 a 066) | O jitter das mensagens picadas vira o teto por hora do Instagram (e, quando o GOWA voltar, o freio de envio dele). A mídia importada do histórico do Instagram conta no armazenamento da empresa. A IA no direct do Instagram consome créditos como no WhatsApp |
| Licenças e superadmin (067 a 070) | Quatro mudanças, na lista abaixo |

### O que muda no plano de licenças

- **Limite de conexões:** um único `max_connections` vale para todos os canais. Com 3 conexões disponíveis, o cliente divide entre Instagram e WhatsApp como quiser, por exemplo 2 de WhatsApp e 1 de Instagram. Aumentar o limite no meio do contrato continua sendo um adicional (`license_addons`).
- **Custos e margem:** o painel passa a contar as contas do Zernio por empresa e a aplicar a faixa gradual sobre o total do sistema, como calculado na seção do Zernio. O câmbio continua manual.
- **Interruptor do GOWA, para depois:** `allow_unofficial_providers` só entra no painel do superadmin quando o GOWA voltar ao escopo.
- **Webhook novo obedece à licença:** `/api/webhooks/zernio` aplica a mesma regra do webhook da Meta: grava mensagem só enquanto a licença está ativa (carência de 5 dias incluída), bloqueia depois e responde 200 sempre. A rota do GOWA seguirá a mesma regra quando existir.

### Encerramento e exclusão de contas, agora por provedor

O Zernio cobra por conta conectada, então a desconexão dele acontece mais cedo que a exclusão. Cada provedor é desconectado no seu momento:

| Provedor | Quando desconecta |
| --- | --- |
| `meta` | Na exclusão, 30 dias depois da suspensão: cancelar a inscrição do app no WABA do cliente, como no plano de licenças |
| `zernio` | No fim dos 5 dias de carência, quando a licença é suspensa por falta de pagamento: desconectar a conta no Zernio, o que interrompe a cobrança por conta conectada |
| `gowa` | Só quando o GOWA voltar: na exclusão, fazer logout do dispositivo e apagar o dispositivo e os arquivos dele no servidor GOWA |

Decidido: as contas do Zernio são desconectadas no fim dos 5 dias de carência, quando a licença é suspensa por falta de pagamento, e não esperam os 30 dias até a exclusão. Isso interrompe a cobrança por conta conectada (de US$ 6 a US$ 1 por conta, conforme a faixa) enquanto o cliente está bloqueado. Licença expirada ou revogada não tem carência no plano de licenças, então nesses casos, se a conta não tiver outra licença vigente ou agendada (a troca de mensal para anual não conta), a desconexão acontece no dia; se você preferir esperar 5 dias também aí, é uma linha no job.

O preço dessa escolha é o atrito de voltar. Se o cliente pagar depois, a licença reativa na hora, mas as conexões do Zernio ficam com `provider_status = 'disconnected'` e o admin precisa autorizar de novo, pelo botão Reconectar em Conexões. O fluxo de autorização leva o id da conexão desde o início, então a reconexão atualiza a mesma linha de `whatsapp_config` e nenhuma conversa ou contato se perde; se o Zernio devolver outro `accountId`, ele substitui o `external_account_id` (confirmar na fase 0). A conexão direta com a Meta não é desconectada nesse momento: ela só para de gravar pelo bloqueio do webhook e é cancelada na exclusão. Outro ponto da fase 0: o que acontece com um número alugado pelo Zernio quando a conta é desconectada. Se o número for perdido, o job desconecta só as contas de número próprio e as de Instagram.

## Testes, riscos e fases

A ordem das fases reduz o risco: primeiro se unifica o que já existe, com testes que provam que nada mudou, e depois entra o Zernio (oficial, para WhatsApp e Instagram). O GOWA, o único com risco de bloqueio de número, ficou fora desta rodada por decisão de 06/11/2026 e aparece como a última fase, adiada. Cada fase deixa o sistema funcionando e pode ser publicada sozinha.

### Fases

| Fase | Entrega | Pronta quando |
| --- | --- | --- |
| 0. Decisões e conferência | Respostas das pendências do Zernio (lista abaixo), com uma conta de teste | A lista está preenchida e este documento está atualizado com o que mudou |
| 1. Unificar envio e entrada | Remetente único no lugar dos quatro caminhos; o webhook da Meta passa a produzir `NormalizedEvent`; testes de caracterização | Os testes de antes passam sem alteração e as quatro rotas enviam igual |
| 2. Interface e adaptador `meta` | `ChannelProvider`, matriz de capacidades, migração 071, tela esconde o que a conexão não faz | A Meta funciona pela interface e a migração está aplicada |
| 3. Zernio no WhatsApp | Conexão por autorização, rota `/api/webhooks/zernio`, `provider_events`, envio, status e templates | Um número conectado pelo Zernio envia e recebe na caixa de entrada |
| 4. Instagram no direct | Migrações 072 e 073, identidades, selo de canal, janela de resposta, importação do histórico | O direct entra, responde dentro da janela, e o histórico não dispara robô |
| 5. Automações do Instagram e IA no direct | Migração 074, `comment_keyword`, `story_reply`, sincronização com o Zernio, logs, IA e fluxos no Instagram | Um comentário com a palavra-chave gera o direct, a resposta pública e a linha de log |
| 6. Painel e licenças | Contagem de contas do Zernio, limite único de conexões para todos os canais, desconexão do Zernio no fim da carência e exclusão pela Meta | Os custos aparecem no painel, o limite divide as conexões entre Instagram e WhatsApp, e as contas do Zernio desconectam no fim da carência em ambiente de teste |
| 7. GOWA com freios (adiada) | Fora desta rodada: o desenho (rota, pareamento por QR, fila com jitter, aquecimento, pausa por risco e interruptor) está na seção do GOWA | Quando a decisão de trazê-lo de volta for tomada; a fase 0 do GOWA vem antes |

Decidido: o Zernio é a prioridade e o GOWA fica para outro momento, sem data. Assim a interface de provedor é testada com um provedor oficial antes de receber o mais delicado.

### Fase 0: o que conferir com conta de teste

**Zernio**

- Corpo exato do envio de cada tipo de mensagem no WhatsApp e endpoints para criar e editar template.
- Como conectar o Instagram e como listar e enviar direct, incluindo se existe a tag Human Agent.
- Se existe resposta privada simples a um comentário, e como editar e apagar uma automação de comentário.
- Se `automation.action.applied` traz o identificador da pessoa no Instagram.
- Se há endpoint de histórico de conversas do Instagram.
- Se o add-on de inbox é exigido para receber webhooks, a moeda da cobrança e como ficou a mensagem de serviço do WhatsApp depois de 30/09/2026.
- O conflito `one_whatsapp_per_profile` na prática, e a seção `paths` do `openapi.yaml`, que não carregou por inteiro na pesquisa.

**GOWA (só quando ele voltar)**

- Pelo openapi: envio de cada tipo, reação e indicador de digitando, e o caminho do QR e do código de pareamento.
- Pelo payload de webhook: se existe id de evento próprio e como o `@lid` aparece.

O Instagram não passa por um app Meta próprio, então não há revisão de app da Meta para esperar.

### Testes

- **Antes da fase 1:** os testes que já existem (`send-message.test.ts`, `automations/meta-send.test.ts`, `broadcast-core.test.ts`) viram a rede de segurança. Faltam testes de caracterização para o envio dos fluxos e da IA, a confirmar na fase 1.
- **Contrato por provedor:** uma suite única roda os mesmos casos contra cada adaptador com HTTP simulado (cada tipo de mensagem, janela, capacidades, erro e limite de ritmo).
- **Webhooks:** assinatura válida e inválida, evento repetido (não grava duas vezes), conexão desconhecida (200 sem gravar), licença vencida (200 sem gravar), comentário, mídia.
- **Identidade:** o mesmo cliente chegando como `@lid` e depois como telefone vira um contato só, e um id do Instagram repetido não cria contato novo.
- **Histórico:** importar duas vezes não duplica, e nada da importação dispara IA, fluxo ou automação.
- **Comentário para direct:** criar, ativar, pausar e apagar sincronizam com o Zernio; uma segunda automação ativa no mesmo post é barrada; falha do Zernio deixa `external_sync_status = 'error'`.
- **GOWA, só quando voltar:** pausa em `device.temporary_ban`, teto por hora respeitado, menu numerado convertido de volta em resposta de botão.
- **À mão, com contas reais:** um Instagram Business de teste, um WhatsApp pelo Zernio com número de teste. O GOWA, quando voltar, só com um número descartável, nunca o de um cliente.

### Variáveis de ambiente novas

| Variável | Para que serve |
| --- | --- |
| `ZERNIO_API_KEY` | Chave global do Zernio, com leitura e escrita |
| `ZERNIO_WEBHOOK_SECRET` | Segredo da assinatura HMAC dos webhooks do Zernio |
| `ZERNIO_API_BASE_URL` | Opcional; padrão `https://zernio.com/api` |
| `GOWA_BASE_URL` | Só quando o GOWA voltar: endereço privado do servidor |
| `GOWA_BASIC_AUTH` | Só quando o GOWA voltar: usuário e senha do Basic auth |
| `GOWA_WEBHOOK_SECRET` | Só quando o GOWA voltar: mesmo valor que o `WHATSAPP_WEBHOOK_SECRET` configurado no processo do GOWA |

### Riscos

| Risco | Resposta |
| --- | --- |
| Depender de um fornecedor só para Instagram e WhatsApp oficial novo | A interface de provedor isola o Zernio; a conexão direta com a Meta continua funcionando; as mensagens ficam no banco do wacrm, não no Zernio |
| Zernio muda preço ou regra | O custo aparece no painel de margem por empresa, e o preço dos planos pode acompanhar |
| Bloqueio de número no GOWA (só quando ele voltar) | Interruptor por empresa, freios de envio, pausa automática, aviso com aceite e números descartáveis nos testes |
| Meta muda a janela ou a tag no Instagram | As regras ficam na matriz de capacidades, em um lugar só |
| Contato duplicado entre canais | `contact_identities` com chave única e ação manual de unir |
| Webhook perdido | Eventos repetidos são ignorados e os faltantes são recuperados pelos logs do Zernio (automações) e pela reimportação do histórico (mensagens), a confirmar na fase 0 |
| Dados de clientes em terceiros | Zernio guarda contatos de comentaristas. A política de privacidade do produto cita isso |
| Fazer as mudanças fora de ordem | Sem a fase 1, o plano de vários números e este tocariam os quatro envios separadamente, com o dobro de regressão possível |

### Decisões tomadas em 06/11/2026

1. **O GOWA fica para outro momento.** O Zernio é a prioridade. A seção do GOWA guarda o desenho, e a coluna `provider` e a interface de canal já deixam o caminho aberto.
2. **A conexão direta com a Meta continua**, porque já está criada. Segue como opção e como saída se o Zernio sair do ar.
3. **Um limite de conexões para todos os canais.** Com 3 conexões disponíveis, o cliente divide entre Instagram e WhatsApp como quiser, por exemplo 2 de WhatsApp e 1 de Instagram.
4. **O Zernio desconecta no fim dos 5 dias de carência**, e não nos 30 dias de suspensão. O cliente que pagar depois reconecta pelo botão Reconectar.

Duas suposições minhas, para você corrigir: licença expirada ou revogada desconecta o Zernio no dia, porque não tem carência no plano de licenças; e a conexão direta com a Meta fica conectada até a exclusão, apenas sem gravar mensagem depois da carência.
