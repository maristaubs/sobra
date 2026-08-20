# Schema

Postgres, no Supabase. Este documento tem o desenho e o SQL real das migrations.

## A decisão que sustenta tudo

**Uma compra parcelada nunca é uma linha só.**

A compra fica em `entries`. Cada mês de vencimento vira uma linha em `installments`.
Uma compra de R$ 500,00 em 5x é uma `entry` e cinco `installments` de R$ 100,00,
espalhadas por cinco meses.

Isso existe porque a planilha antiga controlava parcela com um contador escrito à
mão ("restam 5", "restam 4"). Um contador que uma pessoa decrementa é um contador
que uma pessoa erra, e quando erra ninguém percebe. Com uma linha por mês, "quanto
de janeiro já está comprometido" é um `SUM` agrupado por mês. Não existe contador
para errar.

Consequência direta: **`installments` é a tabela que o app lê**. `entries` guarda a
intenção, `installments` guarda o dinheiro no tempo.

## Relações

```
auth.users
   │
   ├── cards ──────────┐
   ├── people ─────┐   │
   ├── categories ─┼───┼──┐
   │               │   │  │
   ├── entries ────┴───┴──┘        (card_id nullable: pix e dinheiro)
   │      │                        (reimburser_id nullable: quem paga de volta)
   │      └── installments         (1 linha por mês de vencimento)
   │
   ├── incomes
   │
   └── debts ──── debt_payments ──► entries   (o pagamento também é um lançamento)
```

## Tipos

`entries.type` tem exatamente três valores:

| valor | o que é | exemplos |
| --- | --- | --- |
| `recorrente` | sem fim definido | aluguel, luz, internet, DAS, iCloud, YouTube, plano de saúde do pet |
| `parcelado` | número fechado de parcelas | notebook em 12x, dentista em 6x |
| `avulso` | compra única | mercado, iFood, gasolina |

Note que `recorrente` cobre assinatura, mas não se chama assinatura, porque nem
toda recorrente é assinatura. Aluguel e DAS são recorrentes.

`installments.status` tem exatamente dois valores:

| valor | o que é |
| --- | --- |
| `previsto` | gasto que vai acontecer mas ainda não aconteceu, estimativa. Era o fundo amarelo da planilha. Todo mês futuro nasce assim. |
| `confirmado` | aconteceu, valor real |

`cards.mode` tem dois valores: `detalhado` (lançamento item a item, caso do cartão
principal) e `agregado` (só o total da fatura interessa).

## Campos por tabela

### `cards`
Cartões e contas, cadastrados por ela na tela de configurações. `mode` define se
o cartão é lançado item a item, e `is_default` marca o que o bot assume quando
ela não diz qual, no máximo um por pessoa.
`closing_day` e `due_day` são dia do mês, porque fechamento e vencimento não
coincidem. Compra feita a partir do dia do fechamento entra na fatura seguinte, e
fatura que vence antes do dia do fechamento vence no mês seguinte ao que ela
fecha. Os dois deslocamentos mexem em `due_date` e não em `reference_month`.
Ver [ADR 0014](DECISIONS.md). `closing_day` nulo quer dizer sem fechamento.

No próprio dia do fechamento a regra não decide sozinha, porque a fatura fecha
numa hora do dia que ninguém publica. Aí quem decide é ela, por um botão na
confirmação do bot, e `closing_confirmed_month` guarda o mês já respondido.
Ver [ADR 0016](DECISIONS.md).

### `people`
Pessoas envolvidas em reembolso e em dívida.

### `categories`
Nome, cor e ícone. Quinze, fechadas, criadas no seed. O bot escolhe dentro delas
e nunca cria uma nova. Ver [ADR 0021](DECISIONS.md).

### `entries`
A compra ou a conta. `amount_exact` separa a conta fixa, cujo valor já é o valor
final, da conta que varia e cujo valor futuro é chute. Ver
[ADR 0017](DECISIONS.md). `card_id` é nullable, porque pix e dinheiro não têm cartão.
`reimburser_id` é nullable e, quando preenchido, quer dizer que o gasto passou no
cartão dela mas é de outra pessoa. `installments_total` é 1 para tudo que não é
parcelado.

