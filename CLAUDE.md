# CLAUDE.md — Финансовый пульт салона Lalivu

Гайд для будущих сессий Claude по этому проекту. Коротко и по делу.

## Что это
Одностраничное веб-приложение (CFO-дашборд) для учёта финансов салона красоты Lalivu.
Без сборки и зависимостей-пакетов: весь код в одном `index.html` (HTML + CSS + ванильный JS).
Внешние библиотеки только с CDN: `@supabase/supabase-js` и `xlsx` (SheetJS), шрифты Google.

## Файлы
- `index.html` — приложение (актуальный дизайн: тёмно-оранжевая тема, левый сайдбар, 7 экранов: Обзор, Операции, Зарплаты, Сверка, Где деньги, Отчёты, Настройки).
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
- `settings`: `salonName, currency(RUB/USD/EUR/ARS), accounts[], incomeCats[], expenseCats[], openings{счёт:остаток}, encashCats[], staffRules[], avgCheck, clients, taxRate(%)`.
- `tx[]`: `{id,type:'income'|'expense'|'transfer',date:'YYYY-MM-DD',amount, cat,account, from,to, note}`.
- `reports[]`: `{id,date,cash,card,who}` (отчёты персонала). `bank[]`: `{id,date,amount,desc}` (выписка).
- `staffRules[]`: `{id,name,role,mode:'percent'|'fixed',percent,fixed,base,account,cat}`. `base` ∈ `revenue|services|cosmetics|profit|cat:<статья>`. `ruleAmount()` = база×% или фикс; `accrueRule()` создаёт расход-ФОТ на остаток.
- Статья на «ЗП» (`isPayroll`) → ФОТ. Статья инкассации (`isEncash`, по `encashCats` или `/инкасс?ац/i`) — **отдельный тип «Выведено», НЕ расход и НЕ прибыль**.

## Финансовая логика (важно — чтобы цифры сходились со старым сервисом cash)
- **Прибыль** `net = доход − расход` (инкассация исключена из обоих). **Δ денег** `= net − выведено` = реальное изменение остатка на счетах.
- `metrics()`/`computeMetrics(pred)`: `rev,spend` БЕЗ инкассации; `encash`=выведено; `byExp` без инкассации.
- Инкассация уменьшает баланс счёта (в `accountBalances` остаётся как expense), но не попадает в P&L.
- Старый сервис **включает** легитимные «Переводы» между счетами в доход/расход (не схлопывает) и показывает инкассацию отдельной строкой. Поэтому импорт по умолчанию НЕ склеивает переводы (`l_merge` off) — иначе доход/расход занижаются.
- `openings[счёт]` = реальный остаток − движение; задаётся при импорте выписки (опция «подставить стартовые остатки»).

## Архитектура кода (порядок в `<script>`)
1. Справочники (DEF_*, палитры) → `state` → `store`/`save`/`load`/`syncRefs`.
2. Хелперы: `money, pct, ym, uid, parseDate, cleanNum, monthsBack(UTC!), periodMonths, allMonthsRange`.
3. Метрики: `metrics, monthMetric, accountBalances, totalBalance, cumulativeByMonth, trendBadge, derived`.
4. `reconcile()` — сверка отчётов с банком (комиссия ≤6%, задержка ≤5 дней).
5. Билдеры графиков (`capChartHTML, monthlyBarsHTML, structBars, recentOpsHTML`).
6. Рендер-функции по экранам: `renderOverview(layout 1/2/3), renderOps, renderSalary(+renderRulesCard/bindSalary), renderRecon, renderCfo, renderReports(drill-down статей + блок инкассации со своим периодом `encRange`), renderSettings, renderDb`.
7. Парсеры/импорт: `parseCSV` (настоящий CSV: кавычки/`""`/переносы строк внутри полей), `parseTxBulk/Reports/Bank`, `readFileToGrid, gridFromRows`, `startMapping` (универсальный), `startDailyReport` («Отчёт дня»), `startLedger` (выписка `дата;статья;описание;сумма;счёт;валюта`).
8. `seedDemo`, `renderAll`, `showSec`, `bind`, IIFE-init.

## Импорт Excel/CSV
- Кнопка «Excel / CSV» в Операциях → `readFileToGrid` → `isDailyReport`→`startDailyReport`, иначе `isLedger`→`startLedger`, иначе `startMapping`.
- **Выписка операций** (`startLedger`, формат экспорта cash: `date;category;description;amount;account;currency`, сумма со знаком): доход/расход по знаку, инкассация распознаётся по статье, «Переводы» опц. склеиваются в пары (по дате+|сумма|). Опция подставить стартовые остатки под реальные. Реальный файл: `~/Downloads/cash_*_utf8.csv`.
- **«Отчёт дня»** (колонки: дата · выручка · нал · безнал · продажа товаров · сертификаты · зарплата дня · хоз расход): каждый день → набор операций (услуги нал→Наличные, безнал→Терминал, товары/сертификаты, ЗП, хоз). Есть дедуп по `type|date|cat|account|amount`.
- Даты из Excel читаются через `XLSX.SSF.format('yyyy-mm-dd', serial)` — НЕ через Date-объекты (у XLSX.js баг сдвига дат по таймзоне).

## Подводные камни (важно!)
- `monthsBack` считает окно месяцев в UTC, согласованно с `nowYM()` — иначе в поясах ≠UTC выпадает текущий месяц.
- Excel-даты: только через serial+SSF (см. выше).
- Автоопределение счёта: «Терминал» содержит «нал» — для наличных матчить `/налич|касс/`, не `/нал/`.
- `money()` НЕ конвертирует валюту, только меняет символ (суммы хранятся как есть).
- CSV парсить только через `parseCSV` (state-machine с кавычками), НЕ через `split(';')` — в описаниях бывают переносы строк/запятые внутри кавычек.
- Дедуп при импорте — только против УЖЕ существующих `state.tx`; внутри одного файла одинаковые строки (тот же день/статья/сумма/счёт — частый случай) НЕ схлопывать, иначе теряются легитимные операции.

## Локальное тестирование
- Сервер: `python3 -m http.server 8765` в корне репо.
- Браузер: Playwright в venv `/tmp/pw` с системным Chrome (`channel="chrome"`, без скачивания chromium).
  Пример: `/tmp/pw/bin/python script.py`, в скрипте `p.chromium.launch(headless=True, channel="chrome")`.
- Реальный отчёт для проверки импорта: `~/Downloads/Отчет дня.xlsx` (шаблон, 31 день марта 2023).

## Стиль
- UI на русском. Терпеть кириллицу в строках — файл сохранять в UTF-8.
- Рендер через `innerHTML` строками-шаблонами с инлайн-стилями (как в исходном дизайне). Экранировать пользовательский текст `esc()`.
