# Plano técnico: licenças e painel superadmin

Nov 6, 2026 · @Sérgio Bismark

## Resumo e decisões

Toda empresa (conta) só funciona com uma licença vigente. A licença nasce imutável, gera um pagamento por período, e quem cria, suspende, revoga e confere custos é você, num painel superadmin. Este plano amplia o painel do dono que já aparece no plano de créditos de IA: é a mesma área `/admin`, com a mesma tabela `platform_admins`.

| Regra que você definiu | Como o plano a trata |
| --- | --- |
| Uma licença por empresa, obrigatória | No máximo uma licença vigente por conta, garantida pelo banco. Conta sem licença vigente não usa o sistema |
| Período definido na criação | Mensal ou anual, mais preço e limites, tudo gravado na criação |
| Licença não pode ser alterada | Gatilho no banco recusa mudar período, preço, datas e limites. Só mudam o estado (suspender, revogar) e o status dos pagamentos |
| Mudar de mensal para anual | Cria-se uma licença nova e o superadmin revoga a antiga, numa ação única "trocar licença" |
| Pagamento a cada período | Mensal gera um pagamento por mês; anual gera um por ano, no dia da contratação, pelo valor do ano todo |
| Data de início e de fim | Início obrigatório. Fim opcional: com fim, a licença expira sozinha, para de cobrar e bloqueia o acesso; sem fim, renova até ser revogada |
| Bloquear por falta de pagamento, uso ou acesso | Três motivos de suspensão: `payment`, `usage` e `manual`, cada um com a sua rotina |
| Ver custos | Por empresa: custo real de IA, armazenamento e receita paga, com a margem |

Como entendi o que não ficou explícito, para você corrigir:

- **Status do pagamento:** pendente (gerado, ainda no prazo), pago, atrasado e cancelado. Você listou também "ativado" e "desativado"; entendi que isso descreve a licença, não o pagamento, e por isso eles ficam nos estados da licença (ativa, suspensa).
- **Fim com período fechado (confirmado):** o fim precisa cair na fronteira de um período, como 6 meses exatos para um mensal. O formulário pede a quantidade de períodos e calcula a data, para nunca cobrar um mês pela metade.
- **Bloquear por uso:** entendi como limites contratados na própria licença (conexões de WhatsApp ou Instagram, usuários, armazenamento, créditos de IA por período). Passou do limite, a ação é barrada ou, no extremo, a conta é suspensa por você. Para aumentar no meio do contrato, entra um adicional (seção Limites flexíveis).
- **Bloquear por acesso:** entendi como suspensão manual feita por você, com motivo registrado, a qualquer momento.
- **Tolerância e remoção de dados:** passados os 5 dias de tolerância sem pagamento, a conta é suspensa e, 30 dias depois, uma rotina apaga todos os dados e a própria conta, mantendo só o registro financeiro (seção Encerramento e exclusão de contas).
- **Licença gratuita:** entendi como uma licença de preço zero, com ou sem data final. Não há tipo de licença à parte; o pagamento de valor zero nasce pago, então nunca atrasa nem é suspenso por falta de pagamento.
- **Mensagens recebidas:** entendi que o recebimento segue normal enquanto a licença está ativa, inclusive nos 5 dias de tolerância de um pagamento atrasado, e que depois disso mensagens novas deixam de ser gravadas. Licença expirada ou revogada não tem tolerância, então a trava vale desde o primeiro dia sem licença ativa.

Todos os pontos em aberto foram respondidos. Decidido por você: tolerância de 5 dias por licença; exclusão da conta 30 dias depois da suspensão por falta de pagamento; licença gratuita é uma licença de preço zero, sem tipo à parte; licença expirada ou revogada só desativa, com a exclusão automática desses casos numa chave de Configurações que vem desligada; mensagens recebidas gravadas até o fim da tolerância e travadas depois. Sobram só leituras da fase 0, na tabela de fases.

## Vida de uma licença

Uma licença só dá acesso no estado ativa. O estado muda por data (início e data final), por pagamento ou por ação sua.

&#91;embedded content: estados de uma licença · 5 estados, 5 transições\]

Agendada vira ativa no dia de início. Ativa vira expirada na data final, sozinha. Suspensa volta a ativa quando o motivo desaparece: a suspensão por pagamento some quando o pagamento é confirmado, e a manual só você desfaz. Revogar encerra a licença a partir da data escolhida e é o que acontece com a antiga na troca de mensal para anual. A rotina diária desconecta as contas do Zernio de toda conta sem licença vigente, seja suspensa por pagamento, seja expirada ou revogada sem outra licença vigente ou agendada; a troca de mensal para anual não desconecta nada, porque a nova licença nasce na mesma transação.

## Modelo de dados

Três tabelas de base, com os adicionais e os trabalhos de exclusão em seções próprias: `licenses` (o contrato, imutável), `license_payments` (um pagamento por período) e `license_events` (histórico que só cresce). As regras de imutabilidade e de uma licença por conta ficam no banco, não só na tela, para que nenhum bug ou rota esquecida consiga burlá-las.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE licenses (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id            UUID REFERENCES accounts(id) ON DELETE SET NULL,  -- nulo depois que a conta é apagada
  account_ref           UUID NOT NULL,   -- cópia do id da conta, sem chave estrangeira; nunca muda
  account_name          TEXT NOT NULL,   -- nome da empresa na criação, para o histórico
  billing_period        TEXT NOT NULL CHECK (billing_period IN ('monthly','yearly')),
  price_cents           INTEGER NOT NULL CHECK (price_cents >= 0),  -- por período
  currency              TEXT NOT NULL DEFAULT 'BRL',
  starts_on             DATE NOT NULL,
  ends_on               DATE,    -- primeiro dia SEM acesso; nulo = renova até ser revogada
  grace_days            INTEGER NOT NULL DEFAULT 5 CHECK (grace_days BETWEEN 0 AND 30),
  -- limites contratados (nulo = sem limite)
  max_connections       INTEGER,
  max_members           INTEGER,
  storage_limit_bytes   BIGINT,
  ai_credits_per_period NUMERIC(18,6),
  ai_usd_per_credit     NUMERIC(12,8),
  origin                TEXT NOT NULL DEFAULT 'manual' CHECK (origin IN ('manual','migration')),
  replaces_license_id   UUID REFERENCES licenses(id),
  -- únicas colunas que mudam depois de criada
  suspended_at          TIMESTAMPTZ,
  suspended_reason      TEXT CHECK (suspended_reason IN ('payment','usage','manual')),
  suspended_note        TEXT,
  revoked_on            DATE,
  revoked_note          TEXT,
  created_by            UUID REFERENCES auth.users(id),
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (ends_on IS NULL OR ends_on > starts_on),
  CHECK ((suspended_at IS NULL) = (suspended_reason IS NULL))
);