### `installments`
Uma linha por mês. Guarda dois momentos diferentes de propósito:

- `reference_month`: quando o gasto aconteceu, sempre dia 1 do mês. É por aqui que
  o gráfico de categorias agrupa.
- `due_date`: quando o dinheiro sai. É por aqui que a home e os próximos meses
  agrupam. `due_day_exact` diz se o dia dessa data foi decidido por alguém ou
  pelo cartão. Quando é `false`, só o mês vale, e a data guardada é o último dia
  dele. Ver [ADR 0015](DECISIONS.md).

A interface nunca oferece um botão de trocar entre um e outro. A home é sempre por
`due_date`, e só o gráfico de categorias usa `reference_month`, com o rótulo
escrito por extenso.

### `incomes`
As entradas do mês. Tem `status` igual ao das parcelas, porque salário do mês que
vem também é previsto.

### `debts` e `debt_payments`
Módulo separado, não é um `type` de `entries`. `debts` guarda o valor emprestado,
`debt_payments` guarda cada pagamento, e o saldo é sempre derivado. Cada pagamento
aponta para a `entry` avulsa que ele gerou na categoria dívidas, para o dinheiro
aparecer na home no dia em que sai.

---

## Migrations

### `supabase/migrations/0001_init.sql`

```sql
-- ---------------------------------------------------------------------------
-- Sobra, esquema inicial.
-- ---------------------------------------------------------------------------

create extension if not exists pgcrypto;

create type entry_type         as enum ('recorrente', 'parcelado', 'avulso');
create type installment_status as enum ('previsto', 'confirmado');
create type card_mode          as enum ('detalhado', 'agregado');
create type debt_direction     as enum ('eu_devo', 'me_devem');

-- cartões e contas ----------------------------------------------------------
create table public.cards (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users (id) on delete cascade,
  name        text not null,
  mode        card_mode not null default 'detalhado',
  closing_day smallint check (closing_day between 1 and 31),
  due_day     smallint check (due_day     between 1 and 31),
  -- mês do ciclo em que ela já confirmou que a fatura fechou, sempre dia 1.
  -- Enquanto for o ciclo corrente, o bot para de oferecer a saída do dia do
  -- fechamento. Só o bot escreve, o front nunca lê. Ver ADR 0016.
  closing_confirmed_month date
    check (closing_confirmed_month is null
           or extract(day from closing_confirmed_month) = 1),
  color       text,
  -- o cartão que entra quando ela não diz qual. Um por pessoa, garantido pelo
  -- índice parcial abaixo. Ver ADR 0013 para o idioma do nome.
  is_default  boolean not null default false,
  position    smallint not null default 0,
  archived_at timestamptz,
  created_at  timestamptz not null default now(),
  unique (user_id, name)
);

-- no máximo um cartão padrão por pessoa
create unique index cards_one_default
  on public.cards (user_id) where is_default;

-- vínculo do Telegram: um chat por pessoa, uma pessoa por chat. Ver ADR 0019.
create table public.telegram_links (
  user_id   uuid primary key references auth.users (id) on delete cascade,
  chat_id   bigint not null unique,
  linked_at timestamptz not null default now()
);

-- token de pareamento, de uso único e vida curta. Guardado com hash, nunca em
-- texto, e apagado ao ser usado.
create table public.telegram_link_tokens (
  token_hash text primary key,
  user_id    uuid not null references auth.users (id) on delete cascade,
  expires_at timestamptz not null,
  created_at timestamptz not null default now()
);

-- pessoas, para reembolso e dívida -------------------------------------------
create table public.people (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references auth.users (id) on delete cascade,
  name       text not null,
  created_at timestamptz not null default now(),
  unique (user_id, name)
);

-- categorias -----------------------------------------------------------------
create table public.categories (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references auth.users (id) on delete cascade,
  name       text not null,
  color      text not null,
  icon       text not null,
  position   smallint not null default 0,
  created_at timestamptz not null default now(),
  unique (user_id, name)
);

-- a compra ou a conta ---------------------------------------------------------
create table public.entries (
  id                 uuid primary key default gen_random_uuid(),
  user_id            uuid not null references auth.users (id) on delete cascade,
  description        text not null check (length(btrim(description)) > 0),
  total_amount       numeric(12,2) not null check (total_amount >= 0),
  purchase_date      date not null,
  card_id            uuid references public.cards (id)      on delete set null,
  category_id        uuid not null references public.categories (id) on delete restrict,
  type               entry_type not null,
  installments_total smallint not null default 1 check (installments_total >= 1),
  reimburser_id      uuid references public.people (id) on delete set null,
  -- só para recorrente: null quer dizer sem fim definido
  ends_on            date,
  -- true quer dizer que o valor não é estimativa, é o mesmo todo mês, como
  -- aluguel. Recorrente assim confirma sozinho no vencimento. Ver ADR 0017.
  amount_exact       boolean not null default false,
  notes              text,
  source             text not null default 'app'
                     check (source in ('app', 'telegram', 'import')),
  created_at         timestamptz not null default now(),

  -- parcelado é o único type com mais de uma parcela
  constraint entries_installments_match_type check (
    (type =  'parcelado' and installments_total >  1) or
    (type <> 'parcelado' and installments_total = 1)
  ),
  -- ends_on só faz sentido em recorrente
  constraint entries_ends_on_only_recurring check (
    ends_on is null or type = 'recorrente'
  )
);

-- uma linha por mês de vencimento ---------------------------------------------
create table public.installments (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null references auth.users (id) on delete cascade,
  entry_id        uuid not null references public.entries (id) on delete cascade,
  number          smallint not null check (number >= 1),
  amount          numeric(12,2) not null check (amount >= 0),
  -- quando o gasto aconteceu, sempre dia 1
  reference_month date not null check (extract(day from reference_month) = 1),
  -- quando o dinheiro sai
  due_date        date not null,
  -- false quer dizer que só o mês de due_date é confiável, o dia é enchimento.
  -- Acontece em previsto lançado sem dia. Ver ADR 0015.
  due_day_exact   boolean not null default true,
  status          installment_status not null default 'previsto',
  confirmed_at    timestamptz,
  created_at      timestamptz not null default now(),
  unique (entry_id, number),
  constraint installments_confirmado_has_date check (
    (status = 'confirmado' and confirmed_at is not null) or
    (status = 'previsto'   and confirmed_at is null)
  )
);

create index installments_by_due_date on public.installments (user_id, due_date);
create index installments_by_reference_month on public.installments (user_id, reference_month);
create index installments_by_entry      on public.installments (entry_id);
create index entries_by_category       on public.entries (user_id, category_id);
create index entries_by_reimburser       on public.entries (user_id, reimburser_id)
  where reimburser_id is not null;

-- entradas do mês --------------------------------------------------------------
create table public.incomes (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null references auth.users (id) on delete cascade,
  description     text not null,
  amount          numeric(12,2) not null check (amount >= 0),
  reference_month date not null check (extract(day from reference_month) = 1),
  expected_on     date,
  received_on     date,
  status          installment_status not null default 'previsto',
  recurring       boolean not null default false,
  created_at      timestamptz not null default now()
);

create index incomes_by_month on public.incomes (user_id, reference_month);

-- dívidas, módulo separado -------------------------------------------------------
create table public.debts (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references auth.users (id) on delete cascade,
  person_id   uuid not null references public.people (id) on delete restrict,
  description text,
  principal   numeric(12,2) not null check (principal > 0),
  direction   debt_direction not null default 'eu_devo',
  opened_on   date not null,
  closed_at   timestamptz,
  created_at  timestamptz not null default now()
);

create table public.debt_payments (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references auth.users (id) on delete cascade,
  debt_id    uuid not null references public.debts (id) on delete cascade,
  amount     numeric(12,2) not null check (amount > 0),
  paid_on    date not null,
  -- o pagamento também é um lançamento avulso na categoria dívidas
  entry_id   uuid references public.entries (id) on delete set null,
  created_at timestamptz not null default now()
);

create index debt_payments_by_debt on public.debt_payments (debt_id, paid_on);
```

