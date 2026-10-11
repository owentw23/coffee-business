# Plan 1 v2: Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the monorepo, the Supabase schema (menu, milk options, beans, bean cycles, sales), `record_sale` with bean slots and giveaways, the public `site_snapshot()` with the reveal embargo, a Telegram bot skeleton, a staff web app (login + menu editor) and a public website (menu + about), all deployed on a custom domain.

**Architecture:**
- Postgres in Supabase (Singapore) is the ERP.
- Staff read and write through RLS. Price-snapshotting writes go through `security definer` functions.
- The public reads only through `site_snapshot()`.
- Two static React + Vite apps share a `packages/shared` workspace and run on Cloudflare Pages: `apps/site` at `<domain>`, and `apps/staff` at `staff.<domain>`.
- Edge Functions are thin Deno handlers around pure TypeScript in `supabase/functions/_shared/`. Vitest tests those modules in Node.

**Tech Stack:** Supabase (Postgres, Auth, Edge Functions, pgTAP via `supabase test db`), Supabase CLI via npm, npm workspaces, TypeScript, Vitest, React + Vite, React Router, zod, Testing Library, Cloudflare Pages, Telegram Bot API.

**Spec:** `docs/superpowers/specs/2026-10-11-operations-automation-design-v2.md`. Roadmap: `docs/superpowers/plans/2026-10-11-roadmap-v2.md`.

## Global Constraints

- Money is integer **cents** (`*_cents`). Ingredient cost is integer **micros** of SGD per unit (`*_micros`).
- Business dates use **Asia/Singapore**. Use `public.sg_today()` in SQL; never use `current_date`.
- RLS is enabled on every `public` table. The public (`anon`) can only execute `public.site_snapshot()`.
- Bean cycles start on a **Saturday**; reveal = start + 7; end = start + 13. Before reveal, nothing public may contain a weekend bean's real name, origin or notes. `accepted_terms` is never public.
- Hosting: Cloudflare Pages (free) for both apps; Supabase in the Singapore region.
- Secrets live only in Supabase / Cloudflare / GitHub secret stores. Commit `.env.example` only.
- No `console.log` in committed code. User-facing errors are plain-English sentences.
- TypeScript: immutable updates; functions under 50 lines; files under 400 lines.
- TDD on every task. Conventional commits. Do not push unless the owner asks.
- Before starting, read `PROGRESS.md` and check `git status`. Never overwrite another assistant's uncommitted work. `.codex/` belongs to Codex; leave it alone.

## File Structure

```
package.json, tsconfig.json, vitest.config.ts, .env.example, .github/workflows/ci.yml
packages/shared/                       @cafe/shared (consumed as TS source)
  package.json
  src/index.ts
  src/money.ts                         parsePriceToCents, formatCents
  src/snapshot.ts                      SiteSnapshot types + menu grouping helpers
  src/money.test.ts, src/snapshot.test.ts
supabase/config.toml
supabase/migrations/
  20261012000100_catalogue.sql         menu, modifiers, item-modifier links, ingredients, recipes
  20261012000200_staff_rls.sql         staff_roles, is_staff(), is_admin(), catalogue RLS
  20261012000300_bean_cycles.sql       sg_today(), bean_cycles, cycle_beans, cycle_bean_id()
  20261012000400_sales.sql             customers, orders, order_items, stock_movements, record_sale()
  20261012000500_site_settings.sql     settings, site_content, job_runs, site_snapshot()
supabase/tests/00_smoke … 05_site_settings.test.sql
supabase/functions/_shared/telegram.ts
supabase/functions/telegram-webhook/index.ts
tests/functions/telegram.test.ts
apps/staff/                            staff.<domain>: login, role guard, menu editor
apps/site/                             <domain>: home, menu, about
```

---

### Task 1: Monorepo, shared package and CI

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`, `.env.example`, `.github/workflows/ci.yml`, `supabase/tests/00_smoke.test.sql`, `packages/shared/package.json`, `packages/shared/src/index.ts`, `packages/shared/src/money.ts`, `packages/shared/src/money.test.ts`
- Generated: `supabase/config.toml`
- Modify: `.gitignore`

**Interfaces:**
- Produces:
  - commands `npm run db:start | db:reset | db:test`, `npm test`, `npm run typecheck`;
  - `@cafe/shared` exports `parsePriceToCents(input: string): number | null` and `formatCents(cents: number): string`.

- [ ] **Step 1: Root package and tools**

```bash
cd "/Users/owentw/Documents/coffee business"
npm init -y
npm pkg set name=cafe-ops private=true type=module
npm pkg set 'workspaces[0]=packages/*'
npm pkg set scripts.db:start="supabase start" scripts.db:stop="supabase stop" scripts.db:reset="supabase db reset" scripts.db:test="supabase test db"
npm pkg set scripts.test="vitest run" scripts.typecheck="tsc -p tsconfig.json --noEmit"
npm install -D supabase vitest typescript
npx supabase init --force
mkdir -p packages/shared/src
```

`packages/shared/package.json`:

```json
{
  "name": "@cafe/shared",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" }
}
```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "skipLibCheck": true,
    "types": []
  },
  "include": ["tests/**/*.ts", "packages/shared/src/**/*.ts", "supabase/functions/_shared/**/*.ts"]
}
```

`vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: { include: ['tests/**/*.test.ts', 'packages/**/*.test.ts'] },
});
```

- [ ] **Step 2: Write the failing test**

`packages/shared/src/money.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { formatCents, parsePriceToCents } from './money';

describe('parsePriceToCents', () => {
  it.each([
    ['5.5', 550], ['5.50', 550], ['5', 500], ['S$6.30', 630], ['$1', 100], [' 12.05 ', 1205],
  ])('parses %s', (input, cents) => expect(parsePriceToCents(input)).toBe(cents));

  it.each(['', 'abc', '5.555', '-1', '1000', '5,50'])('rejects %s', (input) =>
    expect(parsePriceToCents(input)).toBeNull());
});

describe('formatCents', () => {
  it('formats Singapore dollars', () => expect(formatCents(630)).toBe('S$6.30'));
  it('formats zero', () => expect(formatCents(0)).toBe('S$0.00'));
});
```

`supabase/tests/00_smoke.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(1);
select ok(true, 'pgTAP runs');
select * from finish();
rollback;
```

- [ ] **Step 3: Run and verify the failure**

Run: `npm test`
Expected: FAIL with `Failed to resolve import "./money"`.

- [ ] **Step 4: Implement**

`packages/shared/src/money.ts`:

```ts
const PRICE_PATTERN = /^\d{1,3}(\.\d{1,2})?$/;

export function parsePriceToCents(input: string): number | null {
  const cleaned = input.trim().replace(/^S?\$/i, '');
  if (!PRICE_PATTERN.test(cleaned)) return null;
  const [dollars, cents = ''] = cleaned.split('.');
  return Number(dollars) * 100 + Number(cents.padEnd(2, '0'));
}

export function formatCents(cents: number): string {
  return `S$${(cents / 100).toFixed(2)}`;
}
```

`packages/shared/src/index.ts`:

```ts
export * from './money';
```

- [ ] **Step 5: Run both suites**

Run: `npm test`, then `npm run db:start` (the first run pulls Docker images and takes a few minutes), then `npm run db:test`.
Expected: Vitest 14 passed; pgTAP `00_smoke.test.sql .. ok`.

- [ ] **Step 6: Env example, gitignore, CI**

`.env.example`:

```bash
# apps/site and apps/staff (Cloudflare Pages env vars; local: apps/*/.env.local)
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
# Supabase edge function secrets
TELEGRAM_BOT_TOKEN=
TELEGRAM_WEBHOOK_SECRET=
```

Append to `.gitignore`:

```
node_modules/
dist/
supabase/.temp/
supabase/.branches/
supabase/functions/.env.local
apps/*/.env.local
```

`.github/workflows/ci.yml`:

```yaml
name: ci
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run typecheck
      - run: npx supabase start -x studio,imgproxy,inbucket,edge-runtime,logflare,vector
      - run: npm run db:test
      - run: npm test
```

(Tasks 8 and 10 add the app build steps.)

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json tsconfig.json vitest.config.ts .env.example .gitignore .github/workflows/ci.yml supabase/config.toml supabase/tests/00_smoke.test.sql packages/shared
git commit -m "chore: scaffold monorepo, shared money utils, supabase and ci"
```

---

### Task 2: Catalogue schema (menu, milk options, beans, recipes)

**Files:**
- Create: `supabase/migrations/20261012000100_catalogue.sql`
- Test: `supabase/tests/01_catalogue.test.sql`

**Interfaces:**
- Produces:
  - `menu_items(id, name, description, category, price_cents, sold_on, uses_bean_slot, active, sold_out, sort_rank)`;
  - `modifiers(id, name, group_name, price_delta_cents, active)`;
  - `menu_item_modifiers(menu_item_id, modifier_id)`;
  - `ingredients(..., bean_use)`;
  - `recipe_lines(id, menu_item_id, ingredient_id NULL, chosen_bean, qty)`;
  - `modifier_lines(modifier_id, ingredient_id, qty)`;
  - `set_updated_at()`.

- [ ] **Step 1: Write the failing test**

`supabase/tests/01_catalogue.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(10);

select has_table('public', 'menu_items', 'menu_items exists');
select has_table('public', 'modifiers', 'modifiers exists');
select has_table('public', 'menu_item_modifiers', 'menu_item_modifiers exists');
select has_table('public', 'ingredients', 'ingredients exists');
select has_table('public', 'recipe_lines', 'recipe_lines exists');
select has_table('public', 'modifier_lines', 'modifier_lines exists');

select throws_ok($$insert into public.menu_items (name, price_cents) values ('Latte', -1)$$,
  '23514', null, 'negative prices are rejected');
select throws_ok(
  $$insert into public.ingredients (name, kind, unit, bag_size_g, lead_time_days) values ('House', 'bean', 'g', 1000, 3)$$,
  '23514', null, 'a bean must say whether it is cold_brew or premium');

insert into public.menu_items (id, name, price_cents) values ('c0000000-0000-0000-0000-000000000001', 'Latte', 550);
insert into public.ingredients (id, name, kind, unit) values ('a0000000-0000-0000-0000-000000000001', 'Dairy milk', 'milk', 'ml');
select throws_ok(
  $$insert into public.recipe_lines (menu_item_id, ingredient_id, chosen_bean, qty)
    values ('c0000000-0000-0000-0000-000000000001', 'a0000000-0000-0000-0000-000000000001', true, 18)$$,
  '23514', null, 'a recipe line is either a fixed ingredient or the chosen bean, not both');
select lives_ok(
  $$insert into public.recipe_lines (menu_item_id, chosen_bean, qty)
    values ('c0000000-0000-0000-0000-000000000001', true, 18)$$,
  'a chosen-bean line needs no ingredient');

select * from finish();
rollback;
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm run db:test`
Expected: FAIL on "menu_items exists".

- [ ] **Step 3: Write the migration**

`supabase/migrations/20261012000100_catalogue.sql`:

```sql
create or replace function public.set_updated_at()
returns trigger language plpgsql set search_path = '' as $$
begin
  new.updated_at := now();
  return new;
end $$;

