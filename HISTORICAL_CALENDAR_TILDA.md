# Виджет «Исторический календарь на сегодня» для Tilda

## Рекомендация по источнику данных

**Лучший вариант — JSON через Google Apps Script Web App.** Это устойчивый способ: вы контролируете формат, корректно отдаёте JSON и обходите ограничения CORS/CSV-публикаций. CSV‑публикация проще, но иногда ломается из‑за ограничений Google, кэширования и нюансов CORS.

**Итог:**
- **Надёжный вариант:** Apps Script Web App (JSON).
- **Простой вариант:** опубликованный CSV (подходит для быстрого старта, но менее стабилен).

---

## Структура Google Sheets

**Рекомендуемый формат даты:** поле `mmdd` в виде `0120` (месяц+день). Это самый простой и надёжный ключ для поиска «сегодня». В коде мы формируем `mmdd` из локальной даты пользователя.

**Колонки:**
- `mmdd` — строка в формате `MMDD` (например, `0120`)
- `year` — год (число)
- `text` — описание события
- `image` — URL картинки (может быть пустым)

**Примеры строк (5–7):**

| mmdd | year | text | image |
|---|---:|---|---|
| 0120 | 1969 | Произошла историческая миссия «Аполлон‑8». | https://example.com/img/apollo.jpg |
| 0120 | 1990 | Запуск первого коммерческого спутника связи нового поколения. | |
| 0120 | 2004 | Введён в строй новый мост через реку. | https://example.com/img/bridge.jpg |
| 0121 | 1981 | Открыт музей современного искусства. | |
| 0121 | 2011 | Представлен новый формат городского транспорта. | https://example.com/img/transport.jpg |
| 0122 | 1879 | Впервые использована электрическая лампа в городе. | |
| 0122 | 2020 | Запущен научно‑образовательный портал. | https://example.com/img/portal.jpg |

**Как добавлять новые события:**
- Добавьте новую строку.
- Заполните `mmdd` (две цифры месяца + две цифры дня).
- Укажите `year`, `text` и при наличии `image` (прямая ссылка на изображение).

---

## Надёжный вариант (рекомендуется): Google Apps Script Web App

### Как подготовить таблицу
1. Создайте Google Sheet с колонками: `mmdd`, `year`, `text`, `image`.
2. Заполните строки по образцу выше.

### Как опубликовать JSON через Apps Script
1. В Google Sheets откройте **Extensions → Apps Script**.
2. Вставьте код скрипта ниже.
3. Нажмите **Deploy → New deployment → Web app**.
4. **Execute as:** Me. **Who has access:** Anyone.
5. Сохраните и скопируйте URL веб‑приложения.

**Код Apps Script (для публикации JSON):**
```javascript
function doGet() {
  const sheet = SpreadsheetApp.getActive().getSheetByName('Sheet1');
  const values = sheet.getDataRange().getValues();
  const headers = values[0];
  const rows = values.slice(1);

  const data = rows.map(row => {
    const item = {};
    headers.forEach((h, i) => item[h] = row[i]);
    return item;
  });

  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}
```

---

## Простой вариант: опубликованный CSV

**Как сделать:**
1. В Google Sheets: **File → Share → Publish to web**.
2. Выберите нужный лист, формат `CSV`.
3. Получите ссылку CSV.

**Минусы:**
- Возможны CORS‑ограничения или нестабильность доступа.
- Формат CSV сложнее надёжно парсить, если в тексте есть запятые или переносы.

**Вывод:** можно использовать для быстрого запуска, но лучше перейти на Apps Script.

---

## Финальный HTML‑блок для Tilda (надёжный вариант JSON)

> Вставьте **одним блоком** в HTML‑блок Tilda. Замените `YOUR_WEB_APP_URL` на URL вашего Apps Script веб‑приложения.

