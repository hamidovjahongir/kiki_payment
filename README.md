# Kiki payment

MCHJ, YTT va kichik jamoalar uchun to‘lov jarayonini soddalashtiradi. MVP va tez ishga tushadigan loyihalar uchun qulay.

---

## Havolalar

| | |
|---|---|
| API | http://16.171.154.226:8000 |
| Swagger | http://16.171.154.226:8000/docs |
| Bot | [@kiki_payment_bot](https://t.me/kiki_payment_bot) |

> Avval [@HUMOcardbot](https://t.me/HUMOcardbot) da **Start** bosing.

---

## Tez start

```
Bot → Do'kon → create → mijoz to'laydi → check → SUCCESS
```

1. **Bot** — [@kiki_payment_bot](https://t.me/kiki_payment_bot) → Start → telefon → kod → kirish
2. **Do'kon** — Do'konlar → Do'kon qo'shish → **Shop ID** + **Password** oling
3. **API** — to'lov oching va tekshiring (pastda)

Ro'yxatdan o'tish va do'kon yaratish **faqat bot** orqali. Swaggerda faqat to'lov API ko'rinadi.

---

## API

Har bir so'rovda header:

```
X-Kiki-Shop-Id: sizning_shop_id
X-Kiki-Shop-Password: sizning_shop_password
```

### To'lov yaratish — `POST /payments/create`

Mijozga javobdagi **`payable_amount`** ni ko'rsating (so'ralgan summadan biroz farq qilishi mumkin — bu normal).

```bash
curl -X POST 'http://16.171.154.226:8000/payments/create' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password' \
  -H 'Content-Type: application/json' \
  -d '{"amount": 5000, "external_order_id": "buyurtma-123"}'
```

| Maydon | Izoh |
|--------|------|
| `amount` | Summa (min **5000** so'm) |
| `external_order_id` | Sizning buyurtma ID (masalan `buyurtma-123`) |

### To'lovni tekshirish — `POST /payments/check`

To'lov ekranida har **2–3 soniyada** chaqiring.

```bash
curl -X POST 'http://16.171.154.226:8000/payments/check' \
  -H 'X-Kiki-Shop-Id: sizning_shop_id' \
  -H 'X-Kiki-Shop-Password: sizning_shop_password' \
  -H 'Content-Type: application/json' \
  -d '{"external_order_id": "buyurtma-123"}'
```

**Status:** `PENDING` kutilmoqda · `SUCCESS` to'landi · `EXPIRED` vaqt tugadi

---

## Muhim

- Mijoz **payable_amount** ni aniq yuborishi kerak
- Minimal summa: **5000 so'm**
- To'lov muddati: **~2 daqiqa**
- Faqat HUMO **to'ldirish** (➕) hisobga olinadi

Boshqa endpointlar (Swaggerda): `GET /payments`, `GET /payments/{id}`

---

## Yordam

[@kiki_payment_bot](https://t.me/kiki_payment_bot) → **Yordam**