create table public.menu_items (
  id uuid primary key default gen_random_uuid(),
  name text not null check (length(trim(name)) between 1 and 60),
  description text not null default '' check (length(description) <= 280),
  category text not null default 'coffee' check (category in ('coffee', 'non_coffee', 'food')),
  price_cents integer not null check (price_cents >= 0),
  sold_on text not null default 'weekend_counter'
    check (sold_on in ('weekday_preorder', 'weekend_counter', 'both')),
  uses_bean_slot boolean not null default false,
  active boolean not null default true,
  sold_out boolean not null default false,
  sort_rank integer not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create unique index menu_items_active_name on public.menu_items (lower(name)) where active;
create trigger menu_items_updated_at before update on public.menu_items
  for each row execute function public.set_updated_at();

create table public.modifiers (
  id uuid primary key default gen_random_uuid(),
  name text not null unique check (length(trim(name)) between 1 and 40),
  group_name text not null default 'other' check (length(trim(group_name)) between 1 and 30),
  price_delta_cents integer not null default 0,
  active boolean not null default true
);

create table public.menu_item_modifiers (
  menu_item_id uuid not null references public.menu_items on delete cascade,
  modifier_id uuid not null references public.modifiers on delete cascade,
  primary key (menu_item_id, modifier_id)
);

create table public.ingredients (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  kind text not null check (kind in ('bean', 'milk', 'packaging', 'other')),
  unit text not null check (unit in ('g', 'ml', 'pc')),
  avg_cost_micros bigint not null default 0 check (avg_cost_micros >= 0),
  bean_use text check (bean_use in ('cold_brew', 'premium')),
  roaster text,
  bag_size_g integer check (bag_size_g > 0),
  lead_time_days integer check (lead_time_days >= 0),
  rest_days integer not null default 7 check (rest_days >= 0),
  waste_allowance_pct numeric(5,2) not null default 5 check (waste_allowance_pct between 0 and 100),
  active boolean not null default true,
  check (kind <> 'bean' or (bag_size_g is not null and lead_time_days is not null and bean_use is not null)),
  check (kind = 'bean' or bean_use is null)
);

create table public.recipe_lines (
  id uuid primary key default gen_random_uuid(),
  menu_item_id uuid not null references public.menu_items on delete cascade,
  ingredient_id uuid references public.ingredients on delete restrict,
  chosen_bean boolean not null default false,
  qty numeric(10,2) not null check (qty > 0),
  check ((ingredient_id is null) = chosen_bean)
);
create unique index recipe_lines_item_ingredient on public.recipe_lines (menu_item_id, ingredient_id)
  where ingredient_id is not null;
create unique index recipe_lines_item_chosen_bean on public.recipe_lines (menu_item_id) where chosen_bean;

create table public.modifier_lines (
  modifier_id uuid not null references public.modifiers on delete cascade,
  ingredient_id uuid not null references public.ingredients on delete restrict,
  qty numeric(10,2) not null check (qty <> 0),
  primary key (modifier_id, ingredient_id)
);
```

RLS arrives in Task 3. Do not deploy between Tasks 2 and 3.

- [ ] **Step 4: Run and verify the pass**

Run: `npm run db:reset && npm run db:test`
Expected: `01_catalogue.test.sql` passes 10 tests.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20261012000100_catalogue.sql supabase/tests/01_catalogue.test.sql
git commit -m "feat: add catalogue schema with milk modifiers and chosen-bean recipes"
```

---

### Task 3: Staff roles and catalogue RLS

**Files:**
- Create: `supabase/migrations/20261012000200_staff_rls.sql`
- Test: `supabase/tests/02_staff_rls.test.sql`

**Interfaces:**
- Consumes: the Task 2 tables.
- Produces:
  - `staff_roles(user_id, role in ('admin','staff'))`;
  - `public.is_staff()`, `public.is_admin()`.

  Later tasks protect tables with the same policy pattern.

- [ ] **Step 1: Write the failing test**

`supabase/tests/02_staff_rls.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(6);

insert into auth.users (id, email) values
  ('11111111-1111-1111-1111-111111111111', 'barista@test.local'),
  ('22222222-2222-2222-2222-222222222222', 'stranger@test.local');
insert into public.staff_roles (user_id, role) values ('11111111-1111-1111-1111-111111111111', 'staff');
insert into public.menu_items (name, price_cents) values ('Latte', 550);

set local role authenticated;
set local request.jwt.claims = '{"sub":"11111111-1111-1111-1111-111111111111","role":"authenticated"}';
select is((select count(*) from public.menu_items), 1::bigint, 'staff can read the menu');
select lives_ok($$insert into public.menu_items (name, price_cents) values ('Mocha', 650)$$, 'staff can add items');
select is((select role from public.staff_roles), 'staff', 'staff can read their own role');

set local request.jwt.claims = '{"sub":"22222222-2222-2222-2222-222222222222","role":"authenticated"}';
select is((select count(*) from public.menu_items), 0::bigint, 'non-staff see nothing');
select throws_ok($$insert into public.menu_items (name, price_cents) values ('Hack', 1)$$,
  '42501', null, 'non-staff cannot insert');

reset role;
set local role anon;
select is((select count(*) from public.menu_items), 0::bigint, 'anonymous users see nothing directly');

select * from finish();
rollback;
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm run db:test`
Expected: FAIL with `relation "public.staff_roles" does not exist`.

- [ ] **Step 3: Write the migration**

`supabase/migrations/20261012000200_staff_rls.sql`:

```sql
create table public.staff_roles (
  user_id uuid primary key references auth.users on delete cascade,
  role text not null check (role in ('admin', 'staff'))
);
alter table public.staff_roles enable row level security;
create policy "read own role" on public.staff_roles
  for select to authenticated using (user_id = (select auth.uid()));

create or replace function public.is_staff()
returns boolean language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.staff_roles where user_id = (select auth.uid()));
$$;

create or replace function public.is_admin()
returns boolean language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.staff_roles where user_id = (select auth.uid()) and role = 'admin');
$$;

revoke execute on function public.is_staff(), public.is_admin() from public, anon;
grant execute on function public.is_staff(), public.is_admin() to authenticated;

do $$
declare t text;
begin
  foreach t in array array['menu_items', 'modifiers', 'menu_item_modifiers', 'ingredients',
                           'recipe_lines', 'modifier_lines'] loop
    execute format('alter table public.%I enable row level security', t);
    execute format(
      'create policy "staff manage %1$s" on public.%1$I for all to authenticated
         using ((select public.is_staff())) with check ((select public.is_staff()))', t);
  end loop;
end $$;
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm run db:reset && npm run db:test`
Expected: all files pass; `02_staff_rls.test.sql` passes 6 tests.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20261012000200_staff_rls.sql supabase/tests/02_staff_rls.test.sql
git commit -m "feat: add staff roles and catalogue row-level security"
```

---

### Task 4: Bean cycles

**Files:**
- Create: `supabase/migrations/20261012000300_bean_cycles.sql`
- Test: `supabase/tests/03_bean_cycles.test.sql`

**Interfaces:**
- Consumes: `ingredients` (Task 2) and `is_staff()` (Task 3).
- Produces:
  - `public.sg_today() returns date`;
  - `bean_cycles(id, starts_on, reveal_on, ends_on)` (generated columns);
  - `cycle_beans(cycle_id, slot, ingredient_id, display_name, origin, tasting_notes text[], accepted_terms text[])`;
  - `public.cycle_on(p_day date) returns uuid` (cycle id or null);
  - `public.cycle_bean_id(p_slot text, p_day date) returns uuid` (bean ingredient id or null).

- [ ] **Step 1: Write the failing test**

`supabase/tests/03_bean_cycles.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(8);

insert into public.ingredients (id, name, kind, unit, bean_use, bag_size_g, lead_time_days) values
  ('a0000000-0000-0000-0000-0000000000a1', 'Ethiopia Guji', 'bean', 'g', 'premium', 250, 3),
  ('a0000000-0000-0000-0000-0000000000b1', 'Colombia Huila', 'bean', 'g', 'premium', 250, 3),
  ('a0000000-0000-0000-0000-0000000000c1', 'House cold brew', 'bean', 'g', 'cold_brew', 1000, 3);

select is(public.sg_today(), (now() at time zone 'Asia/Singapore')::date, 'sg_today uses Singapore time');

insert into public.bean_cycles (id, starts_on) values ('e0000000-0000-0000-0000-000000000001', date '2026-10-17');
select is((select reveal_on from public.bean_cycles), date '2026-10-24', 'reveal is start + 7');
select is((select ends_on from public.bean_cycles), date '2026-10-30', 'end is start + 13');

select throws_ok($$insert into public.bean_cycles (starts_on) values (date '2026-10-19')$$,
  '23514', null, 'cycles must start on a Saturday');
select throws_ok($$insert into public.bean_cycles (starts_on) values (date '2026-10-24')$$,
  '23P01', null, 'cycles cannot overlap');

select throws_ok(
  $$insert into public.cycle_beans (cycle_id, slot, ingredient_id, display_name)
    values ('e0000000-0000-0000-0000-000000000001', 'A', 'a0000000-0000-0000-0000-0000000000c1', 'x')$$,
  '23514', null, 'cold brew beans cannot be cycle beans');

insert into public.cycle_beans (cycle_id, slot, ingredient_id, display_name, tasting_notes, accepted_terms) values
  ('e0000000-0000-0000-0000-000000000001', 'A', 'a0000000-0000-0000-0000-0000000000a1', 'Guji Natural',
   '{blueberry,cacao,jasmine}', '{berry,chocolate,floral}'),
  ('e0000000-0000-0000-0000-000000000001', 'B', 'a0000000-0000-0000-0000-0000000000b1', 'Huila Washed',
   '{caramel,red apple}', '{toffee,apple}');

select is(public.cycle_bean_id('B', date '2026-10-20'),
  'a0000000-0000-0000-0000-0000000000b1'::uuid, 'slot B resolves inside the cycle');
select is(public.cycle_bean_id('A', date '2026-11-05'), null, 'no bean outside any cycle');

select * from finish();
rollback;
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm run db:test`
Expected: FAIL with `function public.sg_today() does not exist`.

- [ ] **Step 3: Write the migration**

`supabase/migrations/20261012000300_bean_cycles.sql`:

```sql
create or replace function public.sg_today()
returns date language sql stable set search_path = '' as $$
  select (now() at time zone 'Asia/Singapore')::date;
$$;

create table public.bean_cycles (
  id uuid primary key default gen_random_uuid(),
  starts_on date not null unique check (extract(isodow from starts_on) = 6),
  reveal_on date generated always as (starts_on + 7) stored,
  ends_on date generated always as (starts_on + 13) stored,
  created_at timestamptz not null default now(),
  constraint bean_cycles_no_overlap exclude using gist (daterange(starts_on, starts_on + 13, '[]') with &&)
);

create table public.cycle_beans (
  cycle_id uuid not null references public.bean_cycles on delete cascade,
  slot text not null check (slot in ('A', 'B')),
  ingredient_id uuid not null references public.ingredients,
  display_name text not null check (length(trim(display_name)) between 1 and 60),
  origin text not null default '',
  tasting_notes text[] not null default '{}',
  accepted_terms text[] not null default '{}',
  primary key (cycle_id, slot)
);

create or replace function public.check_cycle_bean()
returns trigger language plpgsql set search_path = '' as $$
begin
  if not exists (select 1 from public.ingredients
                 where id = new.ingredient_id and kind = 'bean' and bean_use = 'premium') then
    raise exception 'cycle beans must be premium beans' using errcode = '23514';
  end if;
  return new;
end $$;
create trigger cycle_beans_premium before insert or update on public.cycle_beans
  for each row execute function public.check_cycle_bean();

create or replace function public.cycle_on(p_day date)
returns uuid language sql stable set search_path = '' as $$
  select id from public.bean_cycles where p_day between starts_on and ends_on;
$$;

create or replace function public.cycle_bean_id(p_slot text, p_day date)
returns uuid language sql stable set search_path = '' as $$
  select cb.ingredient_id
  from public.cycle_beans cb join public.bean_cycles c on c.id = cb.cycle_id
  where cb.slot = p_slot and p_day between c.starts_on and c.ends_on;
$$;

do $$
declare t text;
begin
  foreach t in array array['bean_cycles', 'cycle_beans'] loop
    execute format('alter table public.%I enable row level security', t);
    execute format(
      'create policy "staff manage %1$s" on public.%1$I for all to authenticated
         using ((select public.is_staff())) with check ((select public.is_staff()))', t);
  end loop;
end $$;
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm run db:reset && npm run db:test`
Expected: `03_bean_cycles.test.sql` passes 8 tests.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20261012000300_bean_cycles.sql supabase/tests/03_bean_cycles.test.sql
git commit -m "feat: add 14-day bean cycles with saturday start and slot resolution"
```

---

### Task 5: Sales and `record_sale` (bean slots, modifiers, giveaways)

**Files:**
- Create: `supabase/migrations/20261012000400_sales.sql`
- Test: `supabase/tests/04_sales.test.sql`

**Interfaces:**
- Consumes: Tasks 2–4.
- Produces:
  - tables `customers`, `orders`, `order_items`, `stock_movements`;
  - `line_ingredient_usage(p_menu_item uuid, p_modifiers uuid[], p_bean uuid) returns table(ingredient_id uuid, qty numeric)`;
  - `line_unit_cost_micros(p_menu_item uuid, p_modifiers uuid[], p_bean uuid) returns bigint`;
  - `record_sale(p_order_id uuid, p_payment_method text, p_items jsonb) returns orders`.

  Item JSON: `{"menu_item_id": uuid, "qty": int, "modifier_ids": [uuid]?, "bean_slot": "A"|"B"?, "is_game_giveaway": bool?}`. `p_payment_method` is one of `paynow | cash | none`, where `none` is allowed only when the total is 0. Plan 2 (counter) calls `record_sale`; Plan 3 reuses the line functions for pre-orders.

- [ ] **Step 1: Write the failing test**

`supabase/tests/04_sales.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(14);

insert into auth.users (id, email) values
  ('11111111-1111-1111-1111-111111111111', 'barista@test.local'),
  ('22222222-2222-2222-2222-222222222222', 'stranger@test.local');
insert into public.staff_roles values ('11111111-1111-1111-1111-111111111111', 'staff');

insert into public.ingredients (id, name, kind, unit, avg_cost_micros, bean_use, bag_size_g, lead_time_days) values
  ('a0000000-0000-0000-0000-0000000000a1', 'Premium X', 'bean', 'g', 80000, 'premium', 250, 3),
  ('a0000000-0000-0000-0000-0000000000b1', 'Premium Y', 'bean', 'g', 60000, 'premium', 250, 3);
insert into public.ingredients (id, name, kind, unit, avg_cost_micros) values
  ('a0000000-0000-0000-0000-000000000002', 'Dairy milk', 'milk', 'ml', 3000),
  ('a0000000-0000-0000-0000-000000000003', 'Oat milk', 'milk', 'ml', 5000),
  ('a0000000-0000-0000-0000-000000000004', 'Cup', 'packaging', 'pc', 100000);

-- current cycle = the Saturday on or before today (so today is in the mystery week)
insert into public.bean_cycles (id, starts_on) values ('e0000000-0000-0000-0000-000000000001',
  public.sg_today() - ((extract(isodow from public.sg_today())::int + 1) % 7));
insert into public.cycle_beans (cycle_id, slot, ingredient_id, display_name) values
  ('e0000000-0000-0000-0000-000000000001', 'A', 'a0000000-0000-0000-0000-0000000000a1', 'X'),
  ('e0000000-0000-0000-0000-000000000001', 'B', 'a0000000-0000-0000-0000-0000000000b1', 'Y');

insert into public.menu_items (id, name, price_cents, sold_on, uses_bean_slot) values
  ('c0000000-0000-0000-0000-000000000001', 'Latte', 550, 'weekend_counter', true),
  ('c0000000-0000-0000-0000-000000000002', 'Cold brew white', 600, 'weekday_preorder', false);
insert into public.recipe_lines (menu_item_id, ingredient_id, chosen_bean, qty) values
  ('c0000000-0000-0000-0000-000000000001', null, true, 18),
  ('c0000000-0000-0000-0000-000000000001', 'a0000000-0000-0000-0000-000000000002', false, 180),
  ('c0000000-0000-0000-0000-000000000001', 'a0000000-0000-0000-0000-000000000004', false, 1);
insert into public.modifiers (id, name, group_name, price_delta_cents) values
  ('b0000000-0000-0000-0000-000000000001', 'Oat', 'milk', 100),
  ('b0000000-0000-0000-0000-000000000002', 'Extra shot', 'other', 100);
insert into public.menu_item_modifiers values
  ('c0000000-0000-0000-0000-000000000001', 'b0000000-0000-0000-0000-000000000001');
insert into public.modifier_lines values
  ('b0000000-0000-0000-0000-000000000001', 'a0000000-0000-0000-0000-000000000002', -180),
  ('b0000000-0000-0000-0000-000000000001', 'a0000000-0000-0000-0000-000000000003', 180);

-- cost maths: X 18 g x 80000 x 1.05 = 1512000; dairy 540000; cup 100000 => 2152000
select is(public.line_unit_cost_micros('c0000000-0000-0000-0000-000000000001', '{}',
  'a0000000-0000-0000-0000-0000000000a1'), 2152000::bigint, 'latte with bean X costs 2.152');

set local role authenticated;
set local request.jwt.claims = '{"sub":"11111111-1111-1111-1111-111111111111","role":"authenticated"}';

select is((public.record_sale('d0000000-0000-0000-0000-000000000001', 'paynow', '[
   {"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":2,"bean_slot":"A"},
   {"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1,"bean_slot":"B",
    "modifier_ids":["b0000000-0000-0000-0000-000000000001"]}]'::jsonb)).total_cents,
  1750, 'total = 2 x 5.50 + 1 x 6.50');

select is((public.record_sale('d0000000-0000-0000-0000-000000000001', 'cash', '[]'::jsonb)).total_cents,
  1750, 'same order id returns the original sale');
select is((select count(*) from public.orders), 1::bigint, 'only one order stored');

-- Y 18 x 60000 x 1.05 = 1134000; oat 900000; cup 100000 => 2134000
select results_eq(
  $$select bean_slot, bean_ingredient_id::text, unit_cost_micros from public.order_items order by bean_slot$$,
  $$values ('A', 'a0000000-0000-0000-0000-0000000000a1', 2152000::bigint),
           ('B', 'a0000000-0000-0000-0000-0000000000b1', 2134000::bigint)$$,
  'slots resolve to the cycle beans and costs are snapshotted');

select results_eq(
  $$select ingredient_id::text, sum(qty) from public.stock_movements group by 1 order by 1$$,
  $$values ('a0000000-0000-0000-0000-000000000002', -360.00::numeric),
           ('a0000000-0000-0000-0000-000000000003', -180.00::numeric),
           ('a0000000-0000-0000-0000-000000000004', -3.00::numeric),
           ('a0000000-0000-0000-0000-0000000000a1', -36.00::numeric),
           ('a0000000-0000-0000-0000-0000000000b1', -18.00::numeric)$$,
  'stock moves for resolved beans, milk swap and cups');

select is((public.record_sale('d0000000-0000-0000-0000-000000000002', 'none',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1,"bean_slot":"A","is_game_giveaway":true}]'::jsonb)).total_cents,
  0, 'a correct guess makes the drink free');
select results_eq(
  $$select unit_price_cents, unit_cost_micros, is_game_giveaway from public.order_items
    where order_id = 'd0000000-0000-0000-0000-000000000002'$$,
  $$values (0, 2152000::bigint, true)$$,
  'giveaways keep their cost');

select throws_ok($$select public.record_sale(gen_random_uuid(), 'cash',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1}]'::jsonb)$$,
  '22023', null, 'bean drinks need a slot');
select throws_ok($$select public.record_sale(gen_random_uuid(), 'cash',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1,"bean_slot":"A",
     "modifier_ids":["b0000000-0000-0000-0000-000000000002"]}]'::jsonb)$$,
  '22023', null, 'modifiers not allowed on the item are rejected');
select throws_ok($$select public.record_sale(gen_random_uuid(), 'cash',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000002","qty":1}]'::jsonb)$$,
  '22023', null, 'pre-order-only items cannot be sold at the counter');
select throws_ok($$select public.record_sale(gen_random_uuid(), 'none',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1,"bean_slot":"A"}]'::jsonb)$$,
  '22023', null, 'payment method none only for a zero total');

set local request.jwt.claims = '{"sub":"22222222-2222-2222-2222-222222222222","role":"authenticated"}';
select throws_ok($$select public.record_sale(gen_random_uuid(), 'cash',
  '[{"menu_item_id":"c0000000-0000-0000-0000-000000000001","qty":1,"bean_slot":"A"}]'::jsonb)$$,
  '42501', null, 'non-staff cannot record sales');

reset role;
select throws_ok($$update public.stock_movements set qty = 1$$, '42501', null, 'stock is append-only');

select * from finish();
rollback;
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm run db:test`
Expected: FAIL with `function public.line_unit_cost_micros(...) does not exist`.

- [ ] **Step 3: Write the migration**

`supabase/migrations/20261012000400_sales.sql`:

```sql
create table public.customers (
  id uuid primary key default gen_random_uuid(),
  name text not null default '' check (length(name) <= 80),
  mobile text not null unique check (mobile ~ '^[89][0-9]{7}$'),
  created_at timestamptz not null default now()
);

create table public.orders (
  id uuid primary key,
  code text unique,
  channel text not null check (channel in ('walk_in', 'pre_order')),
  status text not null check (status in
    ('submitted', 'verified', 'flagged', 'rejected', 'ready', 'collected', 'no_show', 'completed')),
  payment_method text not null check (payment_method in ('paynow', 'cash', 'none')),
  total_cents integer not null check (total_cents >= 0),
  customer_id uuid references public.customers,
  pickup_at timestamptz,
  proof_path text,
  proof_check jsonb,
  source text,
  created_at timestamptz not null default now(),
  check ((channel = 'walk_in' and status = 'completed') or (channel = 'pre_order' and status <> 'completed'))
);
create index orders_created_at on public.orders (created_at);

create table public.order_items (
  id uuid primary key default gen_random_uuid(),
  order_id uuid not null references public.orders on delete cascade,
  menu_item_id uuid not null references public.menu_items,
  qty integer not null check (qty between 1 and 50),
  modifier_ids uuid[] not null default '{}',
  bean_slot text check (bean_slot in ('A', 'B')),
  bean_ingredient_id uuid references public.ingredients,
  cycle_id uuid references public.bean_cycles,
  is_game_giveaway boolean not null default false,
  unit_price_cents integer not null check (unit_price_cents >= 0),
  unit_cost_micros bigint not null check (unit_cost_micros >= 0),
  check ((bean_slot is null) = (bean_ingredient_id is null)),
  check (not is_game_giveaway or (unit_price_cents = 0 and bean_slot is not null))
);
create index order_items_order on public.order_items (order_id);

create table public.stock_movements (
  id bigint generated always as identity primary key,
  ingredient_id uuid not null references public.ingredients,
  qty numeric(12,2) not null check (qty <> 0),
  reason text not null check (reason in ('purchase', 'sale', 'waste', 'count_adjust')),
  ref uuid,
  at timestamptz not null default now()
);
create index stock_movements_ingredient on public.stock_movements (ingredient_id, at);

create or replace function public.forbid_mutation()
returns trigger language plpgsql set search_path = '' as $$
begin
  raise exception '% is append-only', tg_table_name using errcode = '42501';
end $$;
create trigger stock_movements_append_only before update or delete on public.stock_movements
  for each row execute function public.forbid_mutation();

create or replace function public.line_ingredient_usage(p_menu_item uuid, p_modifiers uuid[], p_bean uuid)
returns table (ingredient_id uuid, qty numeric) language sql stable set search_path = '' as $$
  select l.ingredient_id, sum(l.qty)
  from (
    select coalesce(r.ingredient_id, p_bean) as ingredient_id, r.qty
    from public.recipe_lines r where r.menu_item_id = p_menu_item
    union all
    select m.ingredient_id, m.qty from public.modifier_lines m where m.modifier_id = any (p_modifiers)
  ) l
  where l.ingredient_id is not null
  group by l.ingredient_id
  having sum(l.qty) > 0;
$$;

create or replace function public.line_unit_cost_micros(p_menu_item uuid, p_modifiers uuid[], p_bean uuid)
returns bigint language sql stable set search_path = '' as $$
  select coalesce(round(sum(
    u.qty * i.avg_cost_micros * (case when i.kind = 'bean' then 1 + i.waste_allowance_pct / 100 else 1 end)
  ))::bigint, 0)
  from public.line_ingredient_usage(p_menu_item, p_modifiers, p_bean) u
  join public.ingredients i on i.id = u.ingredient_id;
$$;

-- one counter line: validates, inserts the item and its stock movements, returns the line total
create or replace function public.record_sale_line(p_order_id uuid, p_item jsonb, p_day date)
returns integer language plpgsql security definer set search_path = '' as $$
declare
  v_menu public.menu_items;
  v_qty integer := (p_item ->> 'qty')::integer;
  v_mods uuid[] := array(select (jsonb_array_elements_text(p_item -> 'modifier_ids'))::uuid);
  v_slot text := p_item ->> 'bean_slot';
  v_give boolean := coalesce((p_item ->> 'is_game_giveaway')::boolean, false);
  v_bean uuid;
  v_cycle uuid;
  v_price integer;
begin
  select * into v_menu from public.menu_items
    where id = (p_item ->> 'menu_item_id')::uuid and active and not sold_out
      and sold_on in ('weekend_counter', 'both');
  if not found then
    raise exception 'item % is not on the counter menu', p_item ->> 'menu_item_id' using errcode = '22023';
  end if;
  if v_qty is null or v_qty < 1 or v_qty > 50 then
    raise exception 'quantity must be between 1 and 50' using errcode = '22023';
  end if;
  if (select count(*) from public.menu_item_modifiers mim join public.modifiers m on m.id = mim.modifier_id
      where mim.menu_item_id = v_menu.id and m.active and m.id = any (v_mods)) <> cardinality(v_mods) then
    raise exception 'modifier not allowed on %', v_menu.name using errcode = '22023';
  end if;
  if exists (select 1 from public.recipe_lines where menu_item_id = v_menu.id and chosen_bean) then
    v_bean := public.cycle_bean_id(v_slot, p_day);
    v_cycle := public.cycle_on(p_day);
    if v_slot is null or v_bean is null then
      raise exception '% needs bean A or B from the current cycle', v_menu.name using errcode = '22023';
    end if;
  elsif v_slot is not null or v_give then
    raise exception '% does not use the A/B beans', v_menu.name using errcode = '22023';
  end if;
  if v_give and p_day >= (select reveal_on from public.bean_cycles where id = v_cycle) then
    raise exception 'giveaways are only for the mystery week' using errcode = '22023';
  end if;

  v_price := case when v_give then 0 else v_menu.price_cents
    + coalesce((select sum(price_delta_cents) from public.modifiers where id = any (v_mods)), 0) end;
  insert into public.order_items (order_id, menu_item_id, qty, modifier_ids, bean_slot, bean_ingredient_id,
                                  cycle_id, is_game_giveaway, unit_price_cents, unit_cost_micros)
  values (p_order_id, v_menu.id, v_qty, v_mods, v_slot, v_bean, v_cycle, v_give, v_price,
          public.line_unit_cost_micros(v_menu.id, v_mods, v_bean));
  insert into public.stock_movements (ingredient_id, qty, reason, ref)
  select u.ingredient_id, -(u.qty * v_qty), 'sale', p_order_id
  from public.line_ingredient_usage(v_menu.id, v_mods, v_bean) u;
  return v_price * v_qty;
end $$;
revoke execute on function public.record_sale_line(uuid, jsonb, date) from public, anon, authenticated;

create or replace function public.record_sale(p_order_id uuid, p_payment_method text, p_items jsonb)
returns public.orders language plpgsql security definer set search_path = '' as $$
declare
  v_order public.orders;
  v_item jsonb;
  v_total integer := 0;
  v_day date := public.sg_today();
begin
  if not public.is_staff() then
    raise exception 'not authorised' using errcode = '42501';
  end if;
  select * into v_order from public.orders where id = p_order_id;
  if found then
    return v_order;
  end if;
  if p_payment_method not in ('paynow', 'cash', 'none') then
    raise exception 'payment method must be paynow, cash or none' using errcode = '22023';
  end if;
  if jsonb_typeof(p_items) <> 'array' or jsonb_array_length(p_items) = 0 then
    raise exception 'a sale needs at least one item' using errcode = '22023';
  end if;

  insert into public.orders (id, channel, status, payment_method, total_cents)
  values (p_order_id, 'walk_in', 'completed', p_payment_method, 0);
  for v_item in select * from jsonb_array_elements(p_items) loop
    v_total := v_total + public.record_sale_line(p_order_id, v_item, v_day);
  end loop;
  if (p_payment_method = 'none') <> (v_total = 0) then
    raise exception 'use payment method none only when the total is zero' using errcode = '22023';
  end if;

  update public.orders set total_cents = v_total where id = p_order_id returning * into v_order;
  return v_order;
end $$;
revoke execute on function public.record_sale(uuid, text, jsonb) from public, anon;
grant execute on function public.record_sale(uuid, text, jsonb) to authenticated;

do $$
declare t text;
begin
  foreach t in array array['customers', 'orders', 'order_items', 'stock_movements'] loop
    execute format('alter table public.%I enable row level security', t);
    execute format('create policy "staff read %1$s" on public.%1$I for select to authenticated
                      using ((select public.is_staff()))', t);
  end loop;
end $$;
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm run db:reset && npm run db:test`
Expected: `04_sales.test.sql` passes 14 tests.
If the idempotency test fails, check that the existing-order lookup runs before payload validation. If the giveaway test fails, check that the test cycle starts on the Saturday on or before today.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20261012000400_sales.sql supabase/tests/04_sales.test.sql
git commit -m "feat: add sales with bean-slot resolution, milk modifiers and game giveaways"
```

---

### Task 6: Settings, website content, job audit and `site_snapshot()`

**Files:**
- Create: `supabase/migrations/20261012000500_site_settings.sql`
- Test: `supabase/tests/05_site_settings.test.sql`

**Interfaces:**
- Consumes: Tasks 2–4.
- Produces:
  - `settings(key, value jsonb)`, seeded with `telegram_allowed_chat_ids` = `[]` and `paynow` = `{"name":"","mobile":""}`;
  - `site_content(key in home|about|info|announcement, title, body, image_path, published)`;
  - `job_runs`;
  - `public.site_snapshot() returns jsonb`, the only public read.

  Shape of `site_snapshot()`:
  `{ today, content: {key:{title,body,image_path}}, menu: [{id,name,description,category,price_cents,sold_on,sold_out,uses_bean_slot,modifiers:[{id,name,group,price_delta_cents}]}], cycle: null | {starts_on,reveal_on,ends_on,revealed,beans:[{slot,name,origin?,tasting_notes?}]} }`.

- [ ] **Step 1: Write the failing test**

`supabase/tests/05_site_settings.test.sql`:

```sql
begin;
create extension if not exists pgtap with schema extensions;
select plan(10);

insert into auth.users (id, email) values
  ('11111111-1111-1111-1111-111111111111', 'barista@test.local'),
  ('33333333-3333-3333-3333-333333333333', 'owner@test.local');
insert into public.staff_roles values
  ('11111111-1111-1111-1111-111111111111', 'staff'),
  ('33333333-3333-3333-3333-333333333333', 'admin');

insert into public.ingredients (id, name, kind, unit, bean_use, bag_size_g, lead_time_days) values
  ('a0000000-0000-0000-0000-0000000000a1', 'Guji', 'bean', 'g', 'premium', 250, 3);
insert into public.menu_items (name, price_cents, sold_on) values
  ('Cold brew black', 500, 'weekday_preorder'), ('Old item', 400, 'both');
update public.menu_items set active = false where name = 'Old item';
update public.site_content set title = 'About us', body = 'Two friends, one balcony.', published = true where key = 'about';
update public.site_content set title = 'Draft', body = 'secret draft' where key = 'announcement';

-- mystery week: cycle started on the Saturday on or before today
insert into public.bean_cycles (id, starts_on) values ('e0000000-0000-0000-0000-000000000001',
  public.sg_today() - ((extract(isodow from public.sg_today())::int + 1) % 7));
insert into public.cycle_beans (cycle_id, slot, ingredient_id, display_name, origin, tasting_notes, accepted_terms) values
  ('e0000000-0000-0000-0000-000000000001', 'A', 'a0000000-0000-0000-0000-0000000000a1', 'Guji Natural', 'Ethiopia',
   '{blueberry,cacao}', '{berry,chocolate}');

set local role anon;
select is(jsonb_array_length(public.site_snapshot() -> 'menu'), 1, 'only active items are public');
select is(public.site_snapshot() #>> '{content,about,body}', 'Two friends, one balcony.', 'published content is public');
select ok(public.site_snapshot() #> '{content,announcement}' is null, 'unpublished content is hidden');
select is(public.site_snapshot() #>> '{cycle,beans,0,name}', 'Mystery Bean A', 'mystery week hides the name');
select ok(position('Guji' in public.site_snapshot()::text) = 0 and position('blueberry' in public.site_snapshot()::text) = 0,
  'mystery week leaks no name, origin or notes');
select is((select count(*) from public.site_content), 0::bigint, 'anon cannot read tables directly (RLS returns nothing)');

reset role;
update public.bean_cycles set starts_on = starts_on - 7 where id = 'e0000000-0000-0000-0000-000000000001';
set local role anon;
select is(public.site_snapshot() #>> '{cycle,beans,0,name}', 'Guji Natural', 'revealed week shows the name');
select ok(position('chocolate' in public.site_snapshot()::text) = 0, 'accepted terms are never public, even after reveal');

reset role;
set local role authenticated;
set local request.jwt.claims = '{"sub":"11111111-1111-1111-1111-111111111111","role":"authenticated"}';
select is((with u as (update public.settings set value = '[1]' where key = 'telegram_allowed_chat_ids' returning 1)
  select count(*) from u), 0::bigint, 'staff cannot change settings');
set local request.jwt.claims = '{"sub":"33333333-3333-3333-3333-333333333333","role":"authenticated"}';
select is((with u as (update public.settings set value = '[1]' where key = 'telegram_allowed_chat_ids' returning 1)
  select count(*) from u), 1::bigint, 'admin can change settings');

select * from finish();
rollback;
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm run db:test`
Expected: FAIL with `relation "public.site_content" does not exist`.

- [ ] **Step 3: Write the migration**

`supabase/migrations/20261012000500_site_settings.sql`:

```sql
create table public.settings (
  key text primary key,
  value jsonb not null,
  updated_at timestamptz not null default now()
);
create trigger settings_updated_at before update on public.settings
  for each row execute function public.set_updated_at();
insert into public.settings (key, value) values
  ('telegram_allowed_chat_ids', '[]'), ('paynow', '{"name": "", "mobile": ""}')
on conflict (key) do nothing;
alter table public.settings enable row level security;
create policy "staff read settings" on public.settings for select to authenticated using ((select public.is_staff()));
create policy "admin writes settings" on public.settings for all to authenticated
  using ((select public.is_admin())) with check ((select public.is_admin()));

create table public.site_content (
  key text primary key check (key in ('home', 'about', 'info', 'announcement')),
  title text not null default '' check (length(title) <= 120),
  body text not null default '' check (length(body) <= 5000),
  image_path text,
  published boolean not null default false,
  updated_at timestamptz not null default now()
);
create trigger site_content_updated_at before update on public.site_content
  for each row execute function public.set_updated_at();
insert into public.site_content (key) values ('home'), ('about'), ('info'), ('announcement')
on conflict (key) do nothing;
alter table public.site_content enable row level security;
create policy "staff manage site_content" on public.site_content for all to authenticated
  using ((select public.is_staff())) with check ((select public.is_staff()));

create table public.job_runs (
  id bigint generated always as identity primary key,
  job text not null,
  started_at timestamptz not null default now(),
  finished_at timestamptz,
  ok boolean,
  detail jsonb
);
create index job_runs_job_started on public.job_runs (job, started_at desc);
alter table public.job_runs enable row level security;
create policy "admin reads job runs" on public.job_runs for select to authenticated using ((select public.is_admin()));

create or replace function public.site_menu()
returns jsonb language sql stable security definer set search_path = '' as $$
  select coalesce(jsonb_agg(jsonb_build_object(
    'id', m.id, 'name', m.name, 'description', m.description, 'category', m.category,
    'price_cents', m.price_cents, 'sold_on', m.sold_on, 'sold_out', m.sold_out, 'uses_bean_slot', m.uses_bean_slot,
    'modifiers', coalesce((
      select jsonb_agg(jsonb_build_object('id', mo.id, 'name', mo.name, 'group', mo.group_name,
                                          'price_delta_cents', mo.price_delta_cents) order by mo.name)
      from public.menu_item_modifiers mim join public.modifiers mo on mo.id = mim.modifier_id
      where mim.menu_item_id = m.id and mo.active), '[]'::jsonb)
  ) order by m.sort_rank, m.name), '[]'::jsonb)
  from public.menu_items m where m.active;
$$;

create or replace function public.site_cycle(p_day date)
returns jsonb language sql stable security definer set search_path = '' as $$
  select jsonb_build_object(
    'starts_on', c.starts_on, 'reveal_on', c.reveal_on, 'ends_on', c.ends_on, 'revealed', p_day >= c.reveal_on,
    'beans', coalesce((
      select jsonb_agg(case when p_day >= c.reveal_on
        then jsonb_build_object('slot', cb.slot, 'name', cb.display_name, 'origin', cb.origin,
                                'tasting_notes', to_jsonb(cb.tasting_notes))
        else jsonb_build_object('slot', cb.slot, 'name', 'Mystery Bean ' || cb.slot) end order by cb.slot)
      from public.cycle_beans cb where cb.cycle_id = c.id), '[]'::jsonb))
  from public.bean_cycles c where p_day between c.starts_on and c.ends_on;
$$;

create or replace function public.site_snapshot()
returns jsonb language sql stable security definer set search_path = '' as $$
  select jsonb_build_object(
    'today', public.sg_today(),
    'content', coalesce((select jsonb_object_agg(key, jsonb_build_object('title', title, 'body', body,
                           'image_path', image_path)) from public.site_content where published), '{}'::jsonb),
    'menu', public.site_menu(),
    'cycle', public.site_cycle(public.sg_today()));
$$;

revoke execute on function public.site_menu(), public.site_cycle(date) from public, anon, authenticated;
revoke execute on function public.site_snapshot() from public;
grant execute on function public.site_snapshot() to anon, authenticated;
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm run db:reset && npm run db:test`
Expected: all 6 test files pass; `05_site_settings.test.sql` passes 10 tests.

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20261012000500_site_settings.sql supabase/tests/05_site_settings.test.sql
git commit -m "feat: add settings, website content, job audit and embargoed site_snapshot"
```

---

### Task 7: Telegram webhook skeleton

**Files:**
- Create: `supabase/functions/_shared/telegram.ts`, `supabase/functions/telegram-webhook/index.ts`
- Test: `tests/functions/telegram.test.ts`

**Interfaces:**
- Consumes: `settings.telegram_allowed_chat_ids` (Task 6).
- Produces:
  - `verifySecret(header: string | null, expected: string): boolean`;
  - `replyFor(update: TelegramUpdate, allowedChatIds: readonly number[]): { chatId: number; text: string } | null`;
  - `type TelegramUpdate`.

- [ ] **Step 1: Write the failing test**

`tests/functions/telegram.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { replyFor, verifySecret, type TelegramUpdate } from '../../supabase/functions/_shared/telegram.ts';

const message = (chatId: number, text: string): TelegramUpdate => ({
  update_id: 1,
  message: { chat: { id: chatId }, from: { first_name: 'Mei' }, text },
});

describe('verifySecret', () => {
  it('accepts the exact secret', () => expect(verifySecret('s3cret', 's3cret')).toBe(true));
  it('rejects a different secret', () => expect(verifySecret('s3creT', 's3cret')).toBe(false));
  it('rejects a missing header', () => expect(verifySecret(null, 's3cret')).toBe(false));
  it('rejects when no secret is configured', () => expect(verifySecret('', '')).toBe(false));
});

describe('replyFor', () => {
  it('tells an unknown chat its id so the owner can allow it', () => {
    const reply = replyFor(message(555, '/start'), [111]);
    expect(reply?.chatId).toBe(555);
    expect(reply?.text).toContain('555');
    expect(reply?.text).toContain('not set up yet');
  });
  it('greets an allowed chat by name', () => {
    expect(replyFor(message(111, '/start'), [111])).toEqual({
      chatId: 111, text: 'Hi Mei, the café bot is connected. You will get orders and alerts here.',
    });
  });
  it('ignores other text for now', () => expect(replyFor(message(111, 'hello'), [111])).toBeNull());
  it('ignores updates without a message', () => expect(replyFor({ update_id: 2 }, [111])).toBeNull());
});
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm test`
Expected: FAIL with `Failed to resolve import "../../supabase/functions/_shared/telegram.ts"`.

- [ ] **Step 3: Implement the pure module**

`supabase/functions/_shared/telegram.ts`:

```ts
export type TelegramUpdate = {
  update_id: number;
  message?: { chat: { id: number }; from?: { first_name?: string }; text?: string };
};

export type Reply = { chatId: number; text: string };

export function verifySecret(header: string | null, expected: string): boolean {
  if (!header || !expected || header.length !== expected.length) return false;
  let diff = 0;
  for (let i = 0; i < expected.length; i += 1) diff |= header.charCodeAt(i) ^ expected.charCodeAt(i);
  return diff === 0;
}

export function replyFor(update: TelegramUpdate, allowedChatIds: readonly number[]): Reply | null {
  const msg = update.message;
  if (!msg?.text) return null;
  const chatId = msg.chat.id;
  const isStart = msg.text.trim() === '/start';
  if (!allowedChatIds.includes(chatId)) {
    return isStart ? { chatId, text: `This chat is not set up yet. Send this chat ID to the owner: ${chatId}` } : null;
  }
  if (!isStart) return null;
  return { chatId, text: `Hi ${msg.from?.first_name ?? 'there'}, the café bot is connected. You will get orders and alerts here.` };
}
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm test && npm run typecheck`
Expected: the 8 Telegram tests pass, along with the shared tests; the typecheck is clean.

- [ ] **Step 5: Deno handler**

`supabase/functions/telegram-webhook/index.ts`:

```ts
import { createClient } from 'npm:@supabase/supabase-js@2';
import { replyFor, verifySecret, type TelegramUpdate } from '../_shared/telegram.ts';

const BOT_TOKEN = Deno.env.get('TELEGRAM_BOT_TOKEN') ?? '';
const WEBHOOK_SECRET = Deno.env.get('TELEGRAM_WEBHOOK_SECRET') ?? '';
const supabase = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);

async function allowedChatIds(): Promise<number[]> {
  const { data, error } = await supabase.from('settings').select('value').eq('key', 'telegram_allowed_chat_ids').single();
  if (error) throw new Error(`could not load Telegram allowlist: ${error.message}`);
  return Array.isArray(data.value) ? data.value.map(Number) : [];
}

async function send(chatId: number, text: string): Promise<void> {
  const res = await fetch(`https://api.telegram.org/bot${BOT_TOKEN}/sendMessage`, {
    method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ chat_id: chatId, text }),
  });
  if (!res.ok) throw new Error(`Telegram sendMessage failed: ${res.status} ${await res.text()}`);
}

Deno.serve(async (req) => {
  if (!verifySecret(req.headers.get('x-telegram-bot-api-secret-token'), WEBHOOK_SECRET)) {
    return new Response('unauthorised', { status: 401 });
  }
  try {
    const reply = replyFor((await req.json()) as TelegramUpdate, await allowedChatIds());
    if (reply) await send(reply.chatId, reply.text);
    return new Response('ok');
  } catch (err) {
    console.error('telegram-webhook failed', err);
    return new Response('error', { status: 500 });
  }
});
```

- [ ] **Step 6: Local smoke test**

Create a git-ignored file `supabase/functions/.env.local` with `TELEGRAM_BOT_TOKEN=test` and `TELEGRAM_WEBHOOK_SECRET=local-secret`. Then:

```bash
npx supabase functions serve telegram-webhook --no-verify-jwt --env-file supabase/functions/.env.local
curl -i -X POST http://127.0.0.1:54321/functions/v1/telegram-webhook -H 'content-type: application/json' -d '{"update_id":1}'
curl -i -X POST http://127.0.0.1:54321/functions/v1/telegram-webhook -H 'x-telegram-bot-api-secret-token: local-secret' -H 'content-type: application/json' -d '{"update_id":1}'
```

Expected: `401`, then `200 ok`.

- [ ] **Step 7: Commit**

```bash
git add supabase/functions/_shared/telegram.ts supabase/functions/telegram-webhook/index.ts tests/functions/telegram.test.ts
git commit -m "feat: add telegram webhook skeleton with secret check and allowlist"
```

---

### Task 8: Staff web app with login (`apps/staff`)

**Files:**
- Create: `apps/staff/` (Vite React TS template), `apps/staff/src/lib/supabase.ts`, `apps/staff/src/auth/AuthContext.tsx`, `apps/staff/src/auth/RequireStaff.tsx`, `apps/staff/src/auth/LoginPage.tsx`, `apps/staff/src/test/setup.ts`, `apps/staff/public/_redirects`
- Modify: `apps/staff/vite.config.ts`, `apps/staff/src/App.tsx`, `apps/staff/src/main.tsx`, root `package.json`, `.github/workflows/ci.yml`
- Test: `apps/staff/src/auth/RequireStaff.test.tsx`, `apps/staff/src/auth/LoginPage.test.tsx`

**Interfaces:**
- Consumes: the `staff_roles` "read own role" policy.
- Produces:
  - `AuthContext`, `useAuth(): AuthState`, `<RequireStaff adminOnly?>`, `supabase`;
  - `AuthState = {status:'loading'} | {status:'signedOut'} | {status:'signedIn'; email; role:'admin'|'staff'|null}`.

- [ ] **Step 1: Scaffold**

```bash
npm create vite@latest apps/staff -- --template react-ts
npm pkg set 'workspaces[1]=apps/*'
npm pkg set --workspace apps/staff 'dependencies.@cafe/shared=*'
npm install --workspace apps/staff @supabase/supabase-js react-router-dom zod
npm install --workspace apps/staff -D vitest jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
npm pkg set --workspace apps/staff scripts.test="vitest run"
npm pkg set scripts.test="vitest run && npm test --workspaces --if-present"
printf '/* /index.html 200\n' > apps/staff/public/_redirects
```

Append to `.github/workflows/ci.yml`: `      - run: npm run build --workspace apps/staff`.

`apps/staff/vite.config.ts`:

```ts
/// <reference types="vitest/config" />
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: { environment: 'jsdom', setupFiles: './src/test/setup.ts' },
});
```

`apps/staff/src/test/setup.ts`:

```ts
import '@testing-library/jest-dom/vitest';
```

`apps/staff/src/lib/supabase.ts`:

```ts
import { createClient } from '@supabase/supabase-js';

