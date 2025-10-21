# PLAYBOOK — WA Event AI (Каталог площадок)

Подробная документация проекта: архитектура, данные, шортлисты, безопасность, деплой и диагностика.

---

## Оглавление
1. Обзор
2. Архитектура и данные
3. Механизм шортлистов (подробно)
4. Правила безопасности (Firestore/Storage)
5. Заполнение данных и медиа
6. Локальная разработка и деплой
7. UI: фильтры, карточки, галерея
8. Диагностика и частые проблемы
9. Чек‑лист запуска

---

## 1) Обзор

Веб‑приложение (статический сайт на Firebase Hosting) отображает каталог площадок из Firestore, поддерживает фильтры, медиа‑галерею и генерацию персональных **шортлистов** — приватных подборок площадок, доступных по ссылке с ограниченным сроком действия и проверкой токена.

**Стек**: Vanilla JS (ES Modules) • Firebase JS SDK v10 • Firebase Auth (Anonymous) • Firestore • Storage • Hosting • (опц.) GitHub Actions.

---

## 2) Архитектура и данные

### Встраиваемая конфигурация (index.html)
```html
<script>
  window.__FIREBASE_CONFIG__ = { /* apiKey, authDomain, projectId, storageBucket, ... */ };
  window.__FIREBASE_COLLECTIONS__ = { venues: "venues", shortlists: "shortlists" };
</script>
```

### Коллекция `venues` (минимальный пример)
```json
{
  "name": "Caspian Grand Ballroom",
  "city": "Baku",
  "district": "Narimanov",
  "capacity_min": 150,
  "capacity_max": 900,
  "price_min": 5,
  "price_max": 40,
  "currency": "AZN",
  "location_url": "https://maps.google.com/?q=40.391,49.854",
  "location_lat": 40.391,
  "location_lng": 49.854,
  "media": { "photos": ["https://.../cover.jpg", "https://.../img2.jpg"] },
  "tags": ["banquet","outdoor"]
}
```

Ключевые поля карточки: `name`, `city`, `district`, `capacity_min|max`, `price_min|max`, `currency`, `location_url` **или** `location_lat/lng`, `media.photos[0]`.

### Каталог загружается в два режима
- **Обычный**: берём *все* документы из `venues` и фильтруем на клиенте.
- **Шортлист**: берём `venue_ids` из `shortlists/<slug>` и загружаем *только их*; фильтры применяются к этому подмножеству.

---

## 3) Механизм шортлистов (подробно)

### Назначение
Генерировать персональные ссылки на подборку площадок: ссылка содержит `shortlist=<slug>&token=<plain>`, а в базе хранится **только хэш** токена. Ссылка истекает по `expires_at`.

### Модель данных `shortlists/<slug>`
```json
{
  "venue_ids": ["garden_orangery", "cottage_lake_gala"],
  "token_hash": "<sha256(plain_token)>",
  "note": "Подбор от 2025‑10‑09, 12:34",
  "created_at": "<serverTimestamp>",
  "expires_at": "2025-10-16T12:34:00.000Z"
}
```

### Процесс создания (UI)
1. Включить **Режим подбора**, выбрать карточки (Set из `id`).
2. Нажать «Сгенерировать ссылку» → модалка (slug, срок, note).
3. Сгенерировать `token = Math.random().toString(36).slice(2,10)`.
4. Посчитать `token_hash = sha256(token)` (Web Crypto API) **на клиенте**.
5. Записать документ:
   ```js
   await setDoc(doc(db,"shortlists",slug), {
     venue_ids: ids,
     token_hash: await sha256Hex(token),
     note,
     created_at: serverTimestamp(),
     expires_at: expiresAt   // Date или ISO‑строка
   });
   ```
6. Сформировать URL `https://<host>/?shortlist=<slug>&token=<token>` → скопировать, показать кнопки WhatsApp/Telegram, (опц.) `navigator.share()`.

**Важно:** в базе нет plain‑token. Потерянную ссылку восстановить нельзя — создаём новый шортлист.

### Процесс открытия
1. Считываем `shortlist` и `token` из URL.
2. Загружаем `shortlists/<slug>`, проверяем:
   - `expires_at` не истёк;
   - `sha256(token_from_url) === token_hash`.
3. По `venue_ids` читаем `venues` (batched), формируем массив.
4. Применяем фильтры к этому массиву; рендерим. Если есть `note`/`expires_at` — показываем баннер.

### Почему хэш
- URL содержит token, но в Firestore хранится **только хэш** (SHA‑256). Даже имея доступ к БД, получить исходный токен нельзя.
- `expires_at` ограничивает «вечные» ссылки.

### Расширения (опционально)
- Пароль вместо токена (хранить Argon2/Bcrypt‑хэш; запрашивать в UI).
- Белый список e‑mail (требует Email/Google‑Auth).
- Подпись URL (JWS) и серверная валидация (Cloud Functions).
- Логи и метрики (GA4 / BigQuery).

---

## 4) Правила безопасности