### `supabase/migrations/0002_rls.sql`

```sql
-- ---------------------------------------------------------------------------
-- RLS em todas as tabelas. App de uma pessoa só, mas um token vazado não pode
-- virar leitura do banco inteiro.
-- ---------------------------------------------------------------------------

do $$
declare t text;
begin
  foreach t in array array[
    'cards', 'people', 'categories', 'entries',
    'installments', 'incomes', 'debts', 'debt_payments'
  ] loop
    execute format('alter table public.%I enable row level security', t);
    execute format('alter table public.%I force  row level security', t);
    execute format($f$
      create policy %I on public.%I
        for all
        to authenticated
        using      (user_id = (select auth.uid()))
        with check (user_id = (select auth.uid()))
    $f$, t || '_dono', t);
  end loop;
end $$;
```

### `supabase/migrations/0003_installments.sql`

```sql
-- ---------------------------------------------------------------------------
-- Criar lançamento é sempre por aqui. Nenhum cliente escreve em installments
-- diretamente, senão a garantia de "N linhas, uma por mês" deixa de valer.
-- ---------------------------------------------------------------------------

-- O hoje dela, não o hoje do servidor. O Worker roda na Cloudflare e o Postgres
-- responde em UTC, e o cartão fecha num dia baixo do mês, então uma mensagem da
-- noite do dia 1 viraria dia 2 e deslocaria a fatura em um mês. Ver ADR 0015.
create or replace function public.sobra_today() returns date
language sql stable as $$
  select (now() at time zone 'America/Sao_Paulo')::date;
$$;

-- Quantos meses a fatura anda em relação ao mês da compra. Vale 0, 1 ou 2:
-- +1 quando a compra é no dia do fechamento ou depois dele,
-- +1 quando o cartão vence antes do dia em que fecha.
-- Sem cartão, ou sem closing_day, não anda. Ver ADR 0014.
--
-- p_before_closing é a saída do dia do fechamento, e só tem efeito quando a
-- compra é no próprio dia em que o cartão fecha, porque é o único dia ambíguo
-- do mês. Ver ADR 0016.
create or replace function public.sobra_due_shift(
  p_card_id        uuid,
  p_purchase_date  date,
  p_before_closing boolean default false
) returns int
language sql stable as $$
  select coalesce((
    select (case when c.closing_day is null
                   or extract(day from p_purchase_date)::int < c.closing_day
                   or (p_before_closing
                       and extract(day from p_purchase_date)::int = c.closing_day)
                 then 0 else 1 end)
         + (case when c.closing_day is null or c.due_day is null
                   or c.due_day >= c.closing_day
                 then 0 else 1 end)
      from public.cards c
     where c.id = p_card_id
  ), 0);
$$;

-- Em que mês cai a fatura de uma compra. Sempre dia 1.
create or replace function public.sobra_due_month(
  p_card_id        uuid,
  p_purchase_date  date,
  p_before_closing boolean default false
) returns date
language sql stable as $$
  select (date_trunc('month', p_purchase_date)
          + make_interval(months => public.sobra_due_shift(p_card_id, p_purchase_date,
                                                           p_before_closing)))::date;
$$;

-- Em que dia o dinheiro sai, dado o cartão e o mês da fatura. Encurta para o
-- último dia quando o mês é mais curto que due_day. Quem chama já resolveu o
-- deslocamento do fechamento com sobra_due_month.
-- Sem cartão (pix, dinheiro), sai no dia da compra.
create or replace function public.sobra_due_date(
  p_card_id  uuid,
  p_month    date,
  p_fallback date,
  p_exact    boolean default true
) returns date
language sql stable as $$
  select case
    -- dia desconhecido: guarda o último dia do mês. Ver ADR 0015.
    when not p_exact
      then (date_trunc('month', p_month) + interval '1 month' - interval '1 day')::date
    when p_card_id is null then p_fallback
    else coalesce((
      select p_month + (least(c.due_day, extract(day from (p_month + interval '1 month' - interval '1 day'))::int) - 1)
      from public.cards c
      where c.id = p_card_id and c.due_day is not null
    ), p_fallback)
  end;
$$;

-- Cria a entry e materializa as installments na mesma transação.
-- Arredondamento: as primeiras parcelas ficam com o centavo para baixo e a
-- última absorve a diferença, para a soma bater com o total exato.
create or replace function public.create_entry_with_installments(
  p_description        text,
  p_total_amount       numeric,
  p_purchase_date      date,
  p_category_id        uuid,
  p_type               entry_type,
  p_installments_total smallint default 1,
  p_card_id            uuid     default null,
  p_reimburser_id      uuid     default null,
  p_status             installment_status default 'previsto',
  p_first_due_month    date     default null,
  p_due_day_exact      boolean  default true,
  p_before_closing     boolean  default false,
  p_ends_on            date     default null,
  p_horizon_months     int      default 12,
  p_notes              text     default null,
  p_source             text     default 'app'
) returns uuid
language plpgsql security invoker as $$
declare
  v_user      uuid := auth.uid();
  v_entry     uuid;
  v_n         int;
  v_base      numeric(12,2);
  v_last    numeric(12,2);
  v_ref_one    date;   -- primeiro reference_month, o mês da compra
  v_due_one    date;   -- mês da primeira fatura, já com o fechamento aplicado
  v_month      date;   -- reference_month da parcela da vez
  v_due        date;   -- mês de fatura da parcela da vez
  v_amount     numeric(12,2);
  i           int;
begin
  if v_user is null then
    raise exception 'sem usuário autenticado';
  end if;

  insert into public.entries (
    user_id, description, total_amount, purchase_date, card_id, category_id,
    type, installments_total, reimburser_id, ends_on, notes, source
  ) values (
    v_user, p_description, p_total_amount, p_purchase_date, p_card_id, p_category_id,
    p_type,
    case when p_type = 'parcelado' then p_installments_total else 1 end,
    p_reimburser_id, p_ends_on, p_notes, p_source
  ) returning id into v_entry;

  -- Dois trilhos: a competência anda pelo mês da compra, a fatura anda pelo
  -- fechamento do cartão. Ver ADR 0014. p_first_due_month, quando vem
  -- preenchido, manda no trilho da fatura e não no da competência.
  v_ref_one := date_trunc('month', p_purchase_date)::date;

  -- Sem dia de compra não há como comparar com o fechamento, então o previsto
  -- inexato fica no mês que ela nomeou. Ver ADR 0015.
  v_due_one := coalesce(p_first_due_month,
                        case when p_due_day_exact
                             then public.sobra_due_month(p_card_id, p_purchase_date,
                                                         p_before_closing)
                             else v_ref_one end);

  if p_type = 'parcelado' then
    -- N parcelas, uma por mês, valor dividido com a última absorvendo o resto
    v_n      := p_installments_total;
    v_base   := trunc(p_total_amount / v_n, 2);
    v_last := p_total_amount - (v_base * (v_n - 1));

    for i in 1 .. v_n loop
      v_due    := (v_due_one + make_interval(months => i - 1))::date;
      v_amount := case when i = v_n then v_last else v_base end;

      insert into public.installments (
        user_id, entry_id, number, amount, reference_month, due_date, due_day_exact,
        status, confirmed_at
      ) values (
        v_user, v_entry, i, v_amount,
        v_ref_one,
        public.sobra_due_date(p_card_id, v_due, (p_purchase_date + ((i - 1) * interval '1 month'))::date, p_due_day_exact),
        p_due_day_exact,
        case when i = 1 then p_status else 'previsto' end,
        case when i = 1 and p_status = 'confirmado' then now() else null end
      );
    end loop;

  elsif p_type = 'recorrente' then
    -- horizonte rolante: materializa p_horizon_months meses à frente
    for i in 1 .. p_horizon_months loop
      v_month := (v_ref_one + make_interval(months => i - 1))::date;
      v_due   := (v_due_one + make_interval(months => i - 1))::date;
      exit when p_ends_on is not null and v_month > date_trunc('month', p_ends_on)::date;

      insert into public.installments (
        user_id, entry_id, number, amount, reference_month, due_date, due_day_exact,
        status, confirmed_at
      ) values (
        v_user, v_entry, i, p_total_amount, v_month,
        public.sobra_due_date(p_card_id, v_due, (v_month + (extract(day from p_purchase_date)::int - 1)), p_due_day_exact),
        p_due_day_exact,
        case when i = 1 then p_status else 'previsto' end,
        case when i = 1 and p_status = 'confirmado' then now() else null end
      );
    end loop;

  else
    -- avulso: uma linha só
    insert into public.installments (
      user_id, entry_id, number, amount, reference_month, due_date, due_day_exact,
      status, confirmed_at
    ) values (
      v_user, v_entry, 1, p_total_amount,
      v_ref_one,
      public.sobra_due_date(p_card_id, v_due_one, p_purchase_date, p_due_day_exact),
      p_due_day_exact,
      p_status,
      case when p_status = 'confirmado' then now() else null end
    );
  end if;

  return v_entry;
end $$;

-- Estende o horizonte dos recorrentes.
-- Chamada pelo front no boot do app e pelo Worker antes de gravar. Sem job.
-- Preenche a partir do último mês já gravado, nunca a partir de hoje, então
-- rodar atrasada preenche o que faltou em vez de deixar buraco no meio.
-- Idempotente: rodar duas vezes seguidas não cria nada na segunda.
create or replace function public.sobra_extend_recurring(p_horizon_months int default 12)
returns int
language plpgsql security definer set search_path = public as $$
declare
  r        record;
  v_limit date := (date_trunc('month', public.sobra_today()) + make_interval(months => p_horizon_months))::date;
  v_next   date;   -- reference_month da próxima linha
  v_shift  int;    -- distância fixa entre competência e fatura, ADR 0014
  v_num    int;
  v_created int := 0;
begin
  for r in
    select e.id, e.user_id, e.total_amount, e.card_id, e.purchase_date, e.ends_on,
           max(i.reference_month)     as last_month,
           max(i.number)              as ultimo_num,
           bool_and(i.due_day_exact)  as due_day_exact
    from public.entries e
    join public.installments i on i.entry_id = e.id
    where e.type = 'recorrente'
      and (e.ends_on is null or e.ends_on >= public.sobra_today())
    group by e.id
  loop
    v_next  := (r.last_month + interval '1 month')::date;
    v_num   := r.ultimo_num;
    v_shift := public.sobra_due_shift(r.card_id, r.purchase_date);

    while v_next <= v_limit
      and (r.ends_on is null or v_next <= date_trunc('month', r.ends_on)::date)
    loop
      v_num := v_num + 1;
      insert into public.installments (
        user_id, entry_id, number, amount, reference_month, due_date, due_day_exact, status
      ) values (
        r.user_id, r.id, v_num, r.total_amount, v_next,
        public.sobra_due_date(r.card_id,
                              (v_next + make_interval(months => v_shift))::date,
                              (v_next + (extract(day from r.purchase_date)::int - 1)),
                              r.due_day_exact),
        r.due_day_exact,
        'previsto'
      );
      v_created := v_created + 1;
      v_next := (v_next + interval '1 month')::date;
    end loop;
  end loop;

  return v_created;
end $$;

-- Confirmar um previsto, opcionalmente com o valor real e a data real.
-- Confirmar é o momento em que o dia deixa de ser chute, então a data dita manda,
-- e o previsto que não tinha dia recebe o de hoje. Quem já tinha dia exato, como
-- parcela de cartão, mantém o dele. Ver ADR 0015.
create or replace function public.confirm_installment(
  p_installment_id uuid,
  p_amount         numeric default null,
  p_due_date       date    default null
) returns void
language sql security invoker as $$
  update public.installments
     set status        = 'confirmado',
         amount        = coalesce(p_amount, amount),
         due_date      = coalesce(p_due_date,
                                  case when due_day_exact then due_date
                                       else public.sobra_today() end),
         due_day_exact = true,
         confirmed_at  = now()
   where id = p_installment_id
     and user_id = (select auth.uid());
$$;

-- Confirma sozinho o recorrente de valor exato que já venceu. Não existe chute
-- a resolver nele, então pedir confirmação seria pedir um toque para não mudar
-- número nenhum. Nunca confirma para frente. Ver ADR 0017.
-- Roda no mesmo boot em que sobra_extend_recurring roda, sem job (ADR 0011).
create or replace function public.sobra_confirm_fixed()
returns int
language plpgsql security invoker as $$
declare
  v_count int;
begin
  update public.installments i
     set status       = 'confirmado',
         confirmed_at = now()
    from public.entries e
   where e.id = i.entry_id
     and e.type = 'recorrente'
     and e.amount_exact
     and i.status = 'previsto'
     and i.due_date <= public.sobra_today()
     and i.user_id = (select auth.uid());

  get diagnostics v_count = row_count;
  return v_count;
end $$;
```