const url = import.meta.env.VITE_SUPABASE_URL;
const anonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
if (!url || !anonKey) throw new Error('Missing VITE_SUPABASE_URL or VITE_SUPABASE_ANON_KEY. Copy .env.example to apps/staff/.env.local.');

export const supabase = createClient(url, anonKey);
```

- [ ] **Step 2: Write the failing tests**

`apps/staff/src/auth/RequireStaff.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { describe, expect, it } from 'vitest';
import { AuthContext, type AuthState } from './AuthContext';
import { RequireStaff } from './RequireStaff';

function renderAt(state: AuthState, adminOnly = false) {
  return render(
    <AuthContext.Provider value={state}>
      <MemoryRouter initialEntries={['/menu']}>
        <Routes>
          <Route path="/login" element={<p>login page</p>} />
          <Route path="/menu" element={<RequireStaff adminOnly={adminOnly}><p>secret</p></RequireStaff>} />
        </Routes>
      </MemoryRouter>
    </AuthContext.Provider>,
  );
}

describe('RequireStaff', () => {
  it('redirects signed-out users to login', () => {
    renderAt({ status: 'signedOut' });
    expect(screen.getByText('login page')).toBeInTheDocument();
  });
  it('shows the page to staff', () => {
    renderAt({ status: 'signedIn', email: 'b@x.sg', role: 'staff' });
    expect(screen.getByText('secret')).toBeInTheDocument();
  });
  it('explains when an account has no staff role', () => {
    renderAt({ status: 'signedIn', email: 'b@x.sg', role: null });
    expect(screen.getByRole('alert')).toHaveTextContent('not set up as staff');
  });
  it('blocks staff from owner-only pages', () => {
    renderAt({ status: 'signedIn', email: 'b@x.sg', role: 'staff' }, true);
    expect(screen.getByRole('alert')).toHaveTextContent('Only the owner');
  });
});
```

`apps/staff/src/auth/LoginPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { LoginPage } from './LoginPage';