### Firestore Rules (пример строгих правил)
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /venues/{id} {
      allow read: if true;      // публичное чтение каталога
      allow write: if false;    // изменяем только через Console/Admin
    }

    match /shortlists/{slug} {
      allow read: if true;
      allow create, update: if request.auth != null
        && request.resource.data.keys().hasOnly([
             'venue_ids','token_hash','note','created_at','expires_at'
           ])
        && request.resource.data.venue_ids.size() > 0
        && request.resource.data.token_hash is string
        && request.resource.data.expires_at != null;
      allow delete: if false;
    }
  }
}
```

### Storage Rules (пример)
```javascript
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read: if true;   // картинки публичны
      allow write: if false;
    }
  }
}
```

---

## 5) Заполнение данных и медиа

1. **Storage** → папка `/venues/<slug>/...`, загружаем изображения/видео.
2. Копируем **Download URL** (с `alt=media&token=`).
3. **Firestore → venues**: добавляем/обновляем документ (см. модель).  
   - `media.photos[0]` — обложка;  
   - `location_url` **или** `location_lat/lng` — чтобы работала кнопка «Карта».

---

## 6) Локальная разработка и деплой

### Подготовка (один раз)
```bash
npm i -g firebase-tools
firebase login
firebase init   # (если нужно) выбрать: Hosting, Firestore Rules, Storage Rules
```

### Деплой правил (опционально)
```bash
firebase deploy --only firestore:rules
firebase deploy --only storage:rules
```

### Деплой сайта
```bash
firebase deploy --only hosting
```
После деплоя появятся домены вида `https://<project-id>.web.app` и `https://<project-id>.firebaseapp.com`.

### Автодеплой через GitHub Actions
1. Firebase Console → Hosting → **Connect to GitHub** → выбрать репозиторий/ветку.
2. Разрешить workflow → Actions создаст yaml и будет деплоить на каждый `push`.

---

## 7) UI: фильтры, карточки, галерея

### Фильтры
- Поиск: строка вхождений по `name`, `city`, `district`, `tags`, `description`.
- Район: точное совпадение `district`.
- Вместимость: проверка **пересечения** запросного диапазона `[capMin,capMax]` и `[capacity_min,capacity_max]` площадки.
- Цена/гость: проверка **пересечения** `[priceMin,priceMax]` и `[price_min,price_max]` площадки.
- Значения фильтров синхронизируются в URL (`?q=...&d=...&cap=...&price=...`).

### Карточка площадки
- Обложка (`media.photos[0]`), название, **район** (иконка 📍).
- **Цена**: «от price_min — до price_max currency» (если оба заданы).
- **Вместимость** (иконка 👥): «capacity_min—capacity_max».
- Кнопки:
  - **Медиа** — модалка‑галерея.
  - **Карта** — обычная кнопка; при клике `window.open(mapHref, '_blank', 'noreferrer')`. `mapHref` берётся из `location_url`, либо генерируется по `location_lat/lng`.

### Галерея
- Модалка (overlay): стрелки/клавиши/свайпы, Esc закрывает, снизу лента превью.
- Нормализатор понимает разные структуры `media` (массив строк или объект `photos[]/videos[]`).

---

## 8) Диагностика

- Каталог пуст:
  - Проверь Firestore Rules (чтение публично), наличие документов в `venues`.
- Фото не видны:
  - Storage Rules и валидность Download URL.
- Фильтры «не работают»:
  - Параметры могли «застрять» в URL. Очистить строку или нажать «Все районы/очистить поля».
- Шортлист:
  - «Неверный токен» → сравнить `sha256(token)` и `token_hash`, проверить полный URL.
  - «Ссылка истекла» → `expires_at` в прошлом или на устройстве неверное время.
  - Пустой шортлист → `venue_ids` пуст или документы удалены.
- Кнопка «Карта» ничего не делает:
  - Нет `location_url` и координат. Добавить одно из двух.

---

## 9) Чек‑лист запуска

1) Создать Firebase проект; включить **Auth Anonymous**, **Firestore**, **Storage**, **Hosting**.  
2) Настроить и задеплоить **Rules** (Firetore/Storage).  
3) Залить медиа в Storage; получить Download URLs.  
4) Заполнить `venues` (минимальные поля).  
5) В `index.html` добавить `window.__FIREBASE_CONFIG__` и `window.__FIREBASE_COLLECTIONS__`.  
6) Проверить локально (или `firebase serve`).  
7) `firebase deploy --only hosting`.  
8) (Опц.) Подключить GitHub для автодеплоя.  
9) Проверить: фильтры, медиа, «Карта», генерацию/открытие шортлистов.

---

## Приложение A — пример админ‑скрипта (Node.js, Admin SDK)

```js
// npm i firebase-admin
import admin from "firebase-admin";
import crypto from "crypto";

admin.initializeApp();
const db = admin.firestore();
const sha256 = s => crypto.createHash("sha256").update(s).digest("hex");

async function createShortlist({ slug, venueIds, note, days=7 }) {
  const token = Math.random().toString(36).slice(2,10);
  const expiresAt = new Date(Date.now() + days*24*60*60*1000);

  await db.collection("shortlists").doc(slug).set({
    venue_ids: venueIds,
    token_hash: sha256(token),
    note,
    created_at: admin.firestore.FieldValue.serverTimestamp(),
    expires_at: expiresAt.toISOString()
  });

  return { url: `https://your-domain/?shortlist=${slug}&token=${token}`, token };
}
```
