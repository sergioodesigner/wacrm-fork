# Plano técnico: créditos de IA, mensagens picadas e armazenamento

Nov 6, 2026 · @Sérgio Bismark

## Resumo e decisões

São três mudanças independentes: créditos de IA (muda de onde vem a chave e como o uso é cobrado), mensagens picadas (muda como a resposta é enviada) e gestão de armazenamento (tela nova e limite por conta). Elas convivem com o plano de vários números: a picada pode ser ajustada por número, enquanto créditos e armazenamento são por conta.

| Decisão | Escolha | Motivo |
| --- | --- | --- |
| Chave de IA | Uma única chave do OpenRouter, da plataforma, em variável de ambiente; some da tela do cliente | O cliente não gerencia provedor nem chave, e você controla custo e modelos |
| Moeda | Créditos próprios, com saldo por conta | Você define o pacote sem expor o preço do provedor |
| Taxa | Por conta: quantos dólares reais do OpenRouter 1 crédito consome (`usd_per_credit`) | É o que você descreveu: N créditos para o cliente, X dólares de consumo real |
| Custo de cada chamada | Lido do campo `cost` que o OpenRouter devolve, sem tabela de preços própria | Preço real, inclusive cache e raciocínio, sem manter tabela desatualizável |
| Saldo zerado | A IA de resposta automática para e a conversa fica para um humano, com nota interna e aviso no painel | O cliente nunca consome além do contratado |
| Picadas | Divisão da resposta em partes curtas, com intervalo, sem nova chamada ao modelo | Mais natural e sem custo extra de tokens |
| Armazenamento | Medido por conta no bucket de mídia; tela Arquivos com limpeza e limite | Mostra só o que é seguro apagar; conversas e contatos nunca são tocados |

Dois pontos em aberto, com a sugestão do plano. Quem escolhe o modelo: o plano assume que só você escolhe os modelos permitidos. E embeddings da base de conhecimento pelo OpenRouter: precisa confirmar que há modelo disponível; se não houver, a busca por texto, que o código já usa quando não há chave de embeddings, continua valendo.

## Créditos de IA com chave global do OpenRouter

Cada conta tem um saldo em créditos e uma taxa em dólares por crédito; cada chamada de IA debita o custo real informado pelo OpenRouter dividido por essa taxa.

```latex
\text{créditos debitados} = \frac{\text{custo real da chamada em USD}}{\text{usd\_per\_credit da conta}}
```

Exemplo: um pacote de 1.000 créditos que deve consumir US$ 5,00 do OpenRouter tem `usd_per_credit` = 5 / 1.000 = 0,005. Uma resposta que custou US$ 0,0021 debita 0,42 crédito.