describe('LoginPage', () => {
  it('signs in with the typed email and password', async () => {
    const signIn = vi.fn().mockResolvedValue(null);
    render(<LoginPage signIn={signIn} />);
    await userEvent.type(screen.getByLabelText('Email'), 'barista@cafe.sg');
    await userEvent.type(screen.getByLabelText('Password'), 'pw');
    await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));
    expect(signIn).toHaveBeenCalledWith('barista@cafe.sg', 'pw');
  });
  it('shows the error when sign-in fails', async () => {
    render(<LoginPage signIn={vi.fn().mockResolvedValue('Wrong email or password.')} />);
    await userEvent.type(screen.getByLabelText('Email'), 'a@b.sg');
    await userEvent.type(screen.getByLabelText('Password'), 'x');
    await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));
    expect(await screen.findByRole('alert')).toHaveTextContent('Wrong email or password.');
  });
});
```

- [ ] **Step 3: Run and verify the failure**

Run: `npm test --workspace apps/staff`
Expected: FAIL with `Failed to resolve import "./AuthContext"`.

- [ ] **Step 4: Implement**

`apps/staff/src/auth/AuthContext.tsx`:

```tsx
import { createContext, useContext, useEffect, useState, type ReactNode } from 'react';
import type { Session, SupabaseClient } from '@supabase/supabase-js';

