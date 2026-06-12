# Kiki payment

Asosiy maqsad: **MCHJ**, **YTT** va kichik jamoalar uchun to‘lov jarayonini soddalashtirish. Ayniqsa **MVP loyihalar** va tez ishga tushirish kerak bo‘lgan sayt/ilovalar uchun qulay.

---

## Havolalar

| Nima | Manzil |
|------|--------|
| API (base URL) | http://16.171.154.226:8000 |
| Swagger (API hujjati) | http://16.171.154.226:8000/docs |
| Telegram bot | [@kiki_payment_bot](https://t.me/kiki_payment_bot) |
---

## 1-qadam — Bot orqali ro‘yxatdan o‘tish

Barcha ro‘yxatdan o‘tish va kirish **faqat Telegram bot** orqali.

1. [@kiki_payment_bot](https://t.me/kiki_payment_bot) ni oching.
2. **Start** yoki `/start` bosing.
3. Tilni tanlang (O‘zbek / Rus / Ingliz).
4. Telefon raqamingizni yuboring.
5. Telegramga kelgan kodni kiriting.
6. Agar 2FA yoqilgan bo‘lsa — qo‘shimcha parolni ham kiriting.
7. Muvaffaqiyatli kirgandan keyin asosiy menyu ochiladi.

> Yangi foydalanuvchi — ro‘yxatdan o‘tadi.  
> Oldin kirgan bo‘lsangiz — mavjud akkauntga kirasiz.

---

## 2-qadam — Do‘kon (shop) yaratish

1. Botda **Do‘konlar** tugmasini bosing.
2. **Do‘kon qo‘shish** ni tanlang.
3. Do‘kon nomini yozing (masalan: `Mening internet do'konim`).
4. Bot sizga beradi:
   - **Shop ID** — do‘kon identifikatori
   - **Password** — API uchun maxfiy parol

Bu ma’lumotlarni xavfsiz joyda saqlang. Sayt yoki ilovangiz har bir API so‘rovida ularni ishlatadi.

> Shop yaratish va boshqarish ham **faqat bot** orqali. Swaggerda bu qismlar ko‘rinmaydi — ataylab yashirilgan.

---

### 3-qadam -  To‘lov yaratish

**URL:** `POST /payments/create`

**Tavsif:** Yangi to‘lov ochish. Javobda `payable_amount` va `expires_in_sec` keladi — ularni mijozga ko‘rsating.

**Misol:**

```bash
curl -X POST 'http://16.171.154.226:8000/payments/create' \
  -H 'accept: application/json' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password' \
  -H 'Content-Type: application/json' \
  -d '{
    "amount": 5000,
    "external_order_id": "buyurtma-123"
  }'
```

**So‘rov maydonlari:**

| Maydon | Turi | Majburiy | Izoh |
|--------|------|----------|------|
| `amount` | integer | Ha | So‘ralgan summa (so‘m), minimal 5000 |
| `external_order_id` | string | Yo‘q | O‘z buyurtma ID ingiz (tavsiya etiladi) |

**Namuna javob:**

```json
{
  "payment_id": "abc123...",
  "shop_id": "shop_xyz",
  "external_order_id": "buyurtma-123",
  "requested_amount": 5000,
  "payable_amount": 5003,
  "expires_at": "2026-06-12T15:30:00",
  "expires_in_sec": 120,
  "status": "PENDING",
  "paid_at": null
}
```

Mijozga **`payable_amount`** ni ko‘rsating — masalan: **5003 so‘m**.

---

### 4.2. To‘lovni tekshirish (tavsiya etiladi)

**URL:** `POST /payments/check`

**Tavsif:** `external_order_id` bo‘yicha to‘lov holatini tekshiradi. HUMO xabarlarini skanerlab, mos kelgan to‘lovni `SUCCESS` qiladi.

**Misol:**

```bash
curl -X POST 'http://16.171.154.226:8000/payments/check' \
  -H 'accept: application/json' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password' \
  -H 'Content-Type: application/json' \
  -d '{
    "external_order_id": "buyurtma-123"
  }'
```

**So‘rov maydonlari:**

| Maydon | Turi | Majburiy | Izoh |
|--------|------|----------|------|
| `external_order_id` | string | Ha | Yaratishda bergan buyurtma ID |

**Integratsiya maslahati:** foydalanuvchi to‘lov ekranida tursa, har 2–3 soniyada shu endpointni chaqiring. `SUCCESS` bo‘lsa — buyurtmani tasdiqlang. `EXPIRED` bo‘lsa — yangi to‘lov oching.

---

### 4.3. Bitta to‘lovni ID bo‘yicha olish

**URL:** `GET /payments/{payment_id}`

**Tavsif:** `payment_id` bo‘yicha to‘lov ma’lumotini qaytaradi (tekshirmaydi, faqat o‘qiydi).

**Misol:**

```bash
curl -X GET 'http://16.171.154.226:8000/payments/abc123...' \
  -H 'accept: application/json' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password'
```

---

### 4.4. To‘lovlar ro‘yxati

**URL:** `GET /payments`

**Tavsif:** Do‘koningizdagi to‘lovlar ro‘yxati. Filtrlar ixtiyoriy.

**Misol:**

```bash
curl -X GET 'http://16.171.154.226:8000/payments?limit=20&offset=0&status=PENDING' \
  -H 'accept: application/json' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password'
```

**Query parametrlar (ixtiyoriy):**

| Parametr | Izoh |
|----------|------|
| `limit` | Nechta yozuv (1–200, default 50) |
| `offset` | Sahifalash uchun offset |
| `status` | `PENDING`, `SUCCESS` yoki `EXPIRED` |
| `external_order_id` | Aniq buyurtma bo‘yicha qidirish |
| `created_from` | Dan (ISO vaqt) |
| `created_to` | Gacha (ISO vaqt) |

---

### 4.5. To‘lovni ID bo‘yicha tekshirish (eski usul)

**URL:** `POST /payments/{payment_id}/check`

**Tavsif:** `payment_id` orqali tekshiradi. Yangi integratsiyalar uchun `POST /payments/check` ni ishlating — u `external_order_id` bilan qulayroq.

**Misol:**

```bash
curl -X POST 'http://16.171.154.226:8000/payments/abc123.../check' \
  -H 'accept: application/json' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password'
```
---

## Aloqa

Savollar va yordam: [@kiki_payment_bot](https://t.me/kiki_payment_bot)