### `supabase/migrations/0004_views.sql`

```sql
-- ---------------------------------------------------------------------------
-- Views que o front lê. O front não recalcula regra de negócio.
-- Lançamento com reimburser_id não entra em nenhum total de gasto dela: o
-- dinheiro sai do cartão, mas o gasto é de outra pessoa.
-- ---------------------------------------------------------------------------

-- Home: por quando o dinheiro sai.
create or replace view public.v_month_cashflow
with (security_invoker = true) as
select
  i.user_id,
  date_trunc('month', i.due_date)::date                              as month,
  sum(i.amount) filter (where i.status = 'confirmado')               as confirmado,
  sum(i.amount) filter (where i.status = 'previsto')                 as previsto,
  sum(i.amount)                                                      as total,
  count(*)      filter (where i.status = 'confirmado')               as confirmado_count,
  count(*)      filter (where i.status = 'previsto')                 as previsto_count
from public.installments i
join public.entries e on e.id = i.entry_id
where e.reimburser_id is null
group by 1, 2;

-- Próximos meses: quanto de cada mês já está comprometido, separado por natureza.
create or replace view public.v_month_committed
with (security_invoker = true) as
select
  i.user_id,
  date_trunc('month', i.due_date)::date                        as month,
  sum(i.amount) filter (where e.type = 'recorrente')           as recorrentes,
  sum(i.amount) filter (where e.type = 'parcelado')            as parcelas,
  sum(i.amount) filter (where e.type = 'avulso')               as avulsos,
  sum(i.amount)                                                as total
from public.installments i
join public.entries e on e.id = i.entry_id
where e.reimburser_id is null
group by 1, 2;

-- Renda por mês, para a mesma tela.
create or replace view public.v_month_income
with (security_invoker = true) as
select
  user_id,
  reference_month                                        as month,
  sum(amount)                                            as total,
  sum(amount) filter (where status = 'confirmado')       as confirmada
from public.incomes
group by 1, 2;

-- Único lugar do app organizado por data da compra.
create or replace view public.v_category_by_purchase_month
with (security_invoker = true) as
select
  i.user_id,
  i.reference_month  as month,
  c.id               as category_id,
  c.name             as category_name,
  c.color            as category_color,
  c.icon             as category_icon,
  sum(i.amount)      as total
from public.installments i
join public.entries    e on e.id = i.entry_id
join public.categories c on c.id = e.category_id
where e.reimburser_id is null
group by 1, 2, 3, 4, 5, 6;

-- Reembolso sempre derivado, nunca digitado.
create or replace view public.v_reimbursements
with (security_invoker = true) as
select
  i.user_id,
  p.id                                                   as person_id,
  p.name                                                 as person_name,
  date_trunc('month', i.due_date)::date                  as month,
  sum(i.amount)                                          as total,
  count(*)                                               as entry_count
from public.installments i
join public.entries e on e.id = i.entry_id
join public.people  p on p.id = e.reimburser_id
group by 1, 2, 3, 4;

-- Saldo de dívida sempre derivado dos pagamentos.
create or replace view public.v_debt_balances
with (security_invoker = true) as
select
  d.user_id,
  d.id                                                        as debt_id,
  d.person_id,
  p.name                                                      as person_name,
  d.direction,
  d.principal,
  coalesce(sum(dp.amount), 0)                                 as paid,
  d.principal - coalesce(sum(dp.amount), 0)                   as balance,
  max(dp.paid_on)                                             as last_payment,
  d.opened_on,
  d.closed_at
from public.debts d
join public.people p on p.id = d.person_id
left join public.debt_payments dp on dp.debt_id = d.id
group by d.id, p.name;
```

