# Express + MongoDB Atlas + Passport: CRUD API

Навчальний проєкт на Node.js, Express, MongoDB Atlas, Mongoose та Passport Local. Він зберігає існуючу авторизацію і сторінки EJS (`/login`, `/register`, `/users`) та додає REST API для CRUD-операцій.

## Можливості

- Реєстрація та вхід через Passport Local; сесії зберігаються в MongoDB Atlas.
- Перегляд сторінок користувачів (`/users`, `/users/:userId`).
- API CRUD із використанням нативних методів MongoDB Node.js Driver: `find`, `insertOne`, `insertMany`, `updateOne`, `updateMany`, `replaceOne`, `deleteOne`, `deleteMany`.
- Проєкція полів у `find()` через параметр `fields`.
- CRUD API захищений існуючим middleware `requireAuth`: спочатку увійдіть у систему, а потім надсилайте запити з тією самою сесією/cookie.

## Вимоги

- Node.js 18 або новіший
- MongoDB Atlas cluster і connection string

## Запуск

1. Розпакуйте архів і відкрийте термінал у папці проєкту.
2. Встановіть залежності:

   ```bash
   npm install
   ```

3. Створіть `.env` на основі `.env.example` і заповніть значення:

   ```env
   PORT=3000
   MONGODB_URI=mongodb+srv://<username>:<password>@<cluster-url>/<database>?retryWrites=true&w=majority
   SESSION_SECRET=replace-with-a-long-random-secret
   ```

   Замініть рядок підключення своїм URI з MongoDB Atlas. Додайте IP-адресу до Network Access в Atlas і переконайтеся, що database user має необхідні права. Не публікуйте `.env` у Git.

4. Запустіть сервер:

   ```bash
   npm run dev
   ```

   або `npm start`. Відкрийте `http://localhost:3000` та створіть користувача через `/register`, потім увійдіть через `/login`.

## CRUD API

Базова адреса: `http://localhost:3000/api/records`.

Усі маршрути нижче вимагають автентифікації. Надсилайте `Content-Type: application/json` для JSON-тiл. У браузері після входу можна використовувати cookie сесії; у Postman увімкніть збереження cookies. Запити з неавтентифікованої сесії перенаправляються на сторінку входу згідно з наявним `requireAuth`.

Документи API зберігаються в окремій колекції `crud_records`, щоб навчальні CRUD-операції не змінювали облікові записи Passport у колекції `users`.

### 1. Прочитати документи: `find()` і projection

`GET /api/records`

Параметри:
- `fields=name,category,price` — повертає лише вказані поля та `_id`.
- `limit=20` — кількість документів (за замовчуванням 50, максимум 100).
- Якщо `fields` не вказано, парольні поля `password` і `passwordHash` виключаються.

Приклад:

```http
GET /api/records?fields=name,category,price&limit=20
```

Приклад відповіді `200 OK`:

```json
{
  "count": 1,
  "records": [
    { "_id": "671a1c2d3e4f506172839405", "name": "Keyboard", "category": "accessories", "price": 49.99 }
  ]
}
```

### 2. Додати один документ: `insertOne()`

`POST /api/records`

Тіло запиту:

```json
{ "name": "Gaming Mouse", "category": "accessories", "price": 29.99 }
```

Відповідь `201 Created` (ідентифікатор буде згенеровано MongoDB):