```html
<div id="kr-calendar" class="kr-calendar kr-calendar--loading">
  <div class="kr-calendar__header">
    <div class="kr-calendar__date"></div>
    <div class="kr-calendar__weekday"></div>
  </div>
  <div class="kr-calendar__list"></div>
</div>

<style>
  /* Минимальные базовые стили (можно переопределять в Tilda Global CSS) */
  .kr-calendar { max-width: 800px; margin: 0 auto; padding: 16px; }
  .kr-calendar__header { margin-bottom: 16px; }
  .kr-calendar__date { font-size: 20px; font-weight: 600; }
  .kr-calendar__weekday { font-size: 14px; opacity: 0.7; }
  .kr-calendar__list { display: grid; gap: 12px; }
  .kr-calendar__item { display: grid; gap: 8px; }
  .kr-calendar__year { font-weight: 600; }
  .kr-calendar__media { max-width: 100%; }
  .kr-calendar__img { width: 100%; height: auto; display: block; border-radius: 4px; }
  .kr-calendar__empty, .kr-calendar__error { padding: 12px; background: #f5f5f5; border-radius: 6px; }
  .kr-calendar--loading { opacity: 0.6; }
</style>

<script>
(() => {
  const CALENDAR_URL = 'YOUR_WEB_APP_URL'; // <-- замените на URL Apps Script Web App

  const container = document.getElementById('kr-calendar');
  const dateEl = container.querySelector('.kr-calendar__date');
  const weekdayEl = container.querySelector('.kr-calendar__weekday');
  const listEl = container.querySelector('.kr-calendar__list');

  const locale = 'ru-RU';
  const today = new Date();

  const mm = String(today.getMonth() + 1).padStart(2, '0');
  const dd = String(today.getDate()).padStart(2, '0');
  const mmdd = `${mm}${dd}`;

  dateEl.textContent = today.toLocaleDateString(locale, { day: 'numeric', month: 'long' });
  weekdayEl.textContent = today.toLocaleDateString(locale, { weekday: 'long' });

  function setLoading(isLoading) {
    container.classList.toggle('kr-calendar--loading', isLoading);
  }

  function renderEmpty() {
    listEl.innerHTML = '<div class="kr-calendar__empty">В этот день ничего не произошло</div>';
  }

  function renderError() {
    listEl.innerHTML = '<div class="kr-calendar__error">Не удалось загрузить данные календаря</div>';
  }

  function renderItems(items) {
    if (!items.length) {
      renderEmpty();
      return;
    }

    const html = items.map(item => {
      const year = item.year ? `<div class="kr-calendar__year">${item.year}</div>` : '';
      const text = item.text ? `<div class="kr-calendar__text">${item.text}</div>` : '';
      const img = item.image
        ? `<div class="kr-calendar__media"><img class="kr-calendar__img" loading="lazy" src="${item.image}" alt="${item.text || 'Историческое событие'} (${item.year || ''})"></div>`
        : '';

      return `
        <div class="kr-calendar__item">
          ${year}
          ${text}
          ${img}
        </div>
      `;
    }).join('');

    listEl.innerHTML = html;
  }

  async function loadData() {
    setLoading(true);
    try {
      const res = await fetch(CALENDAR_URL, { cache: 'no-store' });
      if (!res.ok) throw new Error('Bad response');
      const data = await res.json();

      const filtered = (data || [])
        .filter(item => String(item.mmdd).padStart(4, '0') === mmdd)
        .sort((a, b) => Number(a.year) - Number(b.year));

      renderItems(filtered);
    } catch (e) {
      renderError();
    } finally {
      setLoading(false);
    }
  }

  loadData();
})();
</script>
```

---

## Простой HTML‑блок (CSV‑вариант)

> Этот вариант может работать не всегда из‑за CORS и особенностей CSV, поэтому используйте как запасной.

