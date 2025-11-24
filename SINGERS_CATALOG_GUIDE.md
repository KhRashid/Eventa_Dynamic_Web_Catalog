# Руководство по каталогу певцов

## Быстрый старт

### 1. Структура данных певца в Firestore

Коллекция: `singers`

```json
{
  "slug": "unique-singer-id",
  "name": "Имя Певца",
  "aliases": ["Псевдоним 1", "Псевдоним 2"],
  "gender": "female",
  "status": "published",
  "city": "Baku",
  "regions_covered": ["Baku", "Sumqayit", "Ganja"],
  "phones": ["+994501234567", "+994551234567"],
  "genres": ["pop", "jazz", "ballad"],
  "tags": ["wedding", "corporate", "cover"],
  "languages": ["az", "ru", "en"],
  "contact_public": {
    "instagram": "https://instagram.com/username",
    "youtube": "https://youtube.com/@username",
    "tiktok": "https://tiktok.com/@username"
  },
  "pricing_packages": [
    {
      "title": "Стандарт · 45 мин",
      "description": "Вокальное выступление с фонограммами",
      "price": 800,
      "currency": "AZN",
      "duration_min": 45,
      "is_active": true
    },
    {
      "title": "Премиум · 90 мин",
      "description": "Расширенная программа с живым музыкантом",
      "price": 1500,
      "currency": "AZN",
      "duration_min": 90,
      "is_active": true
    }
  ],
  "repertoire": [
    {
      "title": "Perfect",
      "original_artist": "Ed Sheeran",
      "language": "en",
      "genres": ["pop", "ballad"],
      "duration_sec": 263
    },
    {
      "title": "Sarı Gəlin",
      "original_artist": "Традиционная",
      "language": "az",
      "genres": ["folk"],
      "duration_sec": 210
    }
  ],
  "media": {
    "photos": [
      "https://storage.googleapis.com/.../cover.jpg",
      "https://storage.googleapis.com/.../photo2.jpg"
    ],
    "videos": [
      "https://youtu.be/video_id"
    ]
  }
}
```

### 2. Обязательные поля

- `name` — имя певца
- `status` — **обязательно "published"** для отображения в каталоге
- `pricing_packages` — хотя бы один активный пакет (для фильтра по цене)
- `media.photos[0]` — обложка карточки

### 3. Доступ к каталогу

**Прямая ссылка:**
```
https://your-domain/?catalog=singers
```

**Шортлист певцов:**
```
https://your-domain/?catalog=singers&shortlist=wedding-singers&token=abc123xyz
```

### 4. Firestore Rules

Добавьте в `firestore.rules`:

```javascript
match /singers/{id} {
  allow read: if true;
  allow write: if false;  // изменения только через Console
}

match /singer_shortlists/{slug} {
  allow read: if true;
  allow create, update: if request.auth != null
    && request.resource.data.keys().hasOnly([
         'singer_ids','token_hash','note','created_at','expires_at'
       ])
    && request.resource.data.singer_ids.size() > 0
    && request.resource.data.token_hash is string
    && request.resource.data.expires_at != null;
  allow delete: if false;
}
```

### 5. Особенности карточки певца

**Отображается:**
- Имя + псевдонимы
- Фото (media.photos[0])
- Город + регионы выезда (первые 3)
- Жанры (первые 3)
- Языки исполнения
- Цена: минимум и максимум из pricing_packages

**Кнопки:**
- **Медиа** — галерея фото/видео
- **Репертуар (N)** — модалка со списком песен
  - Группировка по языкам
  - Поиск по названию/исполнителю
  - Фильтр по языку
- **Пакеты** — модалка с тарифами
  - Название, цена, длительность, описание
- **Контакты** — модалка с телефонами и соцсетями

### 6. Фильтры

1. **Поиск** — по имени, псевдонимам, тегам, жанрам
2. **Цена** (основной фильтр) — диапазон из pricing_packages
3. **Пол** — female/male/group/duo
4. **Жанры** — выпадающий список (динамически из базы)
5. **Языки** — выпадающий список (динамически из базы)
6. **Город** — базовый город певца

### 7. Режим подбора и шортлисты

1. Включить "Режим подбора"
2. Выбрать нужных певцов (галочка на карточке)
3. Нажать "Сгенерировать ссылку"
4. Настроить:
   - Slug (идентификатор)
   - Срок действия (7/14/30 дней или свой)
   - Заметка клиенту
5. Получить ссылку с токеном для клиента

**Коллекция шортлистов:** `singer_shortlists`

**Структура:**
```json
{
  "singer_ids": ["singer-1", "singer-2"],
  "token_hash": "sha256...",
  "note": "Певцы на свадьбу 15.11",
  "created_at": "...",
  "expires_at": "2025-11-30T23:59:59.000Z"
}
```

### 8. Тестирование

**Checklist:**
- [ ] Певец с `status: "published"` отображается в каталоге
- [ ] Певец с `status: "draft"` НЕ отображается
- [ ] Фото певца загружается
- [ ] Фильтр по цене работает (из pricing_packages)
- [ ] Модалка "Репертуар" открывается и показывает песни
- [ ] Модалка "Пакеты" открывается и показывает тарифы
- [ ] Модалка "Контакты" показывает телефоны и соцсети
- [ ] Поиск находит певца по имени/псевдониму
- [ ] Шортлист певцов создаётся и открывается по ссылке
- [ ] Переключение табов venues ↔ singers работает

### 9. Частые проблемы

**Певец не отображается:**
- Проверьте `status: "published"`
- Проверьте Firestore Rules (allow read: if true)

**Цена не отображается:**
- Убедитесь что есть хотя бы один пакет с `is_active: true`
- Проверьте что поле `price` — число

**Репертуар пустой:**
- `repertoire` должен быть массивом
- Каждая песня должна иметь `title` и `language`

**Модалки не открываются:**
- Проверьте структуру: `pricing_packages`, `repertoire` — массивы
- `media` — объект с `photos[]` и `videos[]`

### 10. Пример создания певца через Firebase Console

```
1. Firestore → singers → Add document
2. Document ID: "amelia-stone" (slug)
3. Добавить поля:
   - name (string): "Amelia Stone"
   - status (string): "published"
   - gender (string): "female"
   - city (string): "Baku"
   - pricing_packages (array): [
       {
         title: "Solo · 45 min",
         price: 900,
         currency: "AZN",
         duration_min: 45,
         is_active: true
       }
     ]
   - media (map):
     photos (array): ["https://..."]
```

### 11. Storage структура

```
/singers/
  ├─ amelia-stone/
  │   ├─ cover.jpg
  │   ├─ photo2.jpg
  │   └─ video1.mp4
  ├─ gunel-mamedova/
  │   ├─ cover.jpg
  │   └─ live-performance.jpg
  └─ ...
```

**Получение Download URL:**
1. Upload файл в Storage
2. Скопировать Download URL (с токеном)
3. Добавить в `media.photos[]` или `media.videos[]`

---

## Дополнительно

- Полная документация: `PLAYBOOK_CATALOG.md`
- Пример админ-скрипта создания шортлистов: см. Приложение A в PLAYBOOK
- GitHub Issues: для багов и предложений
