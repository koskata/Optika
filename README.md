# Медицински оптик — приложение за записки

Уеб приложение, което се инсталира на телефона като истинско приложение (икона на началния екран, цял екран, работи и без интернет) и се синхронизира с компютъра.

**Не изисква Claude акаунт.** Влиза се с имейл и парола.

---

## Какво има в папката

| Файл | За какво е |
|---|---|
| `index.html` | цялото приложение |
| `config.js` | **единственият файл, който трябва да редактираш** — адрес и ключ за синхронизацията |
| `manifest.webmanifest` | прави го инсталируемо (име, икона, цветове) |
| `sw.js` | работа без интернет |
| `icons/` | иконите за началния екран |

---

## Стъпка 1 — качване в GitHub Pages

1. Направи акаунт в [github.com](https://github.com), ако нямаш.
2. **New repository** → име например `optika` → **Public** → Create.
3. В празното хранилище: **uploading an existing file** → влачи вътре *съдържанието* на папката (`index.html`, `config.js`, `manifest.webmanifest`, `sw.js` и папката `icons`) → **Commit changes**.
4. **Settings** → **Pages** → Source: *Deploy from a branch* → Branch: `main`, папка `/ (root)` → **Save**.
5. След около минута адресът е готов: `https://ТВОЕТО-ИМЕ.github.io/optika/`

Вече работи. Можеш да го отвориш и да го ползваш веднага — но засега пази данните само на устройството. Синхронизацията идва със Стъпка 2.

---

## Стъпка 2 — синхронизация със Supabase

Безплатен план, без карта.

### 2.1 Направи проект

[supabase.com](https://supabase.com) → **New project**. Избери име и парола за базата (тази парола не ти трябва после). Регион: Frankfurt или друг европейски. Изчакай да се вдигне.

### 2.2 Направи таблицата и хранилището

Отвори **SQL Editor** → **New query**, постави всичко отдолу и натисни **Run**:

```sql
-- таблица с данните
create table if not exists public.items (
  user_id    uuid    not null default auth.uid() references auth.users(id) on delete cascade,
  id         text    not null,
  kind       text    not null,
  data       jsonb   not null default '{}'::jsonb,
  updated_at bigint  not null default 0,
  deleted    boolean not null default false,
  primary key (user_id, id)
);

alter table public.items enable row level security;

drop policy if exists "own items" on public.items;
create policy "own items" on public.items
  for all to authenticated
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);

-- обновяване на живо между телефона и компютъра
do $$
begin
  alter publication supabase_realtime add table public.items;
exception when duplicate_object then null;
end $$;

-- хранилище за прикачените файлове
insert into storage.buckets (id, name, public)
values ('files', 'files', false)
on conflict (id) do nothing;

drop policy if exists "own files select" on storage.objects;
create policy "own files select" on storage.objects for select to authenticated
  using (bucket_id = 'files' and (storage.foldername(name))[1] = auth.uid()::text);

drop policy if exists "own files insert" on storage.objects;
create policy "own files insert" on storage.objects for insert to authenticated
  with check (bucket_id = 'files' and (storage.foldername(name))[1] = auth.uid()::text);

drop policy if exists "own files delete" on storage.objects;
create policy "own files delete" on storage.objects for delete to authenticated
  using (bucket_id = 'files' and (storage.foldername(name))[1] = auth.uid()::text);
```

Трябва да пише `Success. No rows returned`.

### 2.3 Изключи потвърждението по имейл (по избор, но по-лесно)

**Authentication** → **Sign In / Providers** → **Email** → изключи **Confirm email** → Save.

Така регистрацията влиза направо. Ако го оставиш включено, просто ще трябва да се потвърди линк от пощата преди първото влизане.

### 2.4 Вземи двата ключа

**Project Settings** → **API** (или **Data API**). Трябват ти:

- **Project URL** — нещо като `https://abcdefghijkl.supabase.co`
- **anon / public** ключ — дългият низ (внимавай: **не** `service_role`)

### 2.5 Сложи ги в `config.js`

В GitHub отвори `config.js` → молива за редакция → попълни:

```js
window.APP_CONFIG = {
  supabaseUrl: "https://abcdefghijkl.supabase.co",
  supabaseAnonKey: "eyJhbGciOi....дългият ключ"
};
```

→ **Commit changes**. След минута е активно.

> **Безопасно ли е, че ключът е публичен?** Да. Затова се казва „публичен“. Достъпът до данните се пази от правилата (RLS) по-горе — всеки вижда само своите редове и своите файлове. Никога не слагай `service_role` ключа тук.

### 2.6 Регистрация

Отвори приложението → облачето горе вдясно → **Регистрация** с нейния имейл и парола. След това същото влизане на другото устройство — и двете показват едно и също.

Точката на облачето показва състоянието: зелена = синхронизирано, жълта = праща в момента, червена = няма връзка (ще опита пак сама), сива = само локално.

---

## Стъпка 3 — инсталиране на телефона

**iPhone (Safari):** отвори адреса → бутона за споделяне → **Добави към началния екран**.

**Android (Chrome):** отвори адреса → менюто с трите точки → **Инсталирай приложението**.

На компютър в Chrome или Edge има икона за инсталиране вдясно в адресната лента.

---

## Добре е да се знае

- **Работи офлайн.** Всичко се пише първо на устройството и се качва, когато има интернет. Може да пише записки в аудитория без сигнал.
- **Прикачените файлове** изискват влизане в профила и интернет — те не се пазят локално.
- **Резервно копие.** От облачето → *Свали резервно копие* сваля един JSON файл с всичко. *Възстанови от копие* го връща обратно.
- **Безплатният план на Supabase** приспива проекта след около седмица пълно бездействие. При ежедневна употреба това не се случва; ако стане, се събужда от таблото на Supabase.
- **Примерните предмети** при първо пускане са примерни. В „Предмети“ има бутон *Махни ги*.

## Обновяване по-късно

Заменяш `index.html` в хранилището с новата версия (Add file → Upload files → същото име → Commit). При промяна в приложението трябва да се смени и номерът на реда `const CACHE = "optika-v1"` в `sw.js`, за да се обнови кешът на телефона.
