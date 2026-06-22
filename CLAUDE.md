# CLAUDE.md — Финансовый пульт салона Lalivu

Гайд для будущих сессий Claude по этому проекту. Коротко и по делу.

## Что это
Одностраничное веб-приложение (CFO-дашборд) для учёта финансов салона красоты Lalivu.
Без сборки и зависимостей-пакетов: весь код в одном `index.html` (HTML + CSS + ванильный JS).
Внешние библиотеки только с CDN: `@supabase/supabase-js` и `xlsx` (SheetJS), шрифты Google.

## Файлы
- `index.html` — приложение (актуальный дизайн: тёмно-оранжевая тема, левый сайдбар, 6 экранов).
- `classic.html` — предыдущий дизайн (сине-зелёный, верхние вкладки). Бэкап, не трогать без нужды.
- `README.md` — пользовательская инструкция + SQL для Supabase.
- `CLAUDE.md` — этот файл.

## Деплой
- Репозиторий: `github.com/salahuddeen0181-crypto/lalivu-salon-cfo` (публичный). `gh` авторизован.
- Живая версия: https://salahuddeen0181-crypto.github.io/lalivu-salon-cfo/ (GitHub Pages, ветка `main`, корень).
- Деплой = `git push origin main` → Pages пересобирается за ~30–60 сек. `classic.html` доступен по `/classic.html`.
- Проверка статуса: `gh api repos/salahuddeen0181-crypto/lalivu-salon-cfo/pages --jq .status`.

## Хранилище данных (`store` в index.html)
Приоритет: **Supabase** (если залогинен) → `window.storage` (внутри Claude) → **localStorage**.
- Ключи: `salon2:settings`, `salon2:tx`, `salon2:reports`, `salon2:bank` (одинаковы во всех версиях — данные переносятся между дизайнами).
- Supabase вшит в `SB_DEFAULT` (функция `sbCfg`): проект ref `fkwqqpwbossecszebijy`, anon-ключ публичный (защита — вход + RLS). Форма в Настройках переопределяет `SB_DEFAULT`.
- Таблица `salon_kv (user_id uuid, k text, v text, updated_at)`, PK `(user_id,k)`, RLS `auth.uid()=user_id`. Вход по email/паролю, Confirm email выключен.

## Модель данных (`state`)
- `settings`: `salonName, currency(RUB/USD/EUR/ARS), accounts[], incomeCats[], expenseCats[], openings{счёт:остаток}, avgCheck, clients, taxRate(%)`.
- `tx[]`: `{id,type:'income'|'expense'|'transfer',date:'YYYY-MM-DD',amount, cat,account, from,to, note}`.
- `reports[]`: `{id,date,cash,card,who}` (отчёты персонала). `bank[]`: `{id,date,amount,desc}` (выписка).
- Статья, начинающаяся на «ЗП» (`isPayroll`), считается ФОТ.

## Архитектура кода (порядок в `<script>`)
1. Справочники (DEF_*, палитры) → `state` → `store`/`save`/`load`/`syncRefs`.
2. Хелперы: `money, pct, ym, uid, parseDate, cleanNum, monthsBack(UTC!), periodMonths, allMonthsRange`.
3. Метрики: `metrics, monthMetric, accountBalances, totalBalance, cumulativeByMonth, trendBadge, derived`.
4. `reconcile()` — сверка отчётов с банком (комиссия ≤6%, задержка ≤5 дней).
5. Билдеры графиков (`capChartHTML, monthlyBarsHTML, structBars, recentOpsHTML`).
6. Рендер-функции по экранам: `renderOverview(layout 1/2/3), renderOps, renderSalary, renderRecon, renderCfo, renderSettings, renderDb`.
7. Парсеры/импорт: `parseTxBulk/Reports/Bank`, `readFileToGrid, gridFromRows, startMapping` (универсальный маппер), `startDailyReport` (умный импорт «Отчёта дня»).
8. `seedDemo`, `renderAll`, `showSec`, `bind`, IIFE-init.

## Импорт Excel/CSV
- Кнопка «Excel / CSV» в Операциях → `readFileToGrid` → если `isDailyReport(grid)` то `startDailyReport`, иначе `startMapping`.
- **«Отчёт дня»** (колонки: дата · выручка · нал · безнал · продажа товаров · сертификаты · зарплата дня · хоз расход): каждый день → набор операций (услуги нал→Наличные, безнал→Терминал, товары/сертификаты, ЗП, хоз). Есть дедуп по `type|date|cat|account|amount`.
- Даты из Excel читаются через `XLSX.SSF.format('yyyy-mm-dd', serial)` — НЕ через Date-объекты (у XLSX.js баг сдвига дат по таймзоне).

## Подводные камни (важно!)
- `monthsBack` считает окно месяцев в UTC, согласованно с `nowYM()` — иначе в поясах ≠UTC выпадает текущий месяц.
- Excel-даты: только через serial+SSF (см. выше).
- Автоопределение счёта: «Терминал» содержит «нал» — для наличных матчить `/налич|касс/`, не `/нал/`.
- `money()` НЕ конвертирует валюту, только меняет символ (суммы хранятся как есть).

## Локальное тестирование
- Сервер: `python3 -m http.server 8765` в корне репо.
- Браузер: Playwright в venv `/tmp/pw` с системным Chrome (`channel="chrome"`, без скачивания chromium).
  Пример: `/tmp/pw/bin/python script.py`, в скрипте `p.chromium.launch(headless=True, channel="chrome")`.
- Реальный отчёт для проверки импорта: `~/Downloads/Отчет дня.xlsx` (шаблон, 31 день марта 2023).

## Стиль
- UI на русском. Терпеть кириллицу в строках — файл сохранять в UTF-8.
- Рендер через `innerHTML` строками-шаблонами с инлайн-стилями (как в исходном дизайне). Экранировать пользовательский текст `esc()`.