-- no máximo uma licença vigente por conta: os intervalos [início, fim) não se sobrepõem
ALTER TABLE licenses ADD CONSTRAINT licenses_one_live_per_account
  EXCLUDE USING gist (
    account_id WITH =,
    daterange(starts_on, COALESCE(LEAST(ends_on, revoked_on), 'infinity'::date), '[)') WITH &&
  );

-- imutabilidade: só suspensão e revogação podem mudar; nada é apagado
CREATE FUNCTION licenses_guard() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'licença não pode ser apagada';
  END IF;
  IF (COALESCE(NEW.account_id, OLD.account_id), NEW.account_ref, NEW.account_name,
      NEW.billing_period, NEW.price_cents, NEW.currency, NEW.starts_on,
      NEW.ends_on, NEW.grace_days, NEW.max_connections, NEW.max_members,
      NEW.storage_limit_bytes, NEW.ai_credits_per_period, NEW.ai_usd_per_credit,
      NEW.origin, NEW.replaces_license_id)
     IS DISTINCT FROM
     (OLD.account_id, OLD.account_ref, OLD.account_name,
      OLD.billing_period, OLD.price_cents, OLD.currency, OLD.starts_on,
      OLD.ends_on, OLD.grace_days, OLD.max_connections, OLD.max_members,
      OLD.storage_limit_bytes, OLD.ai_credits_per_period, OLD.ai_usd_per_credit,
      OLD.origin, OLD.replaces_license_id) THEN
    RAISE EXCEPTION 'licença é imutável: crie outra e revogue esta';
  END IF;
  IF OLD.revoked_on IS NOT NULL AND NEW.revoked_on IS DISTINCT FROM OLD.revoked_on THEN
    RAISE EXCEPTION 'licença revogada não muda';
  END IF;
  RETURN NEW;
END; $$ LANGUAGE plpgsql;
CREATE TRIGGER licenses_guard BEFORE UPDATE OR DELETE ON licenses
  FOR EACH ROW EXECUTE FUNCTION licenses_guard();

CREATE TABLE license_payments (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  license_id     UUID NOT NULL REFERENCES licenses(id) ON DELETE RESTRICT,
  period_index   INTEGER NOT NULL CHECK (period_index >= 0),
  period_start   DATE NOT NULL,
  period_end     DATE NOT NULL,          -- exclusivo
  due_on         DATE NOT NULL,
  amount_cents   INTEGER NOT NULL CHECK (amount_cents >= 0),
  status         TEXT NOT NULL DEFAULT 'pending'
                   CHECK (status IN ('pending','paid','overdue','cancelled')),
  paid_at        TIMESTAMPTZ,
  paid_method    TEXT,                   -- pix, boleto, cartão, dinheiro, gratuito
  receipt_note   TEXT,
  marked_paid_by UUID REFERENCES auth.users(id),
  provider       TEXT,                   -- gancho para gateway no futuro
  external_id    TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (license_id, period_index),
  CHECK ((status = 'paid') = (paid_at IS NOT NULL))
);