export type StaffRole = 'admin' | 'staff';
export type AuthState =
  | { status: 'loading' }
  | { status: 'signedOut' }
  | { status: 'signedIn'; email: string; role: StaffRole | null };

export const AuthContext = createContext<AuthState>({ status: 'loading' });
export const useAuth = (): AuthState => useContext(AuthContext);

async function loadState(client: SupabaseClient, session: Session | null): Promise<AuthState> {
  if (!session) return { status: 'signedOut' };
  const { data, error } = await client.from('staff_roles').select('role').eq('user_id', session.user.id).maybeSingle();
  const role = !error && (data?.role === 'admin' || data?.role === 'staff') ? data.role : null;
  return { status: 'signedIn', email: session.user.email ?? '', role };
}

export function AuthProvider({ client, children }: { client: SupabaseClient; children: ReactNode }) {
  const [state, setState] = useState<AuthState>({ status: 'loading' });
  useEffect(() => {
    let cancelled = false;
    const apply = (session: Session | null) =>
      loadState(client, session).then((next) => { if (!cancelled) setState(next); });
    client.auth.getSession().then(({ data }) => apply(data.session));
    const { data: sub } = client.auth.onAuthStateChange((_event, session) => { void apply(session); });
    return () => { cancelled = true; sub.subscription.unsubscribe(); };
  }, [client]);
  return <AuthContext.Provider value={state}>{children}</AuthContext.Provider>;
}
```

`apps/staff/src/auth/RequireStaff.tsx`:

```tsx
import type { ReactNode } from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from './AuthContext';