### `supabase/migrations/0005_seed.sql`

```sql
-- ---------------------------------------------------------------------------
-- Categorias iniciais. Cores da paleta análoga e dessaturada da identidade.
-- Roda por usuário, no primeiro login.
-- ---------------------------------------------------------------------------

create or replace function public.sobra_seed_categories(p_user uuid)
returns void
language sql security definer set search_path = public as $$
  insert into public.categories (user_id, name, color, icon, position)
  values
    (p_user, 'moradia',         '#7E94A8', 'moradia',       1),
    (p_user, 'mercado',         '#8FA68E', 'mercado',       2),
    (p_user, 'restaurantes',    '#C9A46A', 'restaurantes',  3),
    (p_user, 'transporte',      '#6E9490', 'transporte',    4),
    (p_user, 'saúde',           '#B0779E', 'saude',         5),
    (p_user, 'pet',             '#C08573', 'gato',          6),
    (p_user, 'casa e objetos',  '#B7A88F', 'casa',          7),
    (p_user, 'roupas e beleza', '#9A8FB0', 'roupas',        8),
    (p_user, 'assinaturas',     '#8A93A0', 'assinaturas',   9),
    (p_user, 'lazer',           '#83A8AF', 'ingresso',     10),
    (p_user, 'educação',        '#7D7A9C', 'educacao',     11),
    (p_user, 'viagem',          '#8FA9C0', 'viagem',       12),
    (p_user, 'presentes',       '#C48B8B', 'presentes',    13),
    (p_user, 'impostos',        '#9C9186', 'impostos',     14),
    (p_user, 'dívidas',         '#9E6B72', 'dividas',      15)
  on conflict (user_id, name) do nothing;
$$;
```