CREATE TABLE license_events (
  id         BIGSERIAL PRIMARY KEY,
  license_id UUID NOT NULL REFERENCES licenses(id) ON DELETE RESTRICT,
  payment_id UUID REFERENCES license_payments(id),
  kind       TEXT NOT NULL,   -- created, payment_generated, payment_paid, payment_overdue,
                              -- payment_cancelled, suspended, reactivated, revoked, expired,
                              -- limit_blocked, addon_created, addon_revoked, addon_suspended, purge_scheduled,
                              -- purge_cancelled, purge_started, purge_done, purge_failed, account_purged
  actor_id   UUID REFERENCES auth.users(id),   -- nulo = rotina automática
  data       JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- license_events: gatilho que recusa UPDATE e DELETE, como em licenses
```

Pontos de projeto:

- **Datas, não horas.** Início, fim e vencimento são `DATE`, lidos no fuso `America/Fortaleza`. "Hoje" é `(now() AT TIME ZONE 'America/Fortaleza')::date`, em todas as rotinas e checagens, para a virada do dia não depender do servidor.
- **O fim é exclusivo.** `ends_on` é o primeiro dia sem acesso. Uma licença mensal de 6 meses que começa em 15/01 tem `ends_on` em 15/07.
- **Troca de licença em uma transação.** A função `swap_license(velha, nova)` primeiro revoga a antiga com `revoked_on` igual à data de início da nova, o que encolhe o intervalo dela, e só então insere a nova, sem sobrepor e sem lacuna. A nova guarda `replaces_license_id` para rastreio.
- **Histórico financeiro não some, a conta pode.** `licenses.account_id` é `ON DELETE SET NULL`: quando a conta é apagada (seção de exclusão), a licença, os pagamentos e os eventos ficam, ligados à empresa só por `account_ref` e `account_name`, copiados da conta na criação da licença. As chaves de `license_payments` e `license_events` para `licenses` continuam `RESTRICT`: licença não se apaga. O gatilho `licenses_guard` compara `COALESCE(NEW.account_id, OLD.account_id)`, o que aceita a passagem de `account_id` para nulo e só ela; sem isso o próprio `SET NULL` seria barrado como alteração.
- **Sem política de escrita.** RLS ligada em todas as tabelas deste plano; membros da conta só leem a própria licença, os pagamentos e os adicionais dela. Os trabalhos de exclusão não têm leitura para membros: a tela de bloqueio lê a data pelo servidor. Toda escrita passa por rotas do `/admin` e rotinas com o cliente de serviço, depois de checar `platform_admins`.
- **Limites na licença, imutáveis como o resto.** Aumentar conexões (WhatsApp ou Instagram), usuários ou espaço no meio do contrato é um adicional (seção Limites flexíveis), sem trocar de licença. Recarga avulsa de créditos de IA continua pelo ledger do plano de créditos, sem mexer na licença.

## Pagamentos

Cada licença gera um pagamento por período, pago adiantado, com vencimento no mesmo dia do início. Uma rotina diária cria os próximos, marca os atrasados, suspende por falta de pagamento e encerra as licenças que chegaram à data final.

| Período | Pagamentos | Vencimento | Valor |
| --- | --- | --- | --- |
| Mensal | Um por mês, de `period_index` 0 em diante | `starts_on` mais n meses, sempre contado a partir do início | `price_cents` |
| Anual | Um por ano, no dia da contratação | `starts_on` mais n anos | `price_cents`, o ano inteiro |

Contar sempre a partir do início evita deriva: uma licença mensal iniciada em 31/01 vence em 31/01, 28/02 e 31/03, e não em 28/02, 28/03 e 28/04.

Exemplos com a sua regra de data final: uma licença mensal com início em 15/01 e fim em 15/07 gera seis pagamentos, vencendo em 15/01, 15/02, 15/03, 15/04, 15/05 e 15/06; em 15/07 ela expira, sem sétimo pagamento e sem acesso. Uma licença anual com início em 10/03 gera um pagamento do ano inteiro em 10/03 e o próximo em 10/03 do ano seguinte, a menos que o fim seja essa data.

| Status do pagamento | Quando | Quem muda |
| --- | --- | --- |
| Pendente | Gerado, até o dia do vencimento | A rotina diária, ao gerar |
| Atrasado | Passou o vencimento e não foi pago | A rotina diária |
| Pago | Confirmado, com método, data e observação | O superadmin; um gateway, se um dia houver |
| Cancelado | A licença foi revogada ou expirou antes do período começar | A revogação, ou o superadmin com nota |

Pagamento de valor zero, de qualquer licença gratuita (a de migração e as que você criar com preço zero), nasce pago, com método "gratuito". Valor e datas do pagamento não mudam depois de criados, com o mesmo tipo de gatilho das licenças; só o status e os dados de confirmação mudam.

A rotina `license-daily` roda uma vez por dia, no fuso America/Fortaleza, e cada passo é seguro de repetir:

1. **Gerar:** para cada licença não revogada e não expirada, cria o próximo pagamento se o vencimento estiver a até 7 dias e o início do período for anterior ao fim da licença. A trava é `UNIQUE (license_id, period_index)` com `ON CONFLICT DO NOTHING`. O primeiro pagamento nasce junto com a licença.
2. **Atrasar:** pendente com vencimento anterior a hoje vira atrasado, com evento.
3. **Suspender:** licença com pagamento atrasado há mais de `grace_days` dias recebe suspensão por `payment`, com evento e aviso ao admin da conta e a você, agenda a exclusão da conta e desconecta as contas do Zernio, que cobra por conta conectada (seção Encerramento e exclusão de contas).
4. **Encerrar:** licença com `ends_on` igual ou anterior a hoje registra o evento `expired`; o estado já é calculado pela data, então o acesso cai no dia certo mesmo que a rotina atrase.
5. **Avisar:** pagamento que vence em 3 dias gera aviso para o admin da conta, pelo mecanismo de notificações existente.

A reativação não espera a rotina: ao marcar o pagamento como pago, se a suspensão era por `payment` e não resta nenhum pagamento atrasado, a licença volta a ativa na hora, com evento `reactivated`. Suspensão manual ou por uso não é desfeita por pagamento. As contas do Zernio já desconectadas não voltam sozinhas: o admin da empresa autoriza de novo pelo botão Reconectar em Conexões, e a mesma linha de conexão é reaproveitada.

A conferir na fase 0: qual agendador o projeto já usa para tarefas periódicas (o retomar de broadcast sugere que existe um); se não houver, é `pg_cron` do Supabase ou o agendador da hospedagem. Devolução proporcional na troca de anual para outro plano, ou estorno, ficam fora do sistema: registram-se como nota no pagamento.

## Onde a licença bloqueia

O bloqueio sai de uma única pergunta, "esta conta tem licença vigente hoje?", respondida por uma função SQL `account_license_state(account, hoje)` e por um helper `lib/license/guard.ts` com `getLicenseState` e `assertLicensed`. Todo ponto de entrada chama o helper; nenhum decide por conta própria. O resultado fica em cache de 60 segundos por instância, no mesmo molde de `lib/ai/account-status-cache.ts`, então uma suspensão pode levar até um minuto para valer em todas as instâncias.

| Estado calculado | Quando | O que o cliente vê |
| --- | --- | --- |
| `active` | Hoje está dentro do intervalo da licença e ela não está suspensa | O sistema normal; aviso de pagamento se houver atraso dentro da tolerância |
| `scheduled` | Só existe licença que começa no futuro | "Sua licença começa em dd/mm" |
| `suspended` | Licença vigente com `suspended_at` | Por pagamento: "Pagamento em atraso" com o valor e o vencimento. Por uso ou manual: "Acesso suspenso" com o contato |
| `expired` | Passou de `ends_on` | "Licença expirada em dd/mm" |
| `revoked` | Encerrada por revogação e sem outra vigente | "Licença encerrada" |
| `none` | A conta nunca teve licença, como uma conta nova | "Aguardando ativação da licença" |

Só `active` libera o uso. Os pontos de entrada, e o que cada um faz quando o estado não é `active`:

| Ponto | Onde | Comportamento |
| --- | --- | --- |
| Painel do cliente | `app/(dashboard)/layout.tsx`, no servidor | Tela cheia com o motivo, sem acesso às telas. A tela Minha licença continua aberta para o admin, para ele ver o pagamento em atraso |
| Webhook de entrada | `api/whatsapp/webhook/route.ts` | Sempre responde 200, para a Meta não reenviar nem desativar o webhook. Só grava a mensagem e executa fluxos, automações e IA com a licença ativa, o que inclui os dias de tolerância de um pagamento atrasado. Suspensa, expirada, revogada ou sem licença: confirma com 200 e descarta, sem gravar |
| Envio manual, reação, mídia e broadcast | `send-message.ts`, `api/whatsapp/*`, `broadcast-core.ts` | Recusa com erro 403 e código `license_inactive` |
| Broadcast agendado ou em andamento | `broadcast-resume.ts` | Pausa o envio, retoma se a licença voltar |
| API pública v1 | `requireApiKey` em `lib/auth/api-context.ts` | Um único ponto: qualquer chave da conta recebe 403 `license_inactive` |
| Fluxos e automações | `lib/flows/engine.ts`, `lib/automations/engine.ts` | Não iniciam execuções novas; as em andamento ficam pausadas e são descartadas se a licença expirar |
| IA | `lib/ai/auto-reply.ts`, rascunho, Playground | Não chama o modelo, então não gasta crédito |
| Webhooks de saída | `lib/webhooks/deliver.ts` | Não entrega eventos |

O `src/middleware.ts` do projeto não entra nessa conta: ele roda antes do banco e só cuida de autenticação. O Next 16 tem mudanças nessa área (o `AGENTS.md` manda ler `node_modules/next/dist/docs/` antes de mexer), então confirmar lá se o arquivo ainda tem esse nome antes de tocar nele, embora a licença não precise de middleware.

**Limites de uso.** Passar de um limite efetivo (o da licença mais os adicionais vigentes) barra só a ação que o ultrapassa, e nunca suspende a conta sozinho; você decide se vale uma suspensão por `usage` ao olhar o painel.

| Limite | Onde é conferido |
| --- | --- |
| `max_connections` | Ao conectar um número novo, no plano de vários números: recusa se a conta já tem o limite efetivo de conexões não arquivadas, de qualquer canal (licença mais adicionais) |
| `max_members` | Ao convidar ou aceitar convite, contra o limite efetivo (`lib/auth/invitations.ts`) |
| `storage_limit_bytes` | Vira o teto da conta no plano de armazenamento, somado aos adicionais de espaço; vale o menor entre esse total e o que o cliente definir |
| `ai_credits_per_period` | A cada pagamento marcado como pago, o sistema lança no ledger de créditos um grant desse valor, à taxa `ai_usd_per_credit` da licença. Recarga avulsa continua pelo painel |

Contas novas: o cadastro atual cria conta e dono sozinho (migration `017`); elas nascem sem licença, no estado `none`, e ficam na tela de espera até você criar a licença.

## Painel superadmin e tela Minha licença

O painel é a área `/admin`, a mesma do plano de créditos, aberta só a quem está em `platform_admins`. Cada rota chama `requirePlatformAdmin()` antes de usar o cliente de serviço e grava um evento com quem agiu, porque esse é o papel mais sensível do sistema.

| Tela | O que mostra | O que você faz |
| --- | --- | --- |
| Empresas | Todas as contas: estado da licença, vigência, próximo vencimento, atraso, saldo de créditos, custo de IA dos últimos 30 dias, espaço usado contra o limite | Filtra por estado, atrasadas, vencendo em 7 dias e perto do limite; abre o detalhe |
| Detalhe da empresa | Dados da licença (imutáveis), linha do tempo de pagamentos, eventos, uso contra limites, custos | Suspender, reativar, revogar, trocar licença, marcar pagamento como pago, cancelar pagamento, conceder créditos, adicionar ou revogar adicional, agendar ou cancelar exclusão |
| Nova licença | Formulário: empresa, mensal ou anual, preço, início, duração em períodos ou sem fim, tolerância, limites, créditos por período e taxa | Vê antes de confirmar a lista de pagamentos que serão gerados, com datas e valores, e a data final calculada |
| Pagamentos | Todos os pagamentos, atrasados primeiro | Filtra por status e vencimento; marca como pago |
| Custos e margem | Por empresa e por mês: receita dos pagamentos pagos, custo real de IA, armazenamento, margem | Exporta CSV |
| Auditoria | Todos os eventos de todas as licenças, com autor | Filtra por empresa, tipo e período |
| Configurações | Antecedência de geração (7 dias), aviso de vencimento (3 dias), tolerância padrão (5 dias), câmbio, modelos de IA permitidos | Edita |

As ações, com as regras de cada uma:

- **Suspender:** motivo (`payment`, `usage` ou `manual`) e nota obrigatórios; o acesso cai em até um minuto pelo cache. Reativar exige nota quando o motivo não foi pagamento.
- **Revogar:** data efetiva (hoje ou futura) e nota. Pagamentos pendentes de períodos que começariam depois dessa data são cancelados. Com data futura, o acesso segue até ela.
- **Trocar licença:** abre o formulário de nova licença com a empresa fixa e o início igual à data da troca. Ao confirmar, `swap_license` revoga a antiga nessa data e cria a nova numa transação só. O que já foi pago da antiga não é devolvido pelo sistema; fica a nota.
- **Marcar como pago:** método, data e observação. Se a suspensão era por pagamento e não resta atraso, a licença reativa na hora e, se a licença tem `ai_credits_per_period`, os créditos do período entram no ledger.
- **Cancelar pagamento:** só pendente ou atrasado, com nota.

Custos e margem: a receita vem de `license_payments` pagos, em reais. O custo de IA vem de `ai_usage_log.cost_usd`, em dólares, e o painel converte por um câmbio que você mantém em Configurações, sem cotação automática. O armazenamento aparece em GB, com um custo por GB opcional. O custo das conversas da Meta não aparece, porque cada cliente paga isso direto no próprio WABA, e a hospedagem compartilhada não é rateada por conta. Com o provedor Zernio, o painel passa a contar as contas conectadas por empresa e a aplicar a faixa de preço sobre o total do sistema, em dólares e pelo mesmo câmbio manual.

**Tela Minha licença**, para o admin de cada empresa, somente leitura: período, preço, vigência, próximo vencimento, histórico de pagamentos com status, uso contra os limites efetivos, com a composição licença mais adicionais (conexões, usuários, espaço, créditos), e a data da exclusão em destaque quando houver uma agendada e um texto de como pagar que você edita em Configurações, com um link ou contato. Sem gateway de pagamento no plano, o cliente paga fora e você marca como pago. Agentes comuns não veem preços.

## Limites flexíveis: adicionais

A resposta à sua dúvida: a licença continua fixa e imutável, e o que muda no meio do contrato entra como **adicional**. Um adicional é uma linha pequena, também imutável, presa à licença: "mais 2 conexões", "mais 3 usuários", "mais 20 GB". Ele tem preço, vigência e pagamentos próprios, e soma ao limite da licença enquanto estiver vigente.

Por que não deixar a licença editável: ela é o contrato e o registro do que o cliente pagou (preço, limites, datas). Se o limite pudesse mudar, o passado deixaria de bater com o que foi cobrado, e a sua regra de que mudança de plano é licença nova perderia o sentido. Por que não exigir licença nova a cada acréscimo: quem quer só mais um número teria que trocar de contrato, com revogação, novo ciclo de pagamento e conta de proporcional feita à mão.

| Situação | Como fica |
| --- | --- |
| Cliente quer mais conexões, usuários ou espaço no meio do contrato | Adicional na licença vigente; nada de trocar de licença |
| Cliente quer passar de mensal para anual, ou mudar o pacote-base | Licença nova e revogação da antiga (`swap_license`), como você definiu. O formulário oferece "levar os adicionais", que recria os adicionais na licença nova na mesma transação |
| Cliente quer mais créditos de IA | Recarga avulsa do plano de créditos. Não é adicional, porque crédito é saldo que se consome, não um teto |
| Cliente quer reduzir | Revoga-se o adicional com uma data, ou troca-se a licença |

O limite que vale passa a ser o efetivo: o limite da licença mais a soma dos adicionais vigentes do mesmo tipo. Licença com 2 conexões e um adicional de 2 conexões dá 4. O limite vale para WhatsApp e Instagram juntos, e o cliente divide como quiser: com 3 conexões, por exemplo, 2 de WhatsApp e 1 de Instagram. Se a licença não tem limite (nulo), segue sem limite e adicional não muda nada. Um adicional está vigente quando já começou, não terminou, não foi revogado nem suspenso, e a própria licença está vigente.

```sql
CREATE TABLE license_addons (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  license_id   UUID NOT NULL REFERENCES licenses(id) ON DELETE RESTRICT,
  kind         TEXT NOT NULL CHECK (kind IN ('connections','members','storage_bytes')),
  quantity     BIGINT NOT NULL CHECK (quantity > 0),   -- unidades a mais; bytes em storage_bytes
  price_cents  INTEGER NOT NULL CHECK (price_cents >= 0),  -- por período da licença
  starts_on    DATE NOT NULL,
  ends_on      DATE,           -- exclusivo; nulo = acompanha a licença
  note         TEXT,
  -- únicas colunas que mudam depois de criado
  suspended_at TIMESTAMPTZ,    -- pagamento do adicional atrasado além da tolerância
  revoked_on   DATE,
  revoked_note TEXT,
  created_by   UUID REFERENCES auth.users(id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (ends_on IS NULL OR ends_on > starts_on)
);
-- gatilho de imutabilidade igual ao de licenses: só suspended_at e revoked_* mudam; nada é apagado

-- o pagamento do adicional usa a numeração e as datas dos períodos da licença
ALTER TABLE license_payments
  ADD COLUMN addon_id UUID REFERENCES license_addons(id) ON DELETE RESTRICT;
ALTER TABLE license_payments DROP CONSTRAINT license_payments_license_id_period_index_key;
CREATE UNIQUE INDEX license_payments_one_per_period
  ON license_payments (license_id, addon_id, period_index) NULLS NOT DISTINCT;
```

`NULLS NOT DISTINCT` pede Postgres 15 ou mais; a fase 0 confere a versão do projeto Supabase. A view `account_license_state` ganha as colunas `eff_max_connections`, `eff_max_members` e `eff_storage_limit_bytes`, e `lib/license/guard.ts` lê os limites efetivos dali, no mesmo cache de 60 segundos. Os pontos que conferem limite não mudam de lugar, só passam a ler o valor efetivo.

### Cobrança

- **Pagamentos.** O adicional gera pagamentos pela mesma rotina diária, com a mesma numeração e as mesmas datas dos períodos da licença, para o cliente ter um vencimento só. Se o adicional começa no meio de um período, o primeiro pagamento é proporcional (dias restantes dividido pelos dias do período, vezes o preço) e vence no dia em que o adicional começa; o valor é sugerido pelo formulário e você pode ajustá-lo antes de confirmar.
- **Atraso.** Pagamento de adicional atrasado além da tolerância suspende só o adicional (`suspended_at`): o limite volta ao da licença e a conta segue funcionando. Pagar reativa na hora. Nunca suspende a licença nem agenda exclusão.
- **Fim.** O adicional acaba na data final dele, na data final da licença ou na revogação dela, o que vier primeiro.
- **Limite que cai abaixo do uso.** Quando um adicional acaba, nada é apagado nem desconectado. Só ficam barradas as novas adições (número, convite, upload), e o painel marca a empresa como "acima do limite" para você falar com o cliente.

### No painel

- **Novo adicional,** no detalhe da empresa: tipo, quantidade (em GB para espaço), preço por período, início (padrão hoje) e duração em períodos ou até o fim da licença. A pré-visualização mostra o limite efetivo antes e depois e a lista de pagamentos, com o primeiro proporcional.
- **Revogar adicional:** data efetiva e nota; os pagamentos de períodos futuros são cancelados. **Suspender e reativar** manualmente também existem.
- **Minha licença** mostra o limite efetivo com a composição (licença mais adicionais) e um botão "Pedir mais", que abre o contato que você cadastrar em Configurações.
- **Custos e margem** soma a receita dos adicionais, em linha separada da licença-base.

Se você vende sempre as mesmas 3 ou 4 combinações, vale guardá-las como modelos que preenchem o formulário de licença. Fica para depois, porque não muda nenhuma regra.

## Encerramento e exclusão de contas

Decidido: passados os 5 dias de tolerância sem pagamento, a conta é suspensa e, logo depois, uma rotina apaga todos os dados dela e a própria conta. Só o registro financeiro fica (licença, pagamentos e eventos), sem dado de cliente final. Como a exclusão não tem volta, o plano a trata como a parte mais perigosa do sistema: avisos ao cliente, uma janela curta para cancelar e uma chave de ligação gradual.

Linha do tempo, com vencimento em 10/03, tolerância de 5 dias e intervalo de 30 dias até a exclusão:

| Data | O que acontece | O que o cliente vê |
| --- | --- | --- |
| 10/03 | Vencimento. Pagar até aqui evita tudo | Aviso de vencimento 3 dias antes, já previsto |
| 11/03 | O pagamento vira atrasado; o acesso e o recebimento de mensagens continuam | Aviso ao admin da conta: acesso até 15/03 e, sem pagamento, exclusão dos dados em 15/04 |
| 15/03 | Último dia de acesso, o quinto da tolerância | Aviso final: amanhã o acesso é suspenso |
| 16/03 | A rotina suspende a licença por `payment`, agenda a exclusão para 15/04, e o webhook para de gravar mensagens novas | Tela de bloqueio com a data da exclusão |
| 08/04 e 14/04 | Lembretes de 7 dias e de 1 dia antes da exclusão | Aviso ao admin da conta e a você |
| 15/04 | O executor apaga os dados e a conta | Nada: a conta não existe mais |

A tolerância continua definida por licença, na criação (`grace_days`, padrão 5). O intervalo entre a suspensão e a exclusão é o `purge_delay_days`, em Configurações, padrão 30. Nesses dias a conta fica bloqueada e os dados ficam guardados. As mensagens que chegam nesse período não são gravadas, e a Meta não as reenvia. Pagar a qualquer momento entre 16/03 e a execução reativa a licença, cancela a exclusão e volta a gravar mensagens. Com `purge_delay_days` em 0, a exclusão sai na mesma rotina da suspensão e não sobra janela para cancelar.

Quando uma exclusão é agendada:

- **Automática, por falta de pagamento:** só para suspensão por `payment`, feita pela rotina diária, `purge_delay_days` dias depois da suspensão. Suspensão por `usage` ou `manual` nunca apaga nada sozinha.
- **Licença expirada ou revogada:** só desativa a conta, como no seu exemplo dos 6 meses. A exclusão automática desses casos é uma chave em Configurações, `purge_on_expired`, desligada por padrão. Ligada, a rotina agenda a exclusão para `purge_delay_days` dias depois do fim da licença, e só se a conta não tem outra licença vigente ou agendada.
- **Manual:** no detalhe da empresa você pode agendar a exclusão de qualquer conta, mesmo com a chave desligada, por exemplo de um cliente que encerrou ou que pediu a remoção dos dados. A tela pede para digitar o nome da empresa e exige uma data a partir de amanhã.
- **Nunca:** conta com algum membro em `platform_admins` (a rotina recusa). Por falta de pagamento, também nunca licença de preço zero (o pagamento nasce pago e não atrasa) nem adicional atrasado (ele só perde o adicional).

### O executor

A exclusão não cabe numa função de banco, porque mexe em arquivos do Storage e na Meta. Por isso ela roda no app: uma rota interna protegida por segredo (`CRON_SECRET`), chamada pelo agendador. É a resposta ao seu "cron ou webhook": o agendador faz a chamada, a rota faz o trabalho. A rotina diária chama o executor no fim; uma segunda chamada a cada hora só chama o executor, para retomar o que ficou pela metade.

```sql
CREATE TABLE account_purge_jobs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  license_id    UUID NOT NULL REFERENCES licenses(id) ON DELETE RESTRICT,
  account_ref   UUID NOT NULL,          -- id da conta, sem chave estrangeira: sobrevive à exclusão
  account_name  TEXT NOT NULL,
  reason        TEXT NOT NULL CHECK (reason IN ('payment','expired','manual')),
  episode_at    TIMESTAMPTZ,            -- início do episódio que originou: a suspensão por pagamento ou o fim
                                        -- da licença; evita recriar o trabalho depois de cancelado
  scheduled_for DATE NOT NULL,
  status        TEXT NOT NULL DEFAULT 'scheduled'
                  CHECK (status IN ('scheduled','running','done','failed','cancelled')),
  attempts      INTEGER NOT NULL DEFAULT 0,
  dry_run       BOOLEAN NOT NULL DEFAULT false,
  steps         JSONB NOT NULL DEFAULT '{}',   -- por passo: ok, erro, contagens
  summary       JSONB,                         -- totais do que existia antes de apagar
  note          TEXT,
  created_by    UUID REFERENCES auth.users(id),   -- nulo = rotina automática
  cancelled_by  UUID REFERENCES auth.users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  started_at    TIMESTAMPTZ,
  finished_at   TIMESTAMPTZ
);
-- uma exclusão por episódio: cancelada uma vez, a rotina não recria
CREATE UNIQUE INDEX account_purge_one_per_episode
  ON account_purge_jobs (license_id, episode_at) WHERE episode_at IS NOT NULL;
-- nunca dois trabalhos ativos para a mesma conta
CREATE UNIQUE INDEX account_purge_one_active
  ON account_purge_jobs (account_ref) WHERE status IN ('scheduled','running');
```

Cada execução segue estes passos, em ordem, gravando o resultado de cada um em `steps`. Se parar no meio, a próxima chamada continua de onde parou:

1. **Conferir de novo.** O trabalho ainda está `scheduled`, a licença segue suspensa por `payment`, o pagamento continua sem pagar (ou, na exclusão por fim de licença, a conta segue sem licença vigente nem agendada) e ninguém da conta é `platform_admin`. Se algo mudou, cancela. Para virar `running`, usa `UPDATE ... WHERE status = 'scheduled' RETURNING`, assim duas execuções nunca pegam o mesmo trabalho.
2. **Guardar um resumo.** Contatos, conversas, mensagens, arquivos e bytes, usuários e custo de IA acumulado, em `summary` e no evento. É o único rastro do que existia.
3. **Conexões, por provedor.** Para cada linha de `whatsapp_config`, cancela o que ainda estiver ativo no provedor (`provider`): `meta` cancela a assinatura do app no WABA (`DELETE /{waba_id}/subscribed_apps`), para a Meta parar de enviar eventos; `zernio` já foi desconectado quando a licença saiu da carência, e aqui a rotina só confere e desconecta o que sobrou, para a cobrança por conta conectada parar de vez; `gowa`, quando existir, faz logout do dispositivo e apaga o dispositivo e os arquivos dele no servidor GOWA. Se falhar, registra e segue: os webhooks descartam evento de conexão sem configuração. Os provedores novos estão descritos no [plano de Zernio e Instagram](https://claude.ai/artifact/BTgzSCJsMC9C7zL5aMudYt).
4. **Anotar os usuários da conta.** A participação vem de `profiles.account_id`, e o cascade remove os perfis, então os ids precisam ser guardados antes.
5. **Arquivos.** Pela API do Storage, nunca por SQL em `storage.objects` (isso deixaria os arquivos soltos): lista e remove, em lotes, tudo em `chat-media/account-<id>/` e `flow-media/account-<id>/`, mais `avatars/<user_id>/` de cada usuário anotado.
6. **Banco.** A função `purge_account(conta)` apaga em lotes as tabelas grandes, mensagens primeiro, para não estourar tempo, e então faz `DELETE FROM accounts`. As 27 chaves para `accounts` hoje são `ON DELETE CASCADE`, e os planos novos seguem a mesma regra. Na mesma transação, a licença recebe `revoked_on` igual a hoje, os pagamentos futuros são cancelados e o evento `account_purged` é gravado.
7. **Usuários.** Apaga cada usuário anotado em `auth.users` pela API de admin do Supabase. Sem isso sobram logins sem conta.
8. **Fechar.** Marca `done` com `finished_at`. Falha vira `failed`, com nova tentativa na chamada seguinte, até 5 vezes; depois disso, aviso para você.

O executor trabalha em fatias de até uns 50 segundos por chamada, para caber no tempo limite de rotas, e retoma na chamada seguinte.

### O que é apagado e o que fica

| Situação | O que acontece |
| --- | --- |
| Apagado | Contatos, conversas, mensagens, templates, broadcasts, flows, base de conhecimento, configuração de IA, chaves de API, webhooks de saída, credenciais do WhatsApp, saldo, histórico e uso de créditos de IA, arquivos nos buckets, usuários e perfis |
| Fica | Licença, pagamentos e eventos, ligados à empresa por `account_ref` e `account_name`. Pagamento atrasado continua atrasado, como registro da dívida; só os pagamentos futuros são cancelados. Fica também o trabalho de exclusão com o resumo |
| Não alcança | Backups do Supabase, que seguem a retenção do plano de hospedagem; a fase 0 confere quantos dias são, e os termos do cliente devem dizer isso |

### Proteções

- **Janela de cancelamento.** Entre a suspensão e a execução, `purge_delay_days` dias.
- **Cancelamento automático.** Marcar o pagamento como pago (que reativa a licença), revogar a licença ou trocá-la cancela o trabalho `scheduled`, com evento. Depois de `running`, não há cancelamento.
- **Cancelamento manual.** Botão "Cancelar exclusão" com nota. Cancelada, não volta a ser agendada para a mesma suspensão.
- **Chave de ligação.** `PURGE_ENFORCEMENT=off|dry_run|on`, no mesmo molde do bloqueio. Começa em `dry_run`: o executor faz só os passos 1 e 2 e grava em `summary` quanto apagaria, sem tocar em nada. Só vai para `on` depois de você conferir um ou dois casos reais.
- **Avisos.** Tela de bloqueio e notificação existente para o admin da conta; e-mail também, se a fase 0 achar um envio de e-mail já configurado.
- **Sem exclusão pelo cliente.** `/api/account` continua sem `DELETE`; só você, pelo painel, ou a rotina apagam uma conta. Um pedido de exclusão de dados feito pelo cliente é uma exclusão manual.
- **Contrato.** Os termos do cliente devem dizer a regra dos 5 dias e a exclusão. Vale um aval jurídico, porque a empresa cliente é quem responde pelos dados dos contatos dela.

### No painel

- Empresas ganha a coluna "Exclusão" (data agendada) e o filtro "com exclusão agendada".
- O detalhe da empresa ganha o bloco "Exclusão agendada", com data, motivo e os botões Cancelar e Executar agora (este pede o nome da empresa digitado).
- Configurações ganha `purge_delay_days` (padrão 30) e `purge_on_expired` (padrão desligado), e a Auditoria mostra os eventos `purge_scheduled`, `purge_cancelled`, `purge_started`, `purge_done` e `purge_failed`.

## Migração, testes, riscos e fases

**Contas que já existem** não podem parar no dia em que a regra "sem licença, sem uso" entrar. A migration cria, para cada conta atual, uma licença de migração: mensal, preço zero, início na data da migration, sem data final, sem limites e com `origin = 'migration'`. Os pagamentos dela nascem pagos, pelo método "gratuito". Depois você troca cada cliente para a licença real com "Trocar licença", no seu ritmo. A numeração continua depois dos outros planos: `067` com as tabelas, gatilhos e a restrição de uma licença por conta (já com `account_ref`, `account_name` e o `SET NULL`), e `068` com as funções e a licença de migração. Os adicionais entram na `069`, e a exclusão de contas (tabela de trabalhos e a função `purge_account`) na `070`.

**Ligação gradual.** O bloqueio tem três modos na variável `LICENSE_ENFORCEMENT`: `off`, `log` (só registra o que bloquearia) e `on`. O modo `log` roda por uma semana antes de ligar de verdade, para achar qualquer cliente que seria bloqueado por engano.

| Teste | O que garante |
| --- | --- |
| Imutabilidade | Mudar preço, datas ou limites falha; suspender e revogar funcionam; apagar falha |
| Uma vigente por conta | Licença sobreposta falha; `swap_license` não deixa lacuna nem sobreposição |
| Vencimentos | Mensal iniciada em 31/01 vence em 31/01, 28/02 e 31/03; seis meses geram exatamente seis pagamentos e nenhum sétimo; anual gera um por ano |
| Fim alinhado | `ends_on` fora da fronteira de um período é recusado |
| Rotina diária | Rodar duas vezes não duplica pagamento nem evento; um relógio injetado permite simular meses |
| Atraso e suspensão | Suspende só depois de `grace_days`; pagar reativa na hora; suspensão manual não some com pagamento |
| Fuso | A virada do dia acontece à meia-noite de Fortaleza, não de UTC |
| Pontos de entrada | Cada linha da tabela de bloqueio recusa fora do estado ativo; o webhook responde 200 sempre e só grava mensagem com a licença ativa, tolerância incluída |
| Limites | Conexão ou membro além do limite efetivo é recusado |
| Acesso aos dados | Membro de outra conta não lê a licença; só o cliente de serviço escreve |
| Adicionais | O limite efetivo é licença mais adicionais vigentes; adicional vencido, revogado ou suspenso não conta; licença sem limite continua sem limite; adicional é imutável |
| Adicional atrasado | Suspende só o adicional, nunca a licença, e não agenda exclusão; pagar reativa; o primeiro pagamento proporcional vence no início do adicional |
| Troca com adicionais | `swap_license` com "levar os adicionais" recria os adicionais na licença nova, na mesma transação |
| Limite que cai | Quando um adicional acaba, nada é apagado nem desconectado; só novas adições são barradas |
| Cascade da exclusão | Depois de `purge_account`, nenhuma tabela com coluna `account_id` (achadas por `information_schema`, incluindo as dos outros planos) tem linha da conta; licença, pagamentos e eventos continuam |
| Guard e SET NULL | Apagar a conta passa o `account_id` da licença para nulo sem erro; trocar `account_id` por outro valor falha |
| Agendamento | Só suspensão por `payment` agenda; `usage` e `manual` não; exclusão cancelada não é recriada no mesmo episódio; licença expirada ou revogada só agenda com \`purge\_on\_expired\` ligada, e nunca se a conta tem outra licença vigente ou agendada |
| Cancelamento e corrida | Pagar, revogar ou trocar cancela o trabalho `scheduled`; pagar e executar ao mesmo tempo: só um vence, e a conta que pagou não é apagada |
| Executor | Retomar depois de falha no meio não repete nem pula passos; arquivos do Storage e usuários de `auth.users` somem; `dry_run` não apaga nada |
| Proteções | Conta com `platform_admin` é sempre recusada; licença de preço zero também, na exclusão por falta de pagamento |

Riscos, do mais grave ao mais leve:

- **Bloquear um cliente que pagou.** Por bug de data, fuso ou rotina parada. Mitigação: o modo `log` antes do `on`, a tolerância por licença, suspensão automática só com pagamento realmente atrasado, aviso 3 dias antes e botão de reativar no painel.
- **Apagar a conta de quem pagou.** O pior caso do plano: o cliente pagou fora do sistema, você demorou a marcar como pago, e a rotina apagou tudo. Não há volta. Mitigação: janela de cancelamento, `dry_run` antes de `on`, avisos com a data, nova checagem na hora de executar, a lista "com exclusão agendada" no topo do painel e 30 dias entre a suspensão e a exclusão, com a conta bloqueada e os dados guardados.
- **Exclusão pela metade.** Arquivos soltos, usuário sem conta ou conta que não apaga. Mitigação: passos gravados e retomáveis, teste do cascade varrendo `information_schema`, e `failed` avisa você.
- **Conta grande.** Um `DELETE` único pode estourar o tempo. Mitigação: apagar em lotes, mensagens primeiro, e o executor trabalhar em fatias.
- **Backups.** Os dados apagados seguem nos backups do Supabase pelo prazo de retenção. Mitigação: conferir o prazo na fase 0 e dizer isso nos termos do cliente.
- **Esquecer um ponto de entrada.** O cliente bloqueado continua usando por um caminho que ninguém lembrou. Mitigação: um teste que lista as rotas em `app/api` e exige a chamada do guard, no espírito do teste que veta `.single()` no plano de vários números.
- **Mensagens perdidas.** Depois dos 5 dias de tolerância, o webhook para de gravar mensagem nova em qualquer estado sem licença ativa, e a Meta não reenvia o que já recebeu resposta 200. Quem pagar durante os 30 dias volta a usar, mas não recupera o que chegou nesse intervalo. Mitigação: a tela de bloqueio e os avisos dizem isso com todas as letras.
- **Erro de digitação no preço ou nas datas.** Como a licença não muda, o conserto é revogar e criar outra. A pré-visualização de pagamentos antes de confirmar existe para pegar isso na hora.
- **Devolução proporcional e estorno** ficam fora do sistema; só há nota no pagamento.
- **Painel como ponto sensível.** Quem entra em `/admin` pode suspender e apagar qualquer empresa. Mitigação: lista curta em `platform_admins`, auditoria de toda ação e, se quiser, exigir login recente.
- **Nota fiscal.** O sistema registra a cobrança e não emite nota; isso continua fora dele.

| Fase | Entrega | Pronto quando |
| --- | --- | --- |
| 0. Decisões e leitura | Achar o agendador do projeto; ler os guias do Next 16; conferir Postgres 15 ou mais, a retenção de backup do Supabase, se há envio de e-mail pronto e as funções da `047` e da `019` que mexem em contas (o `DELETE FROM accounts` e o cascade não podem esbarrar nelas) | Leituras feitas e registradas no plano |
| 1. Banco | Migrations `067` e `068`: tabelas, gatilhos, restrição, funções, licença de migração | Testes de banco passam e toda conta existente tem licença de migração |
| 2. Guard em modo log | `lib/license/guard.ts` em todos os pontos de entrada, com `LICENSE_ENFORCEMENT=log` | Uma semana sem bloqueio indevido nos logs |
| 3. Painel, núcleo | `platform_admins` (compartilhado com o plano de créditos), `/admin` com Empresas, Nova licença com pré-visualização, detalhe com suspender, revogar, trocar e marcar pago, e auditoria | Você cria uma licença e vê os pagamentos gerados |
| 4. Rotina diária | Gerar, atrasar, suspender, encerrar, avisar | Simulação de seis meses de uma licença mensal fecha certa |
| 5. Bloqueio ligado | `LICENSE_ENFORCEMENT=on`, tela de bloqueio e Minha licença | Uma conta de teste suspensa fica barrada em todos os pontos da tabela |
| 6. Limites e custos | Limites de conexões, membros e espaço; créditos por pagamento; tela de Custos e margem | A margem de uma empresa de teste bate com a conta feita à mão |
| 7. Adicionais | Migration `069`, cobrança de adicionais na rotina diária, limites efetivos no guard, formulário e telas | Um adicional de teste sobe o limite, cobra o proporcional e cai no fim sem apagar nada |
| 8. Exclusão em dry\_run | Migration `070`, executor e rota interna, agendamento pela rotina, avisos, bloco de exclusão no painel, com `PURGE_ENFORCEMENT=dry_run` | Uma conta de teste suspensa gera o resumo sem apagar nada; pagar e cancelar desarmam o trabalho |
| 9. Exclusão ligada | `PURGE_ENFORCEMENT=on`, depois de conferir um ou dois casos em dry\_run | Uma conta de teste some por completo (banco, arquivos e usuários) e só o registro financeiro fica |

O plano de créditos e este compartilham o `/admin` e o `platform_admins`; quem sair primeiro cria essas peças, e o outro só as usa.