export function RequireStaff({ children, adminOnly = false }: { children: ReactNode; adminOnly?: boolean }) {
  const auth = useAuth();
  if (auth.status === 'loading') return <p>Loading…</p>;
  if (auth.status === 'signedOut') return <Navigate to="/login" replace />;
  if (auth.role === null) return <p role="alert">This account is not set up as staff. Ask the owner to add it.</p>;
  if (adminOnly && auth.role !== 'admin') return <p role="alert">Only the owner can open this page.</p>;
  return <>{children}</>;
}
```

`apps/staff/src/auth/LoginPage.tsx`:

```tsx
import { useState, type FormEvent } from 'react';

type Props = { signIn: (email: string, password: string) => Promise<string | null> };

export function LoginPage({ signIn }: Props) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    setBusy(true);
    setError(await signIn(email.trim(), password));
    setBusy(false);
  }

  return (
    <form onSubmit={onSubmit} aria-label="Sign in">
      <h1>Café staff sign-in</h1>
      <label>Email<input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required /></label>
      <label>Password<input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required /></label>
      {error && <p role="alert">{error}</p>}
      <button type="submit" disabled={busy}>Sign in</button>
    </form>
  );
}
```

- [ ] **Step 5: Run and verify the pass**

Run: `npm test --workspace apps/staff`
Expected: 6 tests pass.

- [ ] **Step 6: Routes**

`apps/staff/src/App.tsx`:

```tsx
import { BrowserRouter, Navigate, Route, Routes, useNavigate } from 'react-router-dom';
import { AuthProvider } from './auth/AuthContext';
import { LoginPage } from './auth/LoginPage';
import { RequireStaff } from './auth/RequireStaff';
import { supabase } from './lib/supabase';

function LoginRoute() {
  const navigate = useNavigate();
  return (
    <LoginPage signIn={async (email, password) => {
      const { error } = await supabase.auth.signInWithPassword({ email, password });
      if (error) return 'Wrong email or password.';
      navigate('/menu');
      return null;
    }} />
  );
}

export default function App() {
  return (
    <AuthProvider client={supabase}>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<LoginRoute />} />
          <Route path="/menu" element={<RequireStaff><p>Menu editor comes in Task 9.</p></RequireStaff>} />
          <Route path="*" element={<Navigate to="/menu" replace />} />
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}
```

Remove the template's demo CSS and assets. `main.tsx` renders `<App />` in `StrictMode`.

- [ ] **Step 7: Manual check (local)**

1. Copy the API URL and anon key from `npx supabase status` into `apps/staff/.env.local`.
2. Create a user in local Studio, then run `insert into public.staff_roles values ('<uuid>', 'admin');`.
3. Run `npm run dev --workspace apps/staff` and sign in.

Expected: you land on `/menu` and see the placeholder.

- [ ] **Step 8: Commit**

```bash
git add apps/staff package.json package-lock.json .github/workflows/ci.yml
git commit -m "feat: add staff web app with login and role guard"
```

---

### Task 9: Staff menu editor

**Files:**
- Create: `apps/staff/src/features/menu/menuForm.ts`, `menuApi.ts`, `MenuItemFormView.tsx`, `MenuAdminPage.tsx` (all in `apps/staff/src/features/menu/`)
- Modify: `apps/staff/src/App.tsx`
- Test: `apps/staff/src/features/menu/menuForm.test.ts`, `apps/staff/src/features/menu/MenuAdminPage.test.tsx`

**Interfaces:**
- Consumes: the `menu_items` columns (Task 2), `@cafe/shared` money helpers, `supabase`.
- Produces:
  - `menuItemFormSchema`, with `MenuItemInput = { name; description; category; price /*cents*/; soldOn: 'weekday_preorder'|'weekend_counter'|'both'; usesBeanSlot: boolean }`;
  - `MenuItemRow`;
  - `MenuApi { list; create; update; setActive; setSoldOut }`;
  - `createMenuApi(client)`.

  Plan 4 adds modifier editing.

- [ ] **Step 1: Write the failing form test**

`apps/staff/src/features/menu/menuForm.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { menuItemFormSchema } from './menuForm';

const base = { name: ' Cold brew white ', description: '', category: 'coffee', price: '6', soldOn: 'weekday_preorder', usesBeanSlot: false };

describe('menuItemFormSchema', () => {
  it('turns a valid form into cents', () => {
    expect(menuItemFormSchema.parse(base)).toEqual({
      name: 'Cold brew white', description: '', category: 'coffee', price: 600, soldOn: 'weekday_preorder', usesBeanSlot: false,
    });
  });
  it('explains a bad price', () => {
    expect(menuItemFormSchema.safeParse({ ...base, price: 'six' }).error?.issues[0]?.message).toBe('Enter a price like 5.50');
  });
  it('requires a name', () => {
    expect(menuItemFormSchema.safeParse({ ...base, name: ' ' }).error?.issues[0]?.message).toBe('Enter a name');
  });
  it('rejects A/B beans on a pre-order-only cold brew item', () => {
    expect(menuItemFormSchema.safeParse({ ...base, usesBeanSlot: true }).error?.issues[0]?.message)
      .toBe('A/B beans are for weekend drinks only');
  });
});
```

- [ ] **Step 2: Run and verify the failure**

Run: `npm test --workspace apps/staff`
Expected: FAIL with `Failed to resolve import "./menuForm"`.

- [ ] **Step 3: Implement `menuForm.ts`**

```ts
import { z } from 'zod';
import { parsePriceToCents } from '@cafe/shared';

export const CATEGORIES = ['coffee', 'non_coffee', 'food'] as const;
export const SOLD_ON = ['weekday_preorder', 'weekend_counter', 'both'] as const;
export const SOLD_ON_LABEL: Record<(typeof SOLD_ON)[number], string> = {
  weekday_preorder: 'Weekday pre-order', weekend_counter: 'Weekend counter', both: 'Both',
};

export const menuItemFormSchema = z.object({
  name: z.string().trim().min(1, 'Enter a name').max(60, 'Keep the name under 60 characters'),
  description: z.string().trim().max(280, 'Keep the description under 280 characters'),
  category: z.enum(CATEGORIES),
  price: z.string().transform((value, ctx) => {
    const cents = parsePriceToCents(value);
    if (cents === null) {
      ctx.addIssue({ code: 'custom', message: 'Enter a price like 5.50' });
      return z.NEVER;
    }
    return cents;
  }),
  soldOn: z.enum(SOLD_ON),
  usesBeanSlot: z.boolean(),
}).refine((v) => !(v.usesBeanSlot && v.soldOn === 'weekday_preorder'), {
  message: 'A/B beans are for weekend drinks only', path: ['usesBeanSlot'],
});

export type MenuItemFormValues = z.input<typeof menuItemFormSchema>;
export type MenuItemInput = z.output<typeof menuItemFormSchema>;
```

- [ ] **Step 4: Run and verify the pass**

Run: `npm test --workspace apps/staff`
Expected: the 4 menuForm tests pass.

- [ ] **Step 5: Write the failing page test**

`apps/staff/src/features/menu/MenuAdminPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import type { MenuApi, MenuItemRow } from './menuApi';
import { MenuAdminPage } from './MenuAdminPage';

const coldBrew: MenuItemRow = {
  id: 'm1', name: 'Cold brew black', description: '', category: 'coffee', price_cents: 500,
  sold_on: 'weekday_preorder', uses_bean_slot: false, active: true, sold_out: false, sort_rank: 0,
};

const fakeApi = (rows: MenuItemRow[] = [coldBrew]): MenuApi => ({
  list: vi.fn().mockResolvedValue(rows),
  create: vi.fn().mockResolvedValue(undefined),
  update: vi.fn().mockResolvedValue(undefined),
  setActive: vi.fn().mockResolvedValue(undefined),
  setSoldOut: vi.fn().mockResolvedValue(undefined),
});