```html
<div id="kr-calendar" class="kr-calendar kr-calendar--loading">
  <div class="kr-calendar__header">
    <div class="kr-calendar__date"></div>
    <div class="kr-calendar__weekday"></div>
  </div>
  <div class="kr-calendar__list"></div>
</div>

<style>
  .kr-calendar { max-width: 800px; margin: 0 auto; padding: 16px; }
  .kr-calendar__header { margin-bottom: 16px; }
  .kr-calendar__date { font-size: 20px; font-weight: 600; }
  .kr-calendar__weekday { font-size: 14px; opacity: 0.7; }
  .kr-calendar__list { display: grid; gap: 12px; }
  .kr-calendar__item { display: grid; gap: 8px; }
  .kr-calendar__year { font-weight: 600; }
  .kr-calendar__media { max-width: 100%; }
  .kr-calendar__img { width: 100%; height: auto; display: block; border-radius: 4px; }
  .kr-calendar__empty, .kr-calendar__error { padding: 12px; background: #f5f5f5; border-radius: 6px; }
  .kr-calendar--loading { opacity: 0.6; }
</style>

<script>
(() => {
  const CSV_URL = 'YOUR_PUBLISHED_CSV_URL'; // <-- замените на ссылку CSV

  const container = document.getElementById('kr-calendar');
  const dateEl = container.querySelector('.kr-calendar__date');
  const weekdayEl = container.querySelector('.kr-calendar__weekday');
  const listEl = container.querySelector('.kr-calendar__list');

  const locale = 'ru-RU';
  const today = new Date();

  const mm = String(today.getMonth() + 1).padStart(2, '0');
  const dd = String(today.getDate()).padStart(2, '0');
  const mmdd = `${mm}${dd}`;

  dateEl.textContent = today.toLocaleDateString(locale, { day: 'numeric', month: 'long' });
  weekdayEl.textContent = today.toLocaleDateString(locale, { weekday: 'long' });

  function setLoading(isLoading) {
    container.classList.toggle('kr-calendar--loading', isLoading);
  }

  function renderEmpty() {
    listEl.innerHTML = '<div class="kr-calendar__empty">В этот день ничего не произошло</div>';
  }

  function renderError() {
    listEl.innerHTML = '<div class="kr-calendar__error">Не удалось загрузить данные календаря</div>';
  }

  function renderItems(items) {
    if (!items.length) {
      renderEmpty();
      return;
    }

    const html = items.map(item => {
      const year = item.year ? `<div class="kr-calendar__year">${item.year}</div>` : '';
      const text = item.text ? `<div class="kr-calendar__text">${item.text}</div>` : '';
      const img = item.image
        ? `<div class="kr-calendar__media"><img class="kr-calendar__img" loading="lazy" src="${item.image}" alt="${item.text || 'Историческое событие'} (${item.year || ''})"></div>`
        : '';

      return `
        <div class="kr-calendar__item">
          ${year}
          ${text}
          ${img}
        </div>
      `;
    }).join('');

    listEl.innerHTML = html;
  }

  function parseCSV(text) {
    const lines = text.trim().split('\n');
    const headers = lines[0].split(',').map(h => h.trim());

    return lines.slice(1).map(line => {
      const values = line.split(',').map(v => v.trim());
      const obj = {};
      headers.forEach((h, i) => obj[h] = values[i] || '');
      return obj;
    });
  }

  async function loadData() {
    setLoading(true);
    try {
      const res = await fetch(CSV_URL, { cache: 'no-store' });
      if (!res.ok) throw new Error('Bad response');
      const text = await res.text();
      const data = parseCSV(text);

      const filtered = (data || [])
        .filter(item => String(item.mmdd).padStart(4, '0') === mmdd)
        .sort((a, b) => Number(a.year) - Number(b.year));

      renderItems(filtered);
    } catch (e) {
      renderError();
    } finally {
      setLoading(false);
    }
  }

  loadData();
})();
</script>
```

---

## Список классов и их назначение

- `.kr-calendar` — контейнер виджета
- `.kr-calendar__header` — шапка
- `.kr-calendar__date` — дата
- `.kr-calendar__weekday` — день недели
- `.kr-calendar__list` — список событий
- `.kr-calendar__item` — элемент события
- `.kr-calendar__year` — год
- `.kr-calendar__text` — описание
- `.kr-calendar__media` — обёртка изображения
- `.kr-calendar__img` — изображение
- `.kr-calendar__empty` — сообщение «пусто»
- `.kr-calendar__error` — сообщение об ошибке
- `.kr-calendar--loading` — состояние загрузки

---

## Как стилизовать (стартовые примеры CSS)

> Эти правила можно вставить в Global CSS Tilda и свободно менять.

```css
.kr-calendar { font-family: Arial, sans-serif; }
.kr-calendar__header { border-bottom: 1px solid #eee; padding-bottom: 12px; }
.kr-calendar__date { font-size: 24px; letter-spacing: 0.2px; }
.kr-calendar__weekday { text-transform: capitalize; color: #666; }
.kr-calendar__item { padding: 12px; border: 1px solid #f0f0f0; border-radius: 8px; }
.kr-calendar__year { color: #222; font-size: 18px; }
.kr-calendar__text { line-height: 1.5; color: #444; }
.kr-calendar__img { border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); }
.kr-calendar__empty, .kr-calendar__error { text-align: center; color: #555; }
.kr-calendar--loading { opacity: 0.7; }
```

---

## Куда вставить код в Tilda

1. Откройте страницу в Tilda.
2. Добавьте блок **HTML** (например, T123).
3. Вставьте полный HTML‑код из раздела «Финальный HTML‑блок».
4. Опубликуйте страницу.

---

## Где менять ссылку на источник данных

- В надёжном варианте: `const CALENDAR_URL = 'YOUR_WEB_APP_URL';`
- В CSV‑варианте: `const CSV_URL = 'YOUR_PUBLISHED_CSV_URL';`

---

## Поведение

- Дата и день недели выводятся по **локальному времени браузера** пользователя.
- События фильтруются по `mmdd` и сортируются по году **по возрастанию**.
- Если данных нет — выводится «В этот день ничего не произошло».
- Если источник недоступен — выводится «Не удалось загрузить данные календаря».
