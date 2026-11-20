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