describe('MenuAdminPage', () => {
  it('lists items with price and where they are sold', async () => {
    render(<MenuAdminPage api={fakeApi()} />);
    expect(await screen.findByText('Cold brew black')).toBeInTheDocument();
    expect(screen.getByText('S$5.00')).toBeInTheDocument();
    expect(screen.getByText('Weekday pre-order')).toBeInTheDocument();
  });

  it('adds a weekend bean drink in cents', async () => {
    const api = fakeApi();
    render(<MenuAdminPage api={api} />);
    await userEvent.click(await screen.findByRole('button', { name: 'Add item' }));
    await userEvent.type(screen.getByLabelText('Name'), 'Latte');
    await userEvent.type(screen.getByLabelText('Price (S$)'), '5.50');
    await userEvent.selectOptions(screen.getByLabelText('Sold on'), 'weekend_counter');
    await userEvent.click(screen.getByLabelText('Uses this cycle’s A/B beans'));
    await userEvent.click(screen.getByRole('button', { name: 'Save' }));
    expect(api.create).toHaveBeenCalledWith({
      name: 'Latte', description: '', category: 'coffee', price: 550, soldOn: 'weekend_counter', usesBeanSlot: true,
    });
  });

  it('marks an item sold out', async () => {
    const api = fakeApi();
    render(<MenuAdminPage api={api} />);
    await userEvent.click(await screen.findByRole('button', { name: 'Sold out' }));
    expect(api.setSoldOut).toHaveBeenCalledWith('m1', true);
  });

  it('shows a friendly error and does not save a bad price', async () => {
    const api = fakeApi();
    render(<MenuAdminPage api={api} />);
    await userEvent.click(await screen.findByRole('button', { name: 'Add item' }));
    await userEvent.type(screen.getByLabelText('Name'), 'Mocha');
    await userEvent.type(screen.getByLabelText('Price (S$)'), 'six');
    await userEvent.click(screen.getByRole('button', { name: 'Save' }));
    expect(await screen.findByRole('alert')).toHaveTextContent('Enter a price like 5.50');
    expect(api.create).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 6: Run and verify the failure**

Run: `npm test --workspace apps/staff`
Expected: FAIL with `Failed to resolve import "./menuApi"`.

- [ ] **Step 7: Implement the API, form view and page**

`apps/staff/src/features/menu/menuApi.ts`:

```ts
import type { SupabaseClient } from '@supabase/supabase-js';
import type { MenuItemInput } from './menuForm';

export type MenuItemRow = {
  id: string; name: string; description: string; category: 'coffee' | 'non_coffee' | 'food';
  price_cents: number; sold_on: 'weekday_preorder' | 'weekend_counter' | 'both';
  uses_bean_slot: boolean; active: boolean; sold_out: boolean; sort_rank: number;
};

export type MenuApi = {
  list(): Promise<MenuItemRow[]>;
  create(input: MenuItemInput): Promise<void>;
  update(id: string, input: MenuItemInput): Promise<void>;
  setActive(id: string, active: boolean): Promise<void>;
  setSoldOut(id: string, soldOut: boolean): Promise<void>;
};

const COLUMNS = 'id,name,description,category,price_cents,sold_on,uses_bean_slot,active,sold_out,sort_rank';
const toRow = (i: MenuItemInput) => ({
  name: i.name, description: i.description, category: i.category, price_cents: i.price,
  sold_on: i.soldOn, uses_bean_slot: i.usesBeanSlot,
});

function check(error: { message: string } | null, action: string): void {
  if (error) throw new Error(`Could not ${action}. ${error.message}`);
}

export function createMenuApi(client: SupabaseClient): MenuApi {
  const patch = async (id: string, values: Record<string, unknown>, action: string) => {
    const { error } = await client.from('menu_items').update(values).eq('id', id);
    check(error, action);
  };
  return {
    async list() {
      const { data, error } = await client.from('menu_items').select(COLUMNS)
        .order('active', { ascending: false }).order('sort_rank').order('name');
      check(error, 'load the menu');
      return (data ?? []) as MenuItemRow[];
    },
    async create(input) {
      const { error } = await client.from('menu_items').insert(toRow(input));
      check(error, 'add the item');
    },
    update: (id, input) => patch(id, toRow(input), 'save the item'),
    setActive: (id, active) => patch(id, { active }, active ? 'show the item' : 'hide the item'),
    setSoldOut: (id, soldOut) => patch(id, { sold_out: soldOut }, 'update sold out'),
  };
}
```

`apps/staff/src/features/menu/MenuItemFormView.tsx`:

```tsx
import { useState, type FormEvent } from 'react';
import { CATEGORIES, SOLD_ON, SOLD_ON_LABEL, menuItemFormSchema, type MenuItemFormValues, type MenuItemInput } from './menuForm';

type Props = { initial: MenuItemFormValues; onSave: (input: MenuItemInput) => Promise<void>; onCancel: () => void };

export function MenuItemFormView({ initial, onSave, onCancel }: Props) {
  const [values, setValues] = useState<MenuItemFormValues>(initial);
  const [error, setError] = useState<string | null>(null);
  const set = (patch: Partial<MenuItemFormValues>) => setValues((v) => ({ ...v, ...patch }));

  async function onSubmit(e: FormEvent) {
    e.preventDefault();
    const parsed = menuItemFormSchema.safeParse(values);
    if (!parsed.success) return setError(parsed.error.issues[0]?.message ?? 'Check the form');
    try {
      await onSave(parsed.data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Could not save the item.');
    }
  }

  return (
    <form onSubmit={onSubmit} aria-label="Menu item">
      <label>Name<input value={values.name} onChange={(e) => set({ name: e.target.value })} /></label>
      <label>Description<textarea value={values.description} onChange={(e) => set({ description: e.target.value })} /></label>
      <label>Category
        <select value={values.category} onChange={(e) => set({ category: e.target.value as MenuItemFormValues['category'] })}>
          {CATEGORIES.map((c) => <option key={c} value={c}>{c.replace('_', ' ')}</option>)}
        </select>
      </label>
      <label>Price (S$)<input inputMode="decimal" value={values.price} onChange={(e) => set({ price: e.target.value })} /></label>
      <label>Sold on
        <select value={values.soldOn} onChange={(e) => set({ soldOn: e.target.value as MenuItemFormValues['soldOn'] })}>
          {SOLD_ON.map((s) => <option key={s} value={s}>{SOLD_ON_LABEL[s]}</option>)}
        </select>
      </label>
      <label><input type="checkbox" checked={values.usesBeanSlot} onChange={(e) => set({ usesBeanSlot: e.target.checked })} />Uses this cycle’s A/B beans</label>
      {error && <p role="alert">{error}</p>}
      <button type="submit">Save</button>
      <button type="button" onClick={onCancel}>Cancel</button>
    </form>
  );
}
```

`apps/staff/src/features/menu/MenuAdminPage.tsx`:

```tsx
import { useCallback, useEffect, useState } from 'react';
import { formatCents } from '@cafe/shared';
import type { MenuApi, MenuItemRow } from './menuApi';
import { SOLD_ON_LABEL, type MenuItemFormValues } from './menuForm';
import { MenuItemFormView } from './MenuItemFormView';

const EMPTY: MenuItemFormValues = { name: '', description: '', category: 'coffee', price: '', soldOn: 'weekday_preorder', usesBeanSlot: false };
const toForm = (r: MenuItemRow): MenuItemFormValues => ({
  name: r.name, description: r.description, category: r.category, price: (r.price_cents / 100).toFixed(2),
  soldOn: r.sold_on, usesBeanSlot: r.uses_bean_slot,
});
type Editing = { mode: 'new' } | { mode: 'edit'; row: MenuItemRow } | null;

export function MenuAdminPage({ api }: { api: MenuApi }) {
  const [items, setItems] = useState<MenuItemRow[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [editing, setEditing] = useState<Editing>(null);
  const reload = useCallback(() => { api.list().then(setItems, (e: Error) => setError(e.message)); }, [api]);
  useEffect(reload, [reload]);
  const act = (p: Promise<void>) => p.then(reload, (e: Error) => setError(e.message));

  if (editing) {
    return (
      <MenuItemFormView
        initial={editing.mode === 'edit' ? toForm(editing.row) : EMPTY}
        onCancel={() => setEditing(null)}
        onSave={async (input) => {
          await (editing.mode === 'edit' ? api.update(editing.row.id, input) : api.create(input));
          setEditing(null);
          reload();
        }}
      />
    );
  }

  return (
    <main>
      <h1>Menu</h1>
      {error && <p role="alert">{error}</p>}
      <button type="button" onClick={() => setEditing({ mode: 'new' })}>Add item</button>
      <table>
        <thead><tr><th>Item</th><th>Price</th><th>Sold on</th><th /></tr></thead>
        <tbody>
          {items.map((row) => (
            <tr key={row.id} aria-disabled={!row.active}>
              <td>{row.name}{row.sold_out ? ' (sold out)' : ''}</td>
              <td>{formatCents(row.price_cents)}</td>
              <td>{SOLD_ON_LABEL[row.sold_on]}</td>
              <td>
                <button type="button" onClick={() => setEditing({ mode: 'edit', row })}>Edit</button>
                <button type="button" onClick={() => act(api.setSoldOut(row.id, !row.sold_out))}>
                  {row.sold_out ? 'Back in stock' : 'Sold out'}
                </button>
                <button type="button" onClick={() => act(api.setActive(row.id, !row.active))}>{row.active ? 'Hide' : 'Show'}</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </main>
  );
}
```

In `App.tsx`:
- add `const menuApi = createMenuApi(supabase);` at module level;
- replace the `/menu` route element with `<RequireStaff><MenuAdminPage api={menuApi} /></RequireStaff>`.

- [ ] **Step 8: Run all tests and the build**

Run: `npm test && npm run typecheck && npm run build --workspace apps/staff`
Expected: all pass, and the build succeeds.

- [ ] **Step 9: Commit**

```bash
git add apps/staff/src/features/menu apps/staff/src/App.tsx
git commit -m "feat: add staff menu editor with sold-on, bean slot and sold-out controls"
```

---

### Task 10: Public website (`apps/site`): home, menu of the week, about

**Files:**
- Create: `packages/shared/src/snapshot.ts`, `packages/shared/src/snapshot.test.ts`, `apps/site/` (Vite React TS), `apps/site/src/lib/supabase.ts`, `apps/site/src/lib/snapshot.ts`, `apps/site/src/pages/MenuPage.tsx`, `apps/site/src/pages/ContentPage.tsx`, `apps/site/src/App.tsx`, `apps/site/src/test/setup.ts`, `apps/site/public/_redirects`
- Modify: `packages/shared/src/index.ts`, `.github/workflows/ci.yml`
- Test: `apps/site/src/pages/MenuPage.test.tsx`

**Interfaces:**
- Consumes: `site_snapshot()` (Task 6).
- Produces (in `@cafe/shared`):
  - `SiteSnapshot`, `SiteMenuItem`, `SiteCycle` types;
  - `splitMenu(menu: SiteMenuItem[]): { coldBrew: SiteMenuItem[]; weekend: SiteMenuItem[] }`.

  Plan 3 adds the pre-order page; Plan 4 adds images and the announcement banner.

- [ ] **Step 1: Write the failing shared test**

`packages/shared/src/snapshot.test.ts`:

```ts
import { describe, expect, it } from 'vitest';
import { splitMenu, type SiteMenuItem } from './snapshot';

const item = (name: string, sold_on: SiteMenuItem['sold_on']): SiteMenuItem => ({
  id: name, name, description: '', category: 'coffee', price_cents: 500, sold_on, sold_out: false,
  uses_bean_slot: false, modifiers: [],
});

describe('splitMenu', () => {
  it('puts weekday items under cold brew, weekend items under weekend, both in each', () => {
    const { coldBrew, weekend } = splitMenu([item('CB', 'weekday_preorder'), item('Latte', 'weekend_counter'), item('Water', 'both')]);
    expect(coldBrew.map((i) => i.name)).toEqual(['CB', 'Water']);
    expect(weekend.map((i) => i.name)).toEqual(['Latte', 'Water']);
  });
});
```

- [ ] **Step 2: Run, watch the failure, then implement**

Run: `npm test`. Expected: FAIL with `Failed to resolve import "./snapshot"`.

`packages/shared/src/snapshot.ts`:

```ts
export type SiteModifier = { id: string; name: string; group: string; price_delta_cents: number };
export type SiteMenuItem = {
  id: string; name: string; description: string; category: 'coffee' | 'non_coffee' | 'food'; price_cents: number;
  sold_on: 'weekday_preorder' | 'weekend_counter' | 'both'; sold_out: boolean; uses_bean_slot: boolean;
  modifiers: SiteModifier[];
};
export type SiteBean = { slot: 'A' | 'B'; name: string; origin?: string; tasting_notes?: string[] };
export type SiteCycle = { starts_on: string; reveal_on: string; ends_on: string; revealed: boolean; beans: SiteBean[] };
export type SiteContent = { title: string; body: string; image_path: string | null };
export type SiteSnapshot = {
  today: string;
  content: Partial<Record<'home' | 'about' | 'info' | 'announcement', SiteContent>>;
  menu: SiteMenuItem[];
  cycle: SiteCycle | null;
};

export function splitMenu(menu: SiteMenuItem[]): { coldBrew: SiteMenuItem[]; weekend: SiteMenuItem[] } {
  return {
    coldBrew: menu.filter((i) => i.sold_on !== 'weekend_counter'),
    weekend: menu.filter((i) => i.sold_on !== 'weekday_preorder'),
  };
}
```

`packages/shared/src/index.ts`:

```ts
export * from './money';
export * from './snapshot';
```

Run: `npm test`. Expected: PASS.

- [ ] **Step 3: Scaffold the site app**

```bash
npm create vite@latest apps/site -- --template react-ts
npm pkg set --workspace apps/site 'dependencies.@cafe/shared=*'
npm install --workspace apps/site @supabase/supabase-js react-router-dom
npm install --workspace apps/site -D vitest jsdom @testing-library/react @testing-library/jest-dom
npm pkg set --workspace apps/site scripts.test="vitest run"
printf '/* /index.html 200\n' > apps/site/public/_redirects
```

Copy `vite.config.ts` and `src/test/setup.ts` exactly as in Task 8, and `src/lib/supabase.ts` from Task 8 with the error message path changed to `apps/site/.env.local`. Append `      - run: npm run build --workspace apps/site` to CI.

- [ ] **Step 4: Write the failing page test**

`apps/site/src/pages/MenuPage.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react';
import { describe, expect, it } from 'vitest';
import type { SiteSnapshot } from '@cafe/shared';
import { MenuPage } from './MenuPage';

const base: SiteSnapshot = {
  today: '2026-10-17', content: {},
  menu: [
    { id: '1', name: 'Cold brew white', description: 'Smooth and milky', category: 'coffee', price_cents: 600,
      sold_on: 'weekday_preorder', sold_out: false, uses_bean_slot: false,
      modifiers: [{ id: 'o', name: 'Oat', group: 'milk', price_delta_cents: 100 }] },
    { id: '2', name: 'Latte', description: '', category: 'coffee', price_cents: 550,
      sold_on: 'weekend_counter', sold_out: true, uses_bean_slot: true, modifiers: [] },
  ],
  cycle: { starts_on: '2026-10-17', reveal_on: '2026-10-24', ends_on: '2026-10-30', revealed: false,
    beans: [{ slot: 'A', name: 'Mystery Bean A' }, { slot: 'B', name: 'Mystery Bean B' }] },
};

describe('MenuPage', () => {
  it('shows cold brew with milk options and prices', () => {
    render(<MenuPage snapshot={base} />);
    expect(screen.getByText('Cold brew white')).toBeInTheDocument();
    expect(screen.getByText('S$6.00')).toBeInTheDocument();
    expect(screen.getByText('Oat +S$1.00')).toBeInTheDocument();
  });
  it('shows mystery beans and invites guessing before the reveal', () => {
    render(<MenuPage snapshot={base} />);
    expect(screen.getByText('Mystery Bean A')).toBeInTheDocument();
    expect(screen.getByText(/Guess the tasting notes at the counter/)).toBeInTheDocument();
  });
  it('shows names and notes after the reveal', () => {
    const revealed: SiteSnapshot = { ...base, cycle: { ...base.cycle!, revealed: true,
      beans: [{ slot: 'A', name: 'Guji Natural', origin: 'Ethiopia', tasting_notes: ['blueberry', 'cacao'] }] } };
    render(<MenuPage snapshot={revealed} />);
    expect(screen.getByText('Guji Natural')).toBeInTheDocument();
    expect(screen.getByText('Ethiopia · blueberry, cacao')).toBeInTheDocument();
  });
  it('labels sold-out items', () => {
    render(<MenuPage snapshot={base} />);
    expect(screen.getByText('Latte (sold out)')).toBeInTheDocument();
  });
});
```

- [ ] **Step 5: Run and verify the failure**

Run: `npm test --workspace apps/site`
Expected: FAIL with `Failed to resolve import "./MenuPage"`.

- [ ] **Step 6: Implement the pages and data loading**

`apps/site/src/pages/MenuPage.tsx`:

```tsx
import { formatCents, splitMenu, type SiteMenuItem, type SiteSnapshot } from '@cafe/shared';

function ItemRow({ item }: { item: SiteMenuItem }) {
  return (
    <li>
      <h3>{item.sold_out ? `${item.name} (sold out)` : item.name}</h3>
      <p>{formatCents(item.price_cents)}</p>
      {item.description && <p>{item.description}</p>}
      {item.modifiers.length > 0 && (
        <ul aria-label={`${item.name} options`}>
          {item.modifiers.map((m) => <li key={m.id}>{`${m.name} +${formatCents(m.price_delta_cents)}`}</li>)}
        </ul>
      )}
    </li>
  );
}

function Beans({ cycle }: { cycle: NonNullable<SiteSnapshot['cycle']> }) {
  return (
    <section aria-label="This fortnight's beans">
      <h2>This fortnight’s beans</h2>
      {!cycle.revealed && <p>Guess the tasting notes at the counter on opening weekend: get it right first try and the drink is free.</p>}
      <ul>
        {cycle.beans.map((b) => (
          <li key={b.slot}>
            <h3>{b.name}</h3>
            {cycle.revealed && <p>{[b.origin, b.tasting_notes?.join(', ')].filter(Boolean).join(' · ')}</p>}
          </li>
        ))}
      </ul>
    </section>
  );
}

export function MenuPage({ snapshot }: { snapshot: SiteSnapshot }) {
  const { coldBrew, weekend } = splitMenu(snapshot.menu);
  return (
    <main>
      <h1>Menu of the week</h1>
      <section aria-label="Weekday cold brew"><h2>Weekday cold brew (pre-order)</h2>
        <ul>{coldBrew.map((i) => <ItemRow key={i.id} item={i} />)}</ul></section>
      <section aria-label="Weekend shop"><h2>Weekend shop</h2>
        <ul>{weekend.map((i) => <ItemRow key={i.id} item={i} />)}</ul></section>
      {snapshot.cycle && <Beans cycle={snapshot.cycle} />}
    </main>
  );
}
```

`apps/site/src/pages/ContentPage.tsx`:

```tsx
import type { SiteContent } from '@cafe/shared';

export function ContentPage({ content, fallbackTitle }: { content?: SiteContent; fallbackTitle: string }) {
  if (!content) return <main><h1>{fallbackTitle}</h1><p>Coming soon.</p></main>;
  return (
    <main>
      <h1>{content.title || fallbackTitle}</h1>
      {content.body.split('\n\n').map((para, i) => <p key={i}>{para}</p>)}
    </main>
  );
}
```

`apps/site/src/lib/snapshot.ts`:

```ts
import type { SiteSnapshot } from '@cafe/shared';
import { supabase } from './supabase';

export async function fetchSnapshot(): Promise<SiteSnapshot> {
  const { data, error } = await supabase.rpc('site_snapshot');
  if (error) throw new Error('The menu could not load. Please refresh in a moment.');
  return data as SiteSnapshot;
}
```

`apps/site/src/App.tsx`:

```tsx
import { useEffect, useState } from 'react';
import { BrowserRouter, Link, Route, Routes } from 'react-router-dom';
import type { SiteSnapshot } from '@cafe/shared';
import { fetchSnapshot } from './lib/snapshot';
import { ContentPage } from './pages/ContentPage';
import { MenuPage } from './pages/MenuPage';

export default function App() {
  const [snapshot, setSnapshot] = useState<SiteSnapshot | null>(null);
  const [error, setError] = useState<string | null>(null);
  useEffect(() => { fetchSnapshot().then(setSnapshot, (e: Error) => setError(e.message)); }, []);

  if (error) return <p role="alert">{error}</p>;
  if (!snapshot) return <p>Loading…</p>;
  return (
    <BrowserRouter>
      <nav><Link to="/">Home</Link> <Link to="/menu">Menu</Link> <Link to="/about">About us</Link> <Link to="/info">Info</Link></nav>
      <Routes>
        <Route path="/" element={<ContentPage content={snapshot.content.home} fallbackTitle="Welcome" />} />
        <Route path="/menu" element={<MenuPage snapshot={snapshot} />} />
        <Route path="/about" element={<ContentPage content={snapshot.content.about} fallbackTitle="About us" />} />
        <Route path="/info" element={<ContentPage content={snapshot.content.info} fallbackTitle="Info" />} />
      </Routes>
    </BrowserRouter>
  );
}
```

Remove the template's demo CSS and assets.

- [ ] **Step 7: Run all tests and builds**

Run: `npm test && npm run typecheck && npm run build --workspace apps/site && npm run build --workspace apps/staff`
Expected: all pass.

- [ ] **Step 8: Manual check (local)**

1. Copy the env values from `npx supabase status` into `apps/site/.env.local`.
2. In local Studio, publish the `about` content and add a menu item.
3. Run `npm run dev --workspace apps/site`.

Expected: `/menu` and `/about` show the data. A cycle in its mystery week shows "Mystery Bean A".

- [ ] **Step 9: Commit**

```bash
git add packages/shared apps/site .github/workflows/ci.yml package-lock.json
git commit -m "feat: add public website with menu of the week, mystery beans and content pages"
```

---

### Task 11: Production on a custom domain (owner-run checklist)

The owner runs this task. It creates accounts, buys a domain and handles secrets; an agent guides but never types credentials.

- [ ] **Step 1: Domain.** Buy the domain (e.g. on Cloudflare Registrar). Its DNS stays on Cloudflare.
- [ ] **Step 2: Supabase.** Create a project in **Southeast Asia (Singapore)**, then:

```bash
npx supabase login
npx supabase link --project-ref <project-ref>
npx supabase db push
```

  Expected: 5 migrations applied, and `npx supabase migration list` shows local and remote in sync.
- [ ] **Step 3: Telegram.**
  1. Create the bot with @BotFather.
  2. Set the secrets and deploy (paste the values yourself; never commit them):

```bash
npx supabase secrets set TELEGRAM_BOT_TOKEN=<token> TELEGRAM_WEBHOOK_SECRET=<random 32+ chars>
npx supabase functions deploy telegram-webhook --no-verify-jwt
curl "https://api.telegram.org/bot<token>/setWebhook" -d "url=https://<project-ref>.supabase.co/functions/v1/telegram-webhook" -d "secret_token=<same secret>"
```

  3. Both owners send `/start` and receive their chat IDs.
  4. Run `update public.settings set value = '[<id1>, <id2>]' where key = 'telegram_allowed_chat_ids';`
  5. Both send `/start` again.

  Expected: "the café bot is connected".
- [ ] **Step 4: Staff accounts.**
  1. Create the 2 users in Supabase Auth.
  2. Insert their roles into `staff_roles` (owner `admin`, barista `staff`).
  3. Disable public sign-ups.
- [ ] **Step 5: Cloudflare Pages (two projects from the same repo).**
  - `cafe-site`:
    - build `npm ci && npm run build --workspace apps/site`, output `apps/site/dist`;
    - custom domain `<domain>`.
  - `cafe-staff`:
    - build `npm ci && npm run build --workspace apps/staff`, output `apps/staff/dist`;
    - custom domain `staff.<domain>`.
  - Both projects: env vars `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` (the publishable key).
  - Supabase Auth → URL configuration: add `https://staff.<domain>`.

  Expected:
  - `<domain>/menu` loads;
  - `staff.<domain>` asks for sign-in;
  - the owner adds "Cold brew black" in the staff app and it appears on the site after a refresh.
- [ ] **Step 6: Tablet.** Open `staff.<domain>` on the tablet and add it to the home screen.
- [ ] **Step 7: Record evidence.** Update `PROGRESS.md` with the project ref (no keys), both URLs, the migration list and the bot result. Commit:

```bash
git add PROGRESS.md
git commit -m "chore: record production foundation setup"
```

---

## Self-review (done at plan time)

- **Spec v2 coverage (Plan 1 scope):**

  | Requirement | Spec section | Task |
  |---|---|---|
  | Cold brew black/white + milk +S$1 as modifiers | §1, §7 | 2 (schema), 5 (pricing), 10 (shown) |
  | Separate cold brew vs premium beans | §1, §7 | 2 (`bean_use`), 4 (cycle beans must be premium) |
  | 14-day Saturday cycles, A/B resolution | §1, §6.5 | 4, 5 |
  | Giveaways at zero revenue with cost kept | §6.4, §7 | 5 |
  | Website with menu of the week, about and info | §6.1 | 6, 10 |
  | Reveal embargo, accepted terms never public | §6.1, §7 | 6 (SQL-enforced and tested) |
  | All-in-one staff app on a subdomain | §5, §6.3 | 8, 9, 11 (Plans 2–5 add the counter, orders, website editor, dashboard) |
  | Hosting: Cloudflare Pages ×2, Supabase Singapore | §5 | 11 |

- **Deferred by design:**
  - pre-order form and proof check → Plan 3;
  - website text editor, modifier editor and schedules → Plan 4;
  - `guesses` table and one-try rule → Plan 2;
  - pickup tables → Plan 3.
- **Type consistency:**
  - `MenuItemInput.price` is cents everywhere;
  - `MenuItemRow` matches `COLUMNS`;
  - `SiteSnapshot` matches the `site_snapshot()` JSON keys;
  - `record_sale` JSON keys match the tests.