---

## Consultas que respondem as três telas

**Home de agosto, os três números mais a renda prevista do alto do card.**

```sql
select (select total from public.v_month_income
         where month = date '2026-08-01') as total,
       confirmado,
       previsto,
       (select total from public.v_month_income
         where month = date '2026-08-01') - total as sobra
from public.v_month_cashflow
where month = date '2026-08-01';
```

**Próximos seis meses, quanto já está comprometido.**

```sql
select c.month,
       c.recorrentes, c.parcelas, c.programados, c.comprometido,
       r.total,
       r.total - c.comprometido            as sobra_prevista,
       round(100 * c.comprometido / r.total) as pct
from public.v_month_committed c
join public.v_month_income    r using (month)
where c.month > date_trunc('month', public.sobra_today())
  and c.month <= date_trunc('month', public.sobra_today()) + interval '6 months'
order by c.month;
```

**Quanto uma pessoa deve neste mês.**

```sql
select person_name, total, entry_count
from public.v_reimbursements
where month = date_trunc('month', public.sobra_today())::date;
```

Se a fatura do cartão for maior que o total dele, a diferença é gasto dela, porque
o que não tem `reimburser_id` já entrou nos totais normais.

## Regras que o schema garante, e as que ele não garante

**Garante:**

- Parcelado sempre tem mais de uma parcela, e só parcelado tem.
- A soma das parcelas bate com o total da compra, ao centavo.
- `reference_month` é sempre dia 1.
- Confirmado sempre tem data de confirmação.
- Ninguém lê linha de outro usuário.

**Não garante, e por isso mora na aplicação:**

- Que a categoria escolhida faça sentido.
- Que a estimativa de um previsto seja boa.
- Que o horizonte dos recorrentes esteja em dia, o que depende de alguém abrir o
  app ou falar com o bot. O horizonte pode ficar curto, mas nunca esburacado.