```json
{
  "acknowledged": true,
  "insertedId": "...",
  "record": {
    "_id": "...",
    "name": "Gaming Mouse",
    "category": "accessories",
    "price": 29.99,
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

### 3. Додати багато документів: `insertMany()`

`POST /api/records/bulk`

Тіло — масив від 1 до 100 об'єктів:

```json
[
  { "name": "Mouse Pad", "category": "accessories", "price": 12.5 },
  { "name": "Headset", "category": "audio", "price": 79.9 }
]
```

Відповідь `201 Created`:

```json
{ "acknowledged": true, "insertedCount": 2, "insertedIds": { "0": "...", "1": "..." } }
```

### 4. Оновити один документ: `updateOne()`

`PATCH /api/records/:id`

`id` — MongoDB ObjectId документа. Надсилаються тільки поля, які потрібно змінити:

```json
{ "price": 34.99, "inStock": true }
```

Відповідь `200 OK`:

```json
{ "acknowledged": true, "matchedCount": 1, "modifiedCount": 1 }
```

### 5. Оновити багато документів: `updateMany()`

`PATCH /api/records/bulk`

Тіло містить фільтр рівності `filter` і поля для оновлення `update`:

```json
{
  "filter": { "category": "accessories" },
  "update": { "onSale": true }
}
```

Відповідь `200 OK`:

```json
{ "acknowledged": true, "matchedCount": 3, "modifiedCount": 3 }
```

### 6. Замінити один документ: `replaceOne()`

`PUT /api/records/:id`

Заміна видаляє попередні поля документа й записує поля з тіла запиту (крім `_id`, який зберігається у фільтрі). Надсилайте повний документ, а не лише змінені поля:

```json
{ "name": "Updated Gaming Mouse", "category": "accessories", "price": 39.99 }
```

Відповідь `200 OK`:

```json
{ "acknowledged": true, "matchedCount": 1, "modifiedCount": 1 }
```

### 7. Видалити один документ: `deleteOne()`

`DELETE /api/records/:id`

Приклад:

```http
DELETE /api/records/671a1c2d3e4f506172839405
```

Відповідь `200 OK`:

```json
{ "acknowledged": true, "deletedCount": 1 }
```

### 8. Видалити багато документів: `deleteMany()`

`DELETE /api/records/bulk`

Тіло запиту має містити непорожній фільтр рівності. Обережно: видаляться всі документи, які відповідають фільтру.

```json
{ "filter": { "category": "temporary" } }
```

Відповідь `200 OK`:

```json
{ "acknowledged": true, "deletedCount": 4 }
```

## Типові помилки

- `400 Bad Request` — некоректний ObjectId, порожнє тіло або неправильний формат масиву/фільтра.
- `404 Not Found` — маршрут не існує (для HTML-сторінок застосовується наявна сторінка помилки).
- `401`/перенаправлення на `/login` — потрібно увійти в систему; поведінка визначається наявним middleware авторизації.
- `500 Internal Server Error` — помилка сервера або MongoDB; перевірте термінал, URI та мережевий доступ Atlas.

## Структура ключових файлів

```text
config/db.js                 # підключення до MongoDB Atlas через Mongoose
config/passport.js           # Passport Local
middleware/authMiddleware.js # захист маршрутів
controllers/crudController.js# CRUD-операції через MongoDB Driver
routes/crudRoutes.js         # маршрути /api/records
routes/userRoutes.js         # наявні сторінки користувачів
app.js                       # реєстрація middleware та маршрутів
server.js                    # запуск сервера
```

## Перевірка та публікація в GitHub

Перевірте всі маршрути в Postman або Insomnia після входу в систему. Перед публікацією переконайтеся, що `.env` не потрапив у коміт. Потім виконайте:

```bash
git add .
git commit -m "Add MongoDB CRUD API and documentation"
git push
```

Якщо локальна папка ще не прив'язана до репозиторію, налаштуйте remote окремо:

```bash
git remote add origin https://github.com/klejnigor5-byte/65les.git
```

Не виконуйте `git remote add`, якщо `origin` уже налаштований; перевірити його можна командою `git remote -v`.


## ДЗ 67.1 — MongoDB cursors та aggregation

Усі маршрути `/api/records` захищені наявною Passport session-авторизацією: спочатку увійдіть у вебзастосунок, а потім виконуйте запити в тій самій сесії або використовуйте клієнт, що зберігає cookie сесії.

### 1. Читання через курсор — `GET /api/records/stream`

Маршрут використовує MongoDB `find()` cursor та `for await...of`. Документи записуються у HTTP-відповідь поступово як JSON-масив; повний набір не накопичується в масиві в пам'яті сервера. Курсор закривається в `finally`.

Параметри запиту:
- `batchSize` — розмір пакета MongoDB, від 1 до 500, типово `100`.
- `limit` — максимальна кількість документів у відповіді, від 1 до 10000, типово `1000`.
- `fields` — необов'язковий список полів через кому, наприклад `name,category,amount`.

Приклад:
```http
GET /api/records/stream?batchSize=100&limit=1000&fields=name,category,amount
```

Приклад відповіді (поля залежать від документів у базі):
```json
[
  { "_id": "...", "name": "Example", "category": "books", "amount": 25 },
  { "_id": "...", "name": "Another", "category": "games", "amount": 60 }
]
```

### 2. Статистика через aggregation — `GET /api/records/stats`

Маршрут виконує pipeline з `$facet`, `$group`, `$sum`, `$avg`, `$match` та `$sort`. Він повертає загальну кількість документів, суму та середнє числового поля `amount`, а також кількість документів і суму `amount` для кожної категорії `category`.

```http
GET /api/records/stats
```

Приклад відповіді:
```json
{
  "collection": "crud_records",
  "totals": {
    "totalRecords": 3,
    "amountSum": 100,
    "amountCount": 3,
    "amountAverage": 33.3333333333
  },
  "categories": [
    { "category": "games", "count": 2, "amountSum": 75 },
    { "category": "books", "count": 1, "amountSum": 25 }
  ]
}
```

`amount` має бути числом, а `category` — рядком. Документи без числового `amount` враховуються в `totalRecords`, але не в `amountCount` і середньому. Документи без непорожньої категорії не потрапляють до списку `categories`. Значення в прикладі умовні.

### Тестування

1. Запустіть MongoDB Atlas-конфігурацію через `.env` і виконайте `npm run dev`.
2. Увійдіть у застосунок через форму входу, щоб отримати session cookie.
3. Створіть кілька записів через `POST /api/records`, наприклад:
```json
{ "name": "Game A", "category": "games", "amount": 50 }
```
```json
{ "name": "Book A", "category": "books", "amount": 25 }
```
4. Відкрийте `/api/records/stream?batchSize=1&limit=10` та перевірте, що повертається JSON-масив записів.
5. Відкрийте `/api/records/stats` і звірте `totalRecords`, `amountSum`, `amountAverage` та групи `categories` з даними у колекції.
6. Для перевірки оптимізації порівняйте `/api/records?limit=100` (звичайний `toArray()` для невеликого результату) з `/api/records/stream?batchSize=100&limit=1000` (поступова передача даних через курсор). Потоковий маршрут обмежує кількість документів параметром `limit`.

> Не додавайте `.env` до Git. У репозиторії має бути лише `.env.example` без реальних паролів та ключів.
