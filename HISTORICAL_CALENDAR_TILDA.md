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

## Финальный JS‑код для Tilda ZeroBlock (надёжный вариант JSON)

> В ZeroBlock вы заранее создаёте элементы с нужными классами. Скрипт **не создаёт разметку**, не добавляет стили и **не использует innerHTML** — он только заполняет существующие элементы и скрывает лишние. Замените `YOUR_WEB_APP_URL` на URL вашего Apps Script веб‑приложения.

```html
<script>
(() => {
  const CALENDAR_URL = 'YOUR_WEB_APP_URL'; // <-- замените на URL Apps Script Web App
  const locale = 'ru-RU';
  const timeZone = 'Europe/Minsk';

  // Скрипт заполняет только первый набор элементов на странице.
  const dateEl = document.querySelector('.kr-calendar-date');
  const emptyEl = document.querySelector('.kr-calendar-empty');
  const itemEls = Array.from(document.querySelectorAll('.kr-calendar-item'));

  if (!dateEl || !emptyEl || itemEls.length === 0) {
    return;
  }

  const today = new Date();
  const formatter = new Intl.DateTimeFormat(locale, { day: 'numeric', month: 'long', timeZone });
  const parts = new Intl.DateTimeFormat(locale, {
    day: '2-digit',
    month: '2-digit',
    timeZone,
  }).formatToParts(today);

  const mm = parts.find(part => part.type === 'month')?.value || '';
  const dd = parts.find(part => part.type === 'day')?.value || '';
  const mmdd = `${mm}${dd}`;

  dateEl.textContent = formatter.format(today);

  function setEmptyVisible(isVisible) {
    emptyEl.style.display = isVisible ? '' : 'none';
  }

  function setItemVisible(el, isVisible) {
    el.style.display = isVisible ? '' : 'none';
  }

  function setImageVisible(wrapperEl, imgEl, isVisible) {
    if (wrapperEl) {
      wrapperEl.style.display = isVisible ? '' : 'none';
    }
    if (imgEl) {
      imgEl.style.display = isVisible ? '' : 'none';
    }
  }

  function fillItems(items) {
    if (!items.length) {
      setEmptyVisible(true);
      itemEls.forEach(el => setItemVisible(el, false));
      return;
    }

    setEmptyVisible(false);

    itemEls.forEach((itemEl, index) => {
      const data = items[index];
      if (!data) {
        setItemVisible(itemEl, false);
        return;
      }

      setItemVisible(itemEl, true);

      const yearEl = itemEl.querySelector('.kr-calendar-year');
      const textEl = itemEl.querySelector('.kr-calendar-text');
      const imageEl = itemEl.querySelector('.kr-calendar-image img') || itemEl.querySelector('.kr-calendar-image');

      if (yearEl) yearEl.textContent = data.year ? String(data.year) : '';
      if (textEl) textEl.textContent = data.text ? String(data.text) : '';

      const imageUrl = data.image ? String(data.image).trim() : '';
      if (imageEl && imageEl.tagName.toLowerCase() === 'img') {
        if (imageUrl) {
          imageEl.src = imageUrl;
          imageEl.alt = data.text ? String(data.text) : 'Историческое событие';
        } else {
          imageEl.removeAttribute('src');
          imageEl.alt = '';
        }
      }

      const imageWrapper = itemEl.querySelector('.kr-calendar-image');
      setImageVisible(imageWrapper, imageEl, Boolean(imageUrl));
    });
  }

  async function loadData() {
    try {
      const res = await fetch(CALENDAR_URL, { cache: 'no-store' });
      if (!res.ok) throw new Error('Bad response');
      const data = await res.json();

      const filtered = (data || [])
        .filter(item => String(item.mmdd || '').padStart(4, '0') === mmdd)
        .sort((a, b) => Number(a.year) - Number(b.year));

      fillItems(filtered);
    } catch (error) {
      setEmptyVisible(true);
      itemEls.forEach(el => setItemVisible(el, false));
      console.error('Не удалось загрузить данные календаря', error);
    }
  }

  loadData();
})();
</script>
```

---

## Простой JS‑код (CSV‑вариант)

> Этот вариант может работать не всегда из‑за CORS и особенностей CSV, поэтому используйте как запасной. Скрипт **не создаёт разметку**, не добавляет стили и **не использует innerHTML**.

