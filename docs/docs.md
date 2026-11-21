# Kiki Payment — qisqa yo'riqnoma / Краткая инструкция

**Base URL:** `https://kiki.salompremium.uz`

---

## 🇺🇿 O'zbekcha

### 1. Do'kon tokeni
Kiki'da do'kon yarating. Sizga **Shop ID** va **Shop Password** beriladi.
Ularni har so'rovda yuboring:

```
X-Kiki-Shop-Id: <SHOP_ID>
X-Kiki-Shop-Password: <PASSWORD>
```

> ⚠️ Parolni faqat serverda saqlang. Frontend, GitHub yoki chatga qo'ymang.

### 2. To'lov yaratish
`POST /payments/create`

```bash
curl -X POST "https://kiki.salompremium.uz/payments/create" \
  -H "X-Kiki-Shop-Id: <SHOP_ID>" -H "X-Kiki-Shop-Password: <PASSWORD>" \
  -H "Content-Type: application/json" \
  -d '{"amount": 50000, "external_order_id": "ORDER-123"}'
```

Javobdan mijozga ko'rsating: `card_number`, `card_holder`, `payable_amount`, `expires_at`.
Mijoz **`payable_amount`** ni to'lashi kerak (`amount` emas).

### 3. Holatni olish
- **Webhook** (tavsiya): do'konda webhook URL kiriting.
- Yoki `POST /payments/check` bilan `{"external_order_id": "ORDER-123"}`.

| Holat | Ma'nosi |
|---|---|
| `PENDING` | Kutilmoqda |
| `SUCCESS` | To'landi — buyurtmani tasdiqlang |
| `EXPIRED` | Muddat tugadi — yangi to'lov oching |
| `LATE_PAID` | Muddatdan keyin pul keldi — qo'lda tekshiring |

### 4. Webhook tekshiruvi
Har so'rovda `X-Kiki-Signature` ni tekshiring:

```python
import hmac, hashlib, time

def verify(secret, raw_body, signature, timestamp):
    if abs(time.time() - int(timestamp)) > 300:
        return False
    msg = timestamp.encode() + b"." + raw_body
    expected = "sha256=" + hmac.new(secret.encode(), msg, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

`secret` — webhook secret (do'kon sozlamalarida). Javob sifatida tez `200` qaytaring.

### 5. Xatolar
`{"error_code": "...", "message": "..."}`

| Kod | Nima qilish |
|---|---|
| `SHOP_CREDENTIALS_INVALID` (401) | Shop ID yoki parol noto'g'ri |
| `BOT_NOT_CONNECTED` (409) | Merchant to'lov botini ulamagan |
| `MIN_AMOUNT_NOT_MET` (422) | Summa minimumdan kam (standart 1000 so'm) |
| `AMOUNT_POOL_EXHAUSTED` (409) | Boshqa summa bilan yoki keyinroq qayta urining |

### Test rejimi
Test do'konda `"test": true` keladi — bu pulsiz "to'landi". Bunday to'lov uchun tovar bermang.

---

## Qo'shimcha ma'lumot
### 1. To'lov yaratish — barcha parametrlar
`POST /payments/create`

| Maydon | Turi | Majburiy | Izoh |
|---|---|---|---|
| `amount` | int | ha | So'mda, > 0. Do'kon minimumidan kam bo'lmasin (standart 1000) |
| `external_order_id` | string ≤128 | yo'q | Sizning buyurtma ID'ingiz. Tavsiya qilinadi |
| `expires_in_sec` | int | yo'q | To'lov muddati: **60–1800 soniya**. Berilmasa do'kon sozlamasi, u ham bo'lmasa **600** (10 daqiqa) |

### 2. Javob maydonlari
`/payments/create`, `/payments/check` va `GET /payments/{payment_id}` javoblarida bir xil. Webhook `data` da `expires_in_sec` va karta maydonlari bo'lmaydi:

| Maydon | Izoh |
|---|---|
| `payment_id` | Kiki to'lov ID'si |
| `shop_id` | Do'kon ID'si |
| `external_order_id` | Siz yuborgan ID (yoki `null`) |
| `requested_amount` | Siz so'ragan summa |
| `payable_amount` | **Mijoz aynan shuni o'tkazadi** (`amount` + 0…999 so'm) |
| `expires_at` | Muddat tugash vaqti, UTC (ISO 8601) |
| `expires_in_sec` | Qancha soniya qoldi (faqat REST javobida) |
| `status` | `PENDING` / `SUCCESS` / `EXPIRED` / `LATE_PAID` |
| `paid_at` | Pul kelgan vaqt (UTC) yoki `null` |
| `card_number`, `card_holder`, `card_last4` | Pul o'tkaziladigan karta (faqat REST javobida) |
| `test` | `true` — test do'kon, pul kelmagan |
| `resolved_by` | Faqat webhook'da: `auto` (Kiki o'zi topdi), `merchant` (sotuvchi qo'lda bog'ladi), `test` |

### 3. Qo'shimcha endpointlar
| So'rov | Nima qiladi |
|---|---|
| `GET /payments/{payment_id}` | To'lovni Kiki ID'si bo'yicha olish |
| `POST /payments/{payment_id}/check` | Xuddi shu, POST orqali |
| `GET /payments` | Ro'yxat. Parametrlar: `limit` (1–200, standart 50), `offset`, `status`, `external_order_id`, `created_from`, `created_to` |

Hammasi bir xil `X-Kiki-Shop-Id` / `X-Kiki-Shop-Password` headerlari bilan ishlaydi.

### 4. `external_order_id` qayta yuborilsa
| Avvalgi to'lov holati | Natija |
|---|---|
| `PENDING` | Yangi to'lov ochilmaydi, **o'sha to'lov qaytadi** (qayta urinish xavfsiz) |
| `SUCCESS` / `LATE_PAID` | `PAYMENT_ALREADY_COMPLETED` (409) |
| `EXPIRED` | `DUPLICATE_EXTERNAL_ORDER_ID` (409) |

➡️ Muddati tugagan to'lov o'rniga **yangi `external_order_id`** bilan yangi to'lov oching.

### 5. Holatlar qanday o'zgaradi
- To'lov oynasi: `expires_at` + **45 soniya** (bank xabari kechikishi uchun). Shu vaqt ichida kelgan pul → `SUCCESS`.
- Oyna yopilgach to'lov `EXPIRED` bo'ladi. Uning summasi yana **15 daqiqa** shu kartada band turadi.
- Shu 15 daqiqa ichida pul kelsa → `LATE_PAID`.
- Undan keyin kelgan pul hech qaysi buyurtmaga avtomatik bog'lanmaydi. Sotuvchi uni Kiki'da qo'lda bog'lashi mumkin → to'lov `SUCCESS` bo'ladi (`resolved_by: "merchant"`).

⚠️ Demak **`EXPIRED` yakuniy holat emas** — keyin `LATE_PAID` yoki `SUCCESS` bo'lishi mumkin. Webhook'ni yoki tekshiruvni muddat tugagandan keyin ham davom ettiring.

### 6. Webhook — to'liq format
**Headerlar:**
```
Content-Type: application/json
X-Kiki-Event: payment.paid
X-Kiki-Delivery: <delivery_id>
X-Kiki-Timestamp: <unix soniya>
X-Kiki-Signature: sha256=<hex>
```
(`X-Kiki-Signature` faqat do'konda webhook secret bo'lsa keladi.)

**Body:**
```json
{"event": "payment.paid", "delivery_id": "...", "data": { ...2-bo'limdagi maydonlar... }}
```

**Eventlar:**
| Event | Qachon |
|---|---|
| `payment.paid` | To'landi (avtomatik, qo'lda bog'langan yoki test) |
| `payment.late_paid` | Muddatdan keyin to'landi — qo'lda tekshiring |
| `payment.expired` | Muddat tugadi |

**Qayta yuborish:**
- Javobingiz **2xx** bo'lsa yetkazildi deb hisoblanadi. Boshqa kod yoki 10 soniyadan ortiq kutish → qayta yuboriladi.
- Kutish vaqti har safar oshadi (10 s, 20 s, 40 s … eng ko'pi 1 soat), jami **8 marta**.
- Bitta xabar bir necha marta kelishi mumkin — **`delivery_id` ni saqlab, takrorini o'tkazib yuboring**.
- Webhook URL **ochiq internet manzili** bo'lishi kerak (localhost / ichki IP qabul qilinmaydi).

### 7. Barcha xato kodlari (to'lov API)
| Kod | HTTP | Ma'nosi |
|---|---|---|
| `SHOP_CREDENTIALS_INVALID` | 401 | Shop ID yoki parol noto'g'ri |
| `SHOP_INACTIVE` | 403 | Do'kon o'chirilgan |
| `PAYMENT_NOT_FOUND` | 404 | Bunday to'lov yo'q |
| `BOT_NOT_CONNECTED` | 409 | Sotuvchining Telegram'i Kiki'ga ulanmagan, to'lov ochilmadi |
| `NO_CARD_AVAILABLE` | 409 | Do'konda faol karta yo'q |
| `AMOUNT_POOL_EXHAUSTED` | 409 | Bu summa uchun bo'sh noyob summa qolmadi |
| `DUPLICATE_EXTERNAL_ORDER_ID` | 409 | Bu ID bilan muddati tugagan to'lov bor |
| `PAYMENT_ALREADY_COMPLETED` | 409 | Bu ID allaqachon to'langan |
| `MIN_AMOUNT_NOT_MET` | 422 | Summa minimumdan kam |
| `PAYMENT_TIMEOUT_OUT_OF_RANGE` | 422 | `expires_in_sec` 60–1800 oralig'ida emas |
| `VALIDATION_ERROR` | 422 | So'rov formati noto'g'ri |
| `INTERNAL_SERVER_ERROR` | 500 | Kiki ichki xatosi — keyinroq qayta urining |

### 8. Test rejimi — aniqroq
Test do'konda `/payments/create` darhol `status: "SUCCESS"`, `test: true` qaytaradi va `payment.paid` webhook yuboradi. `payable_amount` = `amount` (qo'shimcha summa yo'q).

---

---

## 🇷🇺 Русский

### 1. Токен магазина
Создайте магазин в Kiki. Вы получите **Shop ID** и **Shop Password**.
Передавайте их в каждом запросе:

```
X-Kiki-Shop-Id: <SHOP_ID>
X-Kiki-Shop-Password: <PASSWORD>
```

> ⚠️ Храните пароль только на сервере.

### 2. Создать платёж
`POST /payments/create`

Из ответа покажите клиенту `card_number`, `card_holder`, `payable_amount`, `expires_at`.
Клиент должен перевести **`payable_amount`** (не `amount`).

### 3. Статус
- **Webhook** (рекомендуется) — укажите URL в настройках магазина.
- Или `POST /payments/check` с `{"external_order_id": "ORDER-123"}`.

| Статус | Значение |
|---|---|
| `PENDING` | Ожидает оплаты |
| `SUCCESS` | Оплачено — подтвердите заказ |
| `EXPIRED` | Срок истёк — создайте новый платёж |
| `LATE_PAID` | Оплата после срока — проверьте вручную |

### 4. Проверка webhook
Проверяйте `X-Kiki-Signature` (код — см. пример выше на Python, функция `verify`).
Отвечайте `200` быстро.

### 5. Ошибки
| Код | Что делать |
|---|---|
| `SHOP_CREDENTIALS_INVALID` (401) | Неверный Shop ID или пароль |
| `BOT_NOT_CONNECTED` (409) | Владелец не подключил платёжный бот |
| `MIN_AMOUNT_NOT_MET` (422) | Сумма ниже минимума (по умолчанию 1000) |
| `AMOUNT_POOL_EXHAUSTED` (409) | Выберите другую сумму или повторите позже |

### Тестовый режим
В тестовом магазине приходит `"test": true` — оплата без денег. Товар при этом не отдавайте.

---

## Дополнительно
### 1. Создание платежа — все параметры
`POST /payments/create`

| Поле | Тип | Обяз. | Описание |
|---|---|---|---|
| `amount` | int | да | В сумах, > 0, не меньше минимума магазина (по умолчанию 1000) |
| `external_order_id` | string ≤128 | нет | Ваш ID заказа. Рекомендуется |
| `expires_in_sec` | int | нет | Срок оплаты: **60–1800 сек**. Иначе — настройка магазина, иначе **600** (10 мин) |

### 2. Поля ответа
Одинаковы для `/payments/create`, `/payments/check` и `GET /payments/{payment_id}`. В `data` webhook нет `expires_in_sec` и полей карты:

| Поле | Описание |
|---|---|
| `payment_id` | ID платежа в Kiki |
| `shop_id` | ID магазина |
| `external_order_id` | Ваш ID (или `null`) |
| `requested_amount` | Запрошенная сумма |
| `payable_amount` | **Ровно эту сумму переводит клиент** (`amount` + 0…999 сум) |
| `expires_at` | Время окончания, UTC (ISO 8601) |
| `expires_in_sec` | Сколько секунд осталось (только в REST) |
| `status` | `PENDING` / `SUCCESS` / `EXPIRED` / `LATE_PAID` |
| `paid_at` | Время поступления денег (UTC) или `null` |
| `card_number`, `card_holder`, `card_last4` | Карта для перевода (только в REST) |
| `test` | `true` — тестовый магазин, денег не было |
| `resolved_by` | Только в webhook: `auto` (Kiki нашёл сам), `merchant` (продавец привязал вручную), `test` |

### 3. Дополнительные эндпоинты
| Запрос | Что делает |
|---|---|
| `GET /payments/{payment_id}` | Платёж по ID Kiki |
| `POST /payments/{payment_id}/check` | То же, через POST |
| `GET /payments` | Список. Параметры: `limit` (1–200, по умолч. 50), `offset`, `status`, `external_order_id`, `created_from`, `created_to` |

Все — с теми же заголовками `X-Kiki-Shop-Id` / `X-Kiki-Shop-Password`.

### 4. Повторный `external_order_id`
| Статус прежнего платежа | Результат |
|---|---|
| `PENDING` | Новый не создаётся, **возвращается тот же** (повтор безопасен) |
| `SUCCESS` / `LATE_PAID` | `PAYMENT_ALREADY_COMPLETED` (409) |
| `EXPIRED` | `DUPLICATE_EXTERNAL_ORDER_ID` (409) |

➡️ Вместо истёкшего платежа создайте новый с **новым `external_order_id`**.

### 5. Как меняется статус
- Окно оплаты: `expires_at` + **45 сек** (задержка банковского сообщения). Деньги в этом окне → `SUCCESS`.
- После окна платёж становится `EXPIRED`, его сумма ещё **15 минут** зарезервирована на карте.
- Деньги в эти 15 минут → `LATE_PAID`.
- Позже деньги автоматически не привязываются. Продавец может привязать вручную в Kiki → `SUCCESS` (`resolved_by: "merchant"`).

⚠️ **`EXPIRED` — не окончательный статус**: потом может стать `LATE_PAID` или `SUCCESS`. Продолжайте принимать webhook / проверять статус и после истечения срока.

### 6. Webhook — полный формат
**Заголовки:**
```
Content-Type: application/json
X-Kiki-Event: payment.paid
X-Kiki-Delivery: <delivery_id>
X-Kiki-Timestamp: <unix-секунды>
X-Kiki-Signature: sha256=<hex>
```
(`X-Kiki-Signature` — только если в магазине задан webhook secret.)

**Тело:**
```json
{"event": "payment.paid", "delivery_id": "...", "data": { ...поля из раздела 2... }}
```

**События:**
| Событие | Когда |
|---|---|
| `payment.paid` | Оплачено (автоматически, вручную или тест) |
| `payment.late_paid` | Оплачено после срока — проверьте вручную |
| `payment.expired` | Срок истёк |

**Повторы:**
- Доставлено, если ответ **2xx**. Другой код или ожидание > 10 сек → повтор.
- Интервал растёт (10 с, 20 с, 40 с … максимум 1 час), всего **8 попыток**.
- Одно событие может прийти несколько раз — **сохраняйте `delivery_id` и пропускайте повторы**.
- Webhook URL должен быть **публичным адресом** (localhost / внутренние IP не принимаются).

### 7. Все коды ошибок (платёжный API)
| Код | HTTP | Значение |
|---|---|---|
| `SHOP_CREDENTIALS_INVALID` | 401 | Неверный Shop ID или пароль |
| `SHOP_INACTIVE` | 403 | Магазин отключён |
| `PAYMENT_NOT_FOUND` | 404 | Платёж не найден |
| `BOT_NOT_CONNECTED` | 409 | Telegram продавца не подключён к Kiki, платёж не создан |
| `NO_CARD_AVAILABLE` | 409 | В магазине нет активной карты |
| `AMOUNT_POOL_EXHAUSTED` | 409 | Нет свободных уникальных сумм для этой суммы |
| `DUPLICATE_EXTERNAL_ORDER_ID` | 409 | С этим ID есть истёкший платёж |
| `PAYMENT_ALREADY_COMPLETED` | 409 | Этот ID уже оплачен |
| `MIN_AMOUNT_NOT_MET` | 422 | Сумма ниже минимума |
| `PAYMENT_TIMEOUT_OUT_OF_RANGE` | 422 | `expires_in_sec` вне 60–1800 |
| `VALIDATION_ERROR` | 422 | Неверный формат запроса |
| `INTERNAL_SERVER_ERROR` | 500 | Внутренняя ошибка Kiki — повторите позже |

### 8. Тестовый режим — подробнее
В тестовом магазине `/payments/create` сразу возвращает `status: "SUCCESS"`, `test: true` и отправляет webhook `payment.paid`. `payable_amount` = `amount` (без надбавки).