O OpenRouter devolve o custo em toda resposta, sem parâmetro extra, no campo `cost` do objeto `usage`, e também com streaming ([Usage Accounting](https://openrouter.ai/docs/guides/administration/usage-accounting)). Ele informa o valor na unidade de créditos dele, que corresponde a dólar; o plano manda conferir isso nas primeiras dez chamadas, comparando com o painel do OpenRouter, antes de cobrar alguém.

```sql
CREATE TABLE ai_credit_accounts (
  account_id            UUID PRIMARY KEY REFERENCES accounts(id) ON DELETE CASCADE,
  balance               NUMERIC(18,6) NOT NULL DEFAULT 0,
  usd_per_credit        NUMERIC(12,8) NOT NULL CHECK (usd_per_credit > 0),
  low_balance_alert     BOOLEAN NOT NULL DEFAULT true,
  low_balance_threshold NUMERIC(18,6) NOT NULL DEFAULT 100,  -- em créditos
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_credit_ledger (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id     UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  kind           TEXT NOT NULL CHECK (kind IN ('grant','usage','adjustment','refund')),
  credits        NUMERIC(18,6) NOT NULL,   -- positivo entra, negativo sai
  cost_usd       NUMERIC(14,8),            -- só em 'usage'
  usd_per_credit NUMERIC(12,8),            -- taxa vigente no momento
  usage_log_id   UUID REFERENCES ai_usage_log(id),
  note           TEXT,
  created_by     UUID REFERENCES auth.users(id),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE ai_usage_log
  ADD COLUMN cost_usd NUMERIC(14,8),
  ADD COLUMN credits_charged NUMERIC(18,6),
  ADD COLUMN openrouter_generation_id TEXT;
```

O débito roda numa função SQL `consume_ai_credits(account, cost_usd, usage_log_id)`, SECURITY DEFINER e liberada só para `service_role`, no mesmo molde de `claim_ai_reply_slot`. Ela subtrai do saldo e grava a linha do ledger na mesma transação, então duas respostas simultâneas nunca perdem um débito.

| Situação | Comportamento |
| --- | --- |
| Saldo acima de zero | A IA responde normalmente |
| Saldo zerado antes da chamada | O modelo não é chamado; a conversa ganha nota interna "IA pausada: sem créditos" e segue para humano (reaproveita `handoff.ts`); o admin recebe notificação |
| Chamada que estoura o saldo | A resposta sai, porque já foi paga, e o saldo fica levemente negativo; a próxima chamada é bloqueada |
| Saldo abaixo do limite de aviso | Notificação e pop-up no painel com link de recarga, uma vez por cruzamento, não a cada mensagem |
| Rascunho de IA e Playground | Também consomem créditos, pela mesma função |

O aviso de saldo é a tela que você mandou, trocando dólares por créditos: caixa "Avisar quando o saldo estiver acabando" e campo "Limite" em créditos, padrão 100.

Freio de emergência: a chave do OpenRouter da plataforma deve ser criada com limite de gasto (`limit`) e reinicialização (`limit_reset`, diário, semanal ou mensal), que a [API de chaves](https://openrouter.ai/docs/api/api-reference/api-keys/create-keys.md) permite. Se um bug gastar demais, o OpenRouter corta antes de o saldo das contas importar.

## Mudanças no código de IA

O código já aceita o provedor `openai_compatible` com `base_url`, que é o formato do OpenRouter, então não nasce provedor novo: a chave passa a vir do ambiente e o custo passa a ser lido da resposta.

| Ponto | Mudança |
| --- | --- |
| `ai_configs` | `api_key`, `base_url`, `provider` e `embeddings_api_key` deixam de ser usados; `api_key` vira anulável e nada é apagado. `model`, `system_prompt`, `is_active`, `auto_reply_enabled` e o limite por conversa continuam |
| `lib/ai/config.ts` | `loadAiConfig` não exige nem decifra chave da conta; devolve `openai_compatible` com a URL do OpenRouter e a chave de `OPENROUTER_API_KEY` |
| `lib/ai/types.ts` e `providers/openai-compatible.ts` | `AiUsage` ganha `costUsd` e `generationId`; o provedor lê `usage.cost` e o id da resposta. Hoje só os tokens são normalizados |
| `lib/ai/usage.ts` | `logAiUsage` grava `cost_usd` e chama `consume_ai_credits`; falha de débito é registrada em log de erro, nunca derruba a resposta |
| `lib/ai/credits.ts` (novo) | `hasCredits(db, accountId)` e `chargeUsage(...)`, usados por resposta automática, rascunho e Playground |
| `lib/ai/auto-reply.ts` | Antes de gerar, confere o saldo; sem crédito, segue o caminho de transferência para humano que já existe |
| `lib/ai/embeddings.ts` | Usa o OpenRouter se houver modelo de embeddings disponível; senão a base de conhecimento fica em busca por texto, que é o fallback atual |
| Tela de IA em Configurações | Saem provedor, chave e URL. Entram saldo, histórico de consumo, aviso de saldo e os controles de sempre (ativar, resposta automática, limite por conversa, picadas) |

Modelos: uma tabela de uma linha, `platform_ai_settings`, guarda o modelo padrão e a lista de modelos permitidos. A tela do cliente mostra o modelo padrão e, se você permitir, uma lista curta. A lista existe porque o custo por mensagem varia muito entre modelos, e o cliente não deve poder escolher um mais caro do que o pacote comporta.

Contas que já têm chave própria: a migration não as deixa mudas de repente. Cada conta recebe um saldo inicial de cortesia, configurável por você, e a chave antiga fica guardada, sem uso, até você decidir apagá-la.

## Mensagens picadas

A resposta da IA continua sendo gerada numa chamada só e cobrada uma vez; o que muda é o envio, que a divide em partes curtas com pausa entre elas, para não parecer um bloco de automação.

A configuração é a da tela que você mandou: caixa "Mensagens picadas (dividir resposta)" e "Intervalo entre mensagens (s)", padrão 2. Ela existe na conta e pode ser sobrescrita por número, no perfil da IA do plano de vários números: vendas mais picado e rápido, suporte com respostas mais longas.

```sql
ALTER TABLE ai_configs
  ADD COLUMN split_replies       BOOLEAN NOT NULL DEFAULT false,
  ADD COLUMN split_delay_seconds NUMERIC(3,1) NOT NULL DEFAULT 2
    CHECK (split_delay_seconds BETWEEN 0.5 AND 10);

-- por número; nulo = herda da conta
ALTER TABLE ai_connection_profiles
  ADD COLUMN split_replies       BOOLEAN,
  ADD COLUMN split_delay_seconds NUMERIC(3,1)
    CHECK (split_delay_seconds BETWEEN 0.5 AND 10);
```

Como a resposta é dividida, sem nova chamada ao modelo:

1. Corta por parágrafo (linha em branco). Parágrafo com mais de 220 caracteres é cortado por frase.
2. Une pedaços muito curtos (menos de 25 caracteres) ao vizinho, para não sair "Oi!" sozinho seguido de outra coisa.
3. Lista com marcadores e links ficam inteiros, na mesma parte.
4. Limite de 4 partes; o que sobrar entra na última.
5. Quando a picada está ligada, o prompt ganha uma linha pedindo parágrafos curtos, o que faz o corte cair em lugares naturais.

Como cada parte sai:

- A primeira sai logo. As seguintes esperam o intervalo, multiplicado por um fator entre 0,7 e 1,3 e levemente proporcional ao tamanho da parte. A variação evita o ritmo de metrônomo, que é o que denuncia automação.
- Se o cliente mandar outra mensagem no meio da sequência, as partes restantes não são enviadas e a nova mensagem gera uma nova resposta.
- Cada parte vira uma linha em `messages` marcada como gerada por IA, na ordem. Para o limite de respostas por conversa, a sequência inteira conta como uma resposta em `claim_ai_reply_slot`.
- O envio continua passando por `engineSendText`, o mesmo caminho de hoje, uma vez por parte.

Tempo: o webhook roda com `maxDuration` de 60 segundos. A soma das pausas fica limitada a 30 segundos; passando disso, as partes restantes saem juntas. O indicador "digitando" da Meta é ligado à mensagem recebida e dura pouco; falta confirmar na documentação se dá para repeti-lo entre as partes, e se não der, a pausa sozinha cria o ritmo.

## Gestão de armazenamento

A tela Arquivos mostra quanto espaço a conta usa, deixa filtrar e apagar mídia, e aplica um limite por conta, sem nunca tocar em conversas, contatos ou configurações.

O que o código já tem: o bucket `chat-media` guarda anexos em `account-<id>/...` com teto de 16 MB por arquivo (migration `023`), e o webhook espelha a mídia recebida nele, com opção de desligar em `whatsapp_config.mirror_inbound_media` (migration `039`), já que esse crescimento não tem limite. O que falta: `messages` não guarda caminho nem tamanho do arquivo, então hoje não dá para ligar um objeto do bucket à conversa nem para apagá-lo com segurança.

```sql
ALTER TABLE messages
  ADD COLUMN media_path       TEXT,
  ADD COLUMN media_size_bytes BIGINT,
  ADD COLUMN media_deleted_at TIMESTAMPTZ;

CREATE TABLE account_storage_settings (
  account_id       UUID PRIMARY KEY REFERENCES accounts(id) ON DELETE CASCADE,
  limit_bytes      BIGINT,                 -- nulo = sem limite
  warn_at_percent  INTEGER NOT NULL DEFAULT 80,
  auto_delete_received_after_days INTEGER  -- nulo = nunca apaga sozinho
);
```

O preenchimento do que já existe (backfill) extrai o caminho de `media_url` e lê o tamanho em `storage.objects`; o que não casar com nenhuma mensagem aparece como "arquivo sem conversa".

| Parte da tela | Como funciona |
| --- | --- |
| Cartões de espaço | Espaço usado no total, Recebidos, Enviados por você e Mídia de fluxos e templates (bucket `flow-media`), cada um com tamanho e número de arquivos |
| Dados do app | Não aparece na tela do cliente: o banco é compartilhado entre contas e não dá para medir por conta de forma justa. Fica só no painel do dono |
| Abas | "Recebidos" e "Enviados por você" |
| Filtros | Nome ou legenda, conversa (nome ou número), período (de, até, hoje, 7 dias, 30 dias), tipo de arquivo (foto, vídeo, áudio, documento), ordem (mais recentes, maiores) e "só arquivos sem conversa". Com vários números, entra um filtro por número |
| Apagar | Seleciona vários ou apaga tudo do filtro, com confirmação. Remove o objeto do bucket e marca `media_deleted_at`; a bolha na conversa passa a mostrar "arquivo apagado para liberar espaço" |
| Limite | "Definir um limite de espaço" abre `limit_bytes` e `warn_at_percent` |

Quando o espaço chega ao percentual de aviso, o aviso usa o mesmo mecanismo do saldo de créditos. Ao chegar a 100%, o sistema para de espelhar a mídia recebida (volta ao ponteiro para a Meta, que expira em cerca de 30 dias, como era antes da `039`) e recusa anexos novos com uma mensagem clara. Nada é apagado sozinho, a menos que você ligue a limpeza automática de recebidos antigos.

Permissão: só admin da conta apaga e define limite; agentes comuns não veem a tela.

## Painel do dono do sistema

Para você dizer quantos créditos cada cliente tem, o sistema precisa de uma área fora das contas. O wacrm não tem esse conceito (a busca por superadmin ou administrador da plataforma não achou nada), então ele nasce junto.

- **Quem entra:** tabela `platform_admins` com o id do usuário, preenchida por SQL na primeira vez. A rota `/admin` confere essa tabela antes de usar o cliente de serviço, e nenhuma política de RLS de conta é afrouxada.
- **Lista de contas:** nome, saldo em créditos, taxa, consumo dos últimos 30 dias em créditos e em dólares reais, espaço usado e limite, e se a IA está ativa.
- **Detalhe da conta:** conceder ou ajustar créditos, definir a taxa, ver o histórico do ledger e mudar o limite de armazenamento.
- **Modelos:** modelo padrão e lista de modelos permitidos (`platform_ai_settings`).
- **Visão geral:** créditos vendidos contra dólares consumidos no OpenRouter no mês, que mostra a sua margem real.

Ao conceder créditos você informa duas coisas, que são as que você descreveu: quantos créditos o cliente recebe e quantos dólares isso deve consumir de verdade. O sistema calcula a taxa.

Se a conta já tem saldo com outra taxa, o sistema não troca a taxa de tudo: ele calcula uma taxa média ponderada, para o valor real em dólares do saldo restante continuar correto.

```latex
\text{nova taxa} = \frac{\text{saldo atual} \times \text{taxa atual} + \text{créditos novos} \times \text{taxa nova}}{\text{saldo atual} + \text{créditos novos}}
```

Exemplo: saldo de 200 créditos a 0,005 (US$ 1,00) mais 1.000 créditos a 0,004 (US$ 4,00) dão 1.200 créditos a 5 / 1.200 = 0,004167. O total em dólares, US$ 5,00, se mantém.

Pagamento automático da recarga (Pix, cartão) fica de fora deste plano: o link de recarga do aviso de saldo leva a uma página ou contato que você define, e o crédito entra pela ação manual do painel.

Licenças, pagamentos por período e bloqueio por falta de pagamento usam este mesmo painel e a mesma tabela `platform_admins`; estão no plano técnico de licenças e painel superadmin. Lá, a licença pode trazer créditos de IA por período, lançados no ledger deste plano quando o pagamento é confirmado.

## Testes, riscos e ordem de entrega

Os testes cobrem o que, se errar, custa dinheiro ou envia algo estranho ao cliente. O projeto usa vitest, e `lib/ai` já tem testes ao lado de cada arquivo.

| Teste | O que garante |
| --- | --- |
| `consume_ai_credits` com chamadas simultâneas | Duas respostas ao mesmo tempo debitam as duas, sem perder nenhuma |
| Conversão de custo em créditos | O exemplo de 0,0021 dólar a 0,005 dá 0,42 crédito; taxa média ponderada dá 0,004167 |
| Saldo zerado | O modelo não é chamado, a conversa ganha a nota interna e vai para humano |
| Aviso de saldo | Dispara uma vez ao cruzar o limite, não a cada mensagem |
| Resposta sem `cost` | Registra erro e não some em silêncio; a reconciliação diária compara a soma de `cost_usd` com o gasto do OpenRouter |
| Divisão em picadas | Parágrafos, frases longas, união de pedaços curtos, link inteiro, máximo de 4 partes |
| Interrupção | Mensagem nova do cliente cancela as partes restantes |
| Picada por número | Perfil de vendas e de suporte com intervalos diferentes na mesma conta |
| Apagar arquivo | Remove o objeto, marca `media_deleted_at` e preserva mensagem, conversa e contato |
| Limite de espaço | Aos 100% deixa de espelhar mídia recebida e recusa anexo novo |

Riscos, do mais grave ao mais leve:

1. **Contas com chave própria ficarem mudas.** Mitigação: saldo de cortesia na migration e aviso antes de ligar a cobrança.
2. **Chave única como ponto único de falha e de abuso.** Um loop em uma conta pode gastar o saldo de todas. Mitigação: bloqueio por saldo em cada conta, limite de gasto na chave do OpenRouter e o teto de respostas por conversa que já existe.
3. **Unidade do custo.** O `cost` vem em créditos do OpenRouter; confirmar que equivale a dólar antes de cobrar. Sem isso, a taxa de todos os clientes sai errada.
4. **Dados dos clientes passando por uma conta só.** Mensagens de vários clientes vão pelo mesmo provedor. Vale dizer isso nos termos de uso e ler a política de dados do OpenRouter para a sua operação.
5. **Picadas e tempo de execução.** O limite de 30 segundos de pausas existe por causa dos 60 do webhook; se o intervalo ficar muito alto, as partes saem juntas.
6. **Apagar mídia é irreversível.** Confirmação obrigatória, e a limpeza automática começa desligada.

A numeração das migrations continua depois da `063` do plano de vários números. Créditos e armazenamento não dependem dele e podem sair antes; só a picada por número espera a tabela de perfis da `063`, e a picada por conta pode sair antes.

| Fase | Entrega | Pronto quando |
| --- | --- | --- |
| 0. Leitura | `generate.ts`, `providers/openai-compatible.ts`, o envio em `auto-reply.ts`; documentação da Meta sobre o indicador "digitando"; confirmar no OpenRouter a unidade do custo e os embeddings | Lista de dúvidas fechada |
| 1. Créditos, núcleo | Migration `064`; `credits.ts`; leitura de `cost`; `consume_ai_credits`; trava de saldo no `auto-reply.ts`; tela de IA sem chave | Dez chamadas reais debitam valores que batem com o painel do OpenRouter |
| 2. Painel do dono | `platform_admins`, `/admin`, conceder créditos com taxa ponderada, modelos permitidos | Você concede um pacote a uma conta e o saldo aparece para o cliente |
| 3. Aviso de saldo | Notificação e pop-up com link de recarga, com limite em créditos | O aviso aparece uma vez ao cruzar o limite |
| 4. Picadas | Migration `065`; divisão e envio com pausa; caixa e intervalo na tela; por número quando a `063` existir | Uma resposta longa sai em partes, e uma mensagem nova do cliente interrompe a sequência |
| 5. Armazenamento | Migration `066` com backfill; tela Arquivos; limite e aviso | Apagar libera espaço medido, e o limite barra o espelhamento |
