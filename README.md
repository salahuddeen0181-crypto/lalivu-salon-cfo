# Lalivu — Финансовый пульт салона (CFO)

Одностраничное веб-приложение для учёта финансов салона: операции (доходы/расходы/переводы),
зарплаты (ФОТ), сверка отчётов персонала с банковской выпиской и авто-инсайты «где деньги».
Всё в одном файле `index.html`, без сборки.

**Живая версия:** GitHub Pages (см. вкладку Settings → Pages репозитория).

## Хранилище данных

Приложение хранит данные в одном из трёх мест (по приоритету):

1. **Supabase** — если подключена база и выполнен вход (синхронизация между устройствами).
2. **Облако Claude** — когда страница открыта внутри Claude.
3. **localStorage** — обычный браузер без базы (данные только на этом устройстве).

## Подключение синхронизации (Supabase)

Бесплатно, ~2 минуты.

1. Зарегистрируйтесь на [supabase.com](https://supabase.com) → **New project** (регион ближе к вам,
   например Frankfurt), задайте пароль БД.
2. **SQL Editor** → выполните скрипт (таблица + защита доступа Row Level Security):

   ```sql
   create table if not exists public.salon_kv (
     user_id uuid not null references auth.users(id) on delete cascade,
     k text not null,
     v text,
     updated_at timestamptz default now(),
     primary key (user_id, k)
   );
   alter table public.salon_kv enable row level security;
   create policy "kv_select" on public.salon_kv for select using (auth.uid() = user_id);
   create policy "kv_insert" on public.salon_kv for insert with check (auth.uid() = user_id);
   create policy "kv_update" on public.salon_kv for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
   create policy "kv_delete" on public.salon_kv for delete using (auth.uid() = user_id);
   ```

3. **Authentication → Providers → Email** → снимите галку **Confirm email**
   (чтобы вход был мгновенным без письма-подтверждения).
4. **Settings → API** → скопируйте **Project URL** и **anon public key**.
5. Откройте приложение → вкладка **Настройки → База данных** → вставьте URL и ключ → **Подключить**,
   затем зарегистрируйтесь/войдите по email и паролю.

> anon-ключ публичный по дизайну: доступ к данным защищён входом пользователя и политиками RLS —
> каждый видит только свои строки (`user_id = auth.uid()`).

## Резервные копии

Настройки → «Данные · резервные копии»: выгрузка/загрузка `.json` и экспорт операций в CSV.