```html
<script>
(() => {
  const CSV_URL = 'YOUR_PUBLISHED_CSV_URL'; // <-- замените на ссылку CSV
  const locale = 'ru-RU';
  const timeZone = 'Europe/Minsk';

  // Скрипт заполняет только первый набор элементов на странице.
  const dateEl = document.querySelector('.kr-calendar-date');
  const emptyEl = document.querySelector('.kr-calendar-empty');
  const itemEls = Array.from(document.querySelectorAll('.kr-calendar-item'));

  if (!dateEl || !emptyEl || itemEls.length === 0) {
    return;
  }

  const today = new Date();
  const formatter = new Intl.DateTimeFormat(locale, { day: 'numeric', month: 'long', timeZone });
  const parts = new Intl.DateTimeFormat(locale, {
    day: '2-digit',
    month: '2-digit',
    timeZone,
  }).formatToParts(today);

  const mm = parts.find(part => part.type === 'month')?.value || '';
  const dd = parts.find(part => part.type === 'day')?.value || '';
  const mmdd = `${mm}${dd}`;

  dateEl.textContent = formatter.format(today);

  function setEmptyVisible(isVisible) {
    emptyEl.style.display = isVisible ? '' : 'none';
  }

  function setItemVisible(el, isVisible) {
    el.style.display = isVisible ? '' : 'none';
  }

  function setImageVisible(wrapperEl, imgEl, isVisible) {
    if (wrapperEl) {
      wrapperEl.style.display = isVisible ? '' : 'none';
    }
    if (imgEl) {
      imgEl.style.display = isVisible ? '' : 'none';
    }
  }

  function fillItems(items) {
    if (!items.length) {
      setEmptyVisible(true);
      itemEls.forEach(el => setItemVisible(el, false));
      return;
    }

    setEmptyVisible(false);

    itemEls.forEach((itemEl, index) => {
      const data = items[index];
      if (!data) {
        setItemVisible(itemEl, false);
        return;
      }

      setItemVisible(itemEl, true);

      const yearEl = itemEl.querySelector('.kr-calendar-year');
      const textEl = itemEl.querySelector('.kr-calendar-text');
      const imageEl = itemEl.querySelector('.kr-calendar-image img') || itemEl.querySelector('.kr-calendar-image');

      if (yearEl) yearEl.textContent = data.year ? String(data.year) : '';
      if (textEl) textEl.textContent = data.text ? String(data.text) : '';

      const imageUrl = data.image ? String(data.image).trim() : '';
      if (imageEl && imageEl.tagName.toLowerCase() === 'img') {
        if (imageUrl) {
          imageEl.src = imageUrl;
          imageEl.alt = data.text ? String(data.text) : 'Историческое событие';
        } else {
          imageEl.removeAttribute('src');
          imageEl.alt = '';
        }
      }

      const imageWrapper = itemEl.querySelector('.kr-calendar-image');
      setImageVisible(imageWrapper, imageEl, Boolean(imageUrl));
    });
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
    try {
      const res = await fetch(CSV_URL, { cache: 'no-store' });
      if (!res.ok) throw new Error('Bad response');
      const text = await res.text();
      const data = parseCSV(text);

      const filtered = (data || [])
        .filter(item => String(item.mmdd || '').padStart(4, '0') === mmdd)
        .sort((a, b) => Number(a.year) - Number(b.year));

      fillItems(filtered);
    } catch (error) {
      setEmptyVisible(true);
      itemEls.forEach(el => setItemVisible(el, false));
      console.error('Не удалось загрузить данные календаря', error);
    }
  }

  loadData();
})();
</script>
```

---

## Список классов и их назначение (ZeroBlock)

- `.kr-calendar-date` — элемент для строки даты (например, «20 января»)
- `.kr-calendar-empty` — элемент с текстом «Мы ничего не знаем о знаковых событиях в этот день» (показывается только если событий нет или ошибка загрузки)
- `.kr-calendar-item` — карточка события (можно создать несколько заранее)
  - `.kr-calendar-year` — год события
  - `.kr-calendar-text` — описание события
  - `.kr-calendar-image` — контейнер изображения или сам `<img>` (если пустой URL — скрывается)

---

## Как стилизовать

Стилизацию полностью делайте через ZeroBlock/Global CSS Tilda — скрипт не добавляет стили и не создаёт новые элементы.

---

## Что создать в ZeroBlock

1. Текстовый элемент с классом `.kr-calendar-date` — сюда подставляется строка даты (например, «20 января»).
2. Текстовый элемент с классом `.kr-calendar-empty` и текстом **«Мы ничего не знаем о знаковых событиях в этот день»** — будет показываться только если событий нет или произошла ошибка загрузки.
3. Несколько одинаковых карточек события с классом `.kr-calendar-item`. Внутри каждой карточки:
   - `.kr-calendar-year` — место для года.
   - `.kr-calendar-text` — место для описания.
   - `.kr-calendar-image` — контейнер изображения или сам `<img>` (если URL пустой, элемент будет скрыт).

Скрипт заполнит карточки по порядку и скроет лишние, если событий меньше, чем карточек.

---

## Куда вставить код в Tilda

1. Откройте страницу в Tilda.
2. Добавьте блок **HTML** (например, T123).
3. Вставьте `<script>...</script>` из раздела «Финальный JS‑код для Tilda ZeroBlock».
4. Опубликуйте страницу.

---

## Где менять ссылку на источник данных

- В надёжном варианте: `const CALENDAR_URL = 'YOUR_WEB_APP_URL';`
- В CSV‑варианте: `const CSV_URL = 'YOUR_PUBLISHED_CSV_URL';`

---

## Поведение

- Дата выводится на русском языке с таймзоной **Europe/Minsk** (через `Intl.DateTimeFormat`).
- События фильтруются по `mmdd` и сортируются по году **по возрастанию**.
- Если данных нет или источник недоступен — показывается элемент `.kr-calendar-empty`.
