# Bankimun Panel — TAM Güvenlik Denetimi (birleşik, satır-satır)

> v3.292.0 · Next.js 14.2.21 · 95 API route + 37 lib dosyası.
> Bu, önceki güvenlik raporu + son turdaki + satır-satır geçişte çıkan **tüm** bulguların birleşik halidir.

## Kapsam & dürüst metodoloji
**Satır satır okundu:** middleware + `lib/auth`, `lib/db`, müşteri auth zinciri (login/register/verify/reset/me/update/addresses/OAuth), TÜM ödeme akışları (checkout stripe+paypal, pay-link token, webhook, invoice payment), `lib/coupons`+`quoteOrder`, `lib/orders` (createOrder/markPaid/refund/payLink/token), `lib/customers` kritik fonksiyonlar, `lib/cloudinary`+tüm media route'ları, `lib/email`, `lib/csv`+import, `lib/invoices` token/void, forms/submit, subscribe, finansal admin route'ları (refund/mark-paid/bulk/manual/record-payment/void), `next.config`.
**Gating düzeyinde doğrulandı (derin okunmadı):** basit admin CRUD route'ları (products/[id], categories, shipping, slider, pages, settings, subscribers, carriers) — hepsi middleware-korumalı; saf sunum lib'leri (theme, svgTree, sliderHelpers, useDragList, regions, segments, tracking, stock, shipping) — auth/ödeme/girdi-güveni yüzeyi yok.
**Yapmadığım:** canlı sızma testi (exploit denemesi), `npm audit` (Next dışı CVE), runtime header testi.

---

## Özet tablosu

| # | Bulgu | Önem | Durum |
|---|---|---|---|
| 1 | Admin oturum cookie'si taklit edilebilir (`bk_admin=1`) | 🔴 KRİTİK | ilk rapor |
| 2 | **Next.js 14.2.21 — CVE-2025-29927 middleware bypass** | 🔴 KRİTİK (koşullu) | son tur |
| 3 | `/api/contacts/*` (12 uç) tamamen korumasız | 🔴 KRİTİK | doğrulandı (tek gap) |
| 4 | Checkout fiyat manipülasyonu (item.price client'tan) | 🔴 KRİTİK | ilk rapor |
| 5 | Stripe webhook: secret yoksa imzasız + sıralı sipariş no | 🟠 YÜKSEK | ilk rapor |
| 6 | **Host-header injection → şifre-sıfırlama linki zehirleme** | 🟠 YÜKSEK | YENİ |
| 7 | Rate-limit kapsamı yok (e-posta bomba / kupon enum / spam) | 🟠 YÜKSEK | son tur (genişletildi) |
| 8 | `ADMIN_PASSWORD` yoksa varsayılan `"bankimun"` | 🟠 YÜKSEK | ilk rapor |
| 9 | Tek-blob JSONB → eşzamanlı yazımda veri kaybı | 🟠 YÜKSEK | ilk rapor |
| 10 | `SESSION_SECRET` yoksa sabit `"bk-dev-secret"` | 🟡 ORTA | ilk rapor |
| 11 | `payLinkToken` zayıf (`Math.random`) = login'siz sipariş erişimi | 🟡 ORTA | genişletildi |
| 12 | **Cloudinary cloud adı + API key kaynak koda gömülü** | 🟡 ORTA | YENİ |
| 13 | **CSV export formül enjeksiyonu** (müşteri alanları) | 🟡 ORTA | YENİ |
| 14 | **Güvenlik header'ları yok** (clickjacking/CSP/HSTS) | 🟡 ORTA | YENİ |
| 15 | **`register` kullanıcı enumerasyonu** (`email-taken`) | 🟡 ORTA | YENİ |
| 16 | Küçükler: 2× dangerouslySetInnerHTML, Maps key, 213 sessiz catch, zayıf subscriber email doğrulama, `===` parola karşılaştırma | 🟢 DÜŞÜK | karışık |

---

## 🔴 KRİTİK

### 1. Admin oturum cookie'si taklit edilebilir
`bk_admin=1` düz değer, imza yok. `curl -H "Cookie: bk_admin=1"` ile tüm panel + korumalı API açılır. `httpOnly` korumaz (saldırgan cookie'yi kendi gönderiyor). **Fix:** HMAC imzalı + süreli token — kod önceki `AUDIT.md`'de (lib/auth.js Node + middleware Web Crypto). Müşteri tarafı zaten imzalı; admin'i ona eşitle.

### 2. Next.js CVE-2025-29927 — middleware authorization bypass
Sürüm **14.2.21**; 14.x için yama **14.2.25**. Saldırgan `x-middleware-subrequest: middleware` header'ı göndererek middleware'i **tamamen baypas** edebiliyor → tüm admin auth + `/api/products|shipping|slider|...` koruması düşer.
**Koşul:** sadece **self-hosted** (`next start` + `output:'standalone'`) etkilenir. **Vercel'de barındırıyorsan bu CVE seni vurmuyor** (routing decoupled). Self-hosted'san KRİTİK.
**Fix:** `npm i next@14.2.25` (veya 14.2.x'in en günceli) — tek satır. Geçici: proxy/WAF'ta `x-middleware-subrequest` header'ını strip et. Bu CVE, #1 ve #3'ün de neden tehlikeli olduğunu kanıtlıyor: **auth'u yalnız middleware'e bırakma; route'lara da kontrol koy** (resmi azaltma önerisi).

### 3. `/api/contacts/*` tamamen korumasız (tek auth gap)
12 ucun hiçbiri middleware PROTECTED'da değil ve `isAuthed()` çağırmıyor: `delete, update, manual, notes, block, hide, tag, labels, pinned, segments, set-member, import`. Login'siz müşteri kaydı **silinebilir/değiştirilebilir**, sahte lead basılabilir. Tüm route taramasında **korumasız kalan tek namespace bu** (geri kalan her şey kapalı).
**Fix:** ya `PROTECTED`'a `/api/contacts` ekle (tek satır), ya da daha sağlamı — modeli ters çevir (tüm `/api/*` varsayılan korumalı, açık PUBLIC listesi serbest). Kod önceki `AUDIT.md`'de.

### 4. Checkout fiyat manipülasyonu
`quoteOrder`/`createOrder` satır fiyatını client `item.price`'tan alıyor; sunucu katalogdan doğrulamıyor. `{"id":"p-sf3","price":0.51}` ile pahalı ürün 51 sente alınır. **Fix:** `quoteOrder` başında `priceFor(product, variant)` ile yeniden fiyatla — kod önceki `AUDIT.md`'de. (Not: `orders/manual` admin-only olduğu için orada client fiyatı **kasıtlı** — sorun değil.)

---

## 🟠 YÜKSEK

### 5. Stripe webhook imzasız fallback + sıralı no
`STRIPE_WEBHOOK_SECRET` yoksa `JSON.parse(body)` ile imzasız kabul → sahte `payment_intent.succeeded` ile sipariş "paid". Sipariş no'ları `BK-10001…` sıralı → hedef bulmak kolay. **Fix:** secret yoksa 503 dön, asla imzasız kabul etme.

### 6. Host-header injection → şifre-sıfırlama linki zehirleme (YENİ)
`forgot-password` ve `account/update` (email-change) maildeki linki `req.headers.get("origin") || \`https://${req.headers.get("host")}\`` ile kuruyor:
```js
const origin = req.headers.get("origin") || `https://${req.headers.get("host")}`;
const link = `${origin}/reset-password?token=${r.token}`;
```
`Host`/`Origin` header'ını enjekte edebilen bir proxy arkasındaysan, saldırgan `Host: evil.com` ile **kurbanın şifre-sıfırlama linkini saldırganın domainine** yönlendirir → kurban tıklayınca token evil.com'a sızar → hesap ele geçirme. **Fix:** link tabanını request header'ından TÜRETME; sabit env kullan — `NEXT_PUBLIC_SITE_URL` zaten tanımlı:
```js
const origin = process.env.NEXT_PUBLIC_SITE_URL || `https://${req.headers.get("host")}`;
```
(Vercel'de Host genelde sabit/güvenli, ama self-hosted + yanlış proxy'de gerçek risk; env'e geçmek her durumda doğru.)

### 7. Rate-limit kapsamı yok (genişletildi)
`lib/ratelimit.js` var ama **sadece admin + müşteri login**'de kullanılıyor. Diğer hassas uçlarda yok:
- `forgot-password` → **şifre-sıfırlama maili bombalama** (cooldown bile yok — aynı kurbana sınırsız).
- `send-code` → per-email cooldown VAR ama IP-başı/global YOK → farklı adreslere bomba.
- `register`, `register-resend` → hesap/mail spam.
- `coupons/validate` (public) → **kupon kodu enumerasyonu** (binlerce kod dene).
- `forms/submit` + `subscribe` (public) → form/bülten spam (honeypot+timing var ama IP-limit yok).
**Fix:** `checkLimit` (zaten yazılı) bu uçlara da, özellikle public olanlara IP-başına uygula.

### 8. `ADMIN_PASSWORD` varsayılanı `"bankimun"`
`process.env.ADMIN_PASSWORD || "bankimun"` — env unutulursa panel bilinen parolayla açılır. **Fix:** fallback'i kaldır; env yoksa giriş başarısız olsun (+ prod build'de `throw`).

### 9. Tek-blob JSONB → eşzamanlı yazımda veri kaybı
Her tür (store/orders/invoices/customers/forms) tek JSONB satırı; her yazma oku-değiştir-yaz. Vercel'de eşzamanlı iki istek birbirini ezer → kayıp sipariş/stok/kupon. **Fix:** atomik `jsonb_set` (tek sipariş append) ya da `version` ile optimistic locking; uzun vade satır-başına kayıt.

---

## 🟡 ORTA

### 10. `SESSION_SECRET` fallback `"bk-dev-secret"`
İkisi de yoksa müşteri oturum imzası sabit secret'la → token taklit (kurbanın `cust-` id'sini bilmek gerekir). **Fix:** ayrı zorunlu `SESSION_SECRET`, fallback yok.

### 11. `payLinkToken` zayıf = login'siz sipariş erişimi
`pay/[token]` GET **public** ve token'la korunuyor; ama token `Date.now()+Math.random` (~31 bit, crypto YOK). (Karşılaştırma: order/invoice `viewToken` = `crypto.randomUUID` → güçlü; sadece payLinkToken zayıf.) Tahmin/enumerasyonla sipariş özeti (ürünler, tutarlar, müşteri adı) görülebilir + ödenebilir. **Fix:** `crypto.randomBytes(24).toString("hex")`.

### 12. Cloudinary kimlik bilgileri kaynak koda gömülü (YENİ)
`lib/cloudinary.js`:
```js
export const CLOUD_NAME = process.env.CLOUDINARY_CLOUD_NAME || "do6zuvjhe";
export const API_KEY = process.env.CLOUDINARY_API_KEY || "352857239531564";
```
Cloud adı + API key gerçek-görünümlü değerlerle hardcoded (secret değil — o `""`). Repo public olursa sızar; ayrıca `/api/media` GET bunları döndürüyor (admin-only ama). **Fix:** fallback'leri kaldır, sadece env. (Medya route'larının hepsi admin-only ✓ — başka media sorunu yok; tek not: `media/sign` dosya tipi/boyut kısıtlamıyor, `media/delete` ROOT_FOLDER'a scope'lu değil — admin-trusted, düşük.)

### 13. CSV export formül enjeksiyonu (YENİ)
`OrdersView`'daki `csvCell` sadece `",\n` kaçırıyor; baştaki `= + - @` nötrlemiyor. Müşteri adına/adresine/form alanına `=HYPERLINK("http://evil")` yazarsa, admin export edip Excel'de açınca **formül çalışır**. **Fix:** export'ta `^[=+\-@\t\r]` ile başlayan hücreye `'` (tek tırnak) ön-ek koy.

### 14. Güvenlik header'ları yok (YENİ)
`next.config.mjs`'te CSP/HSTS/X-Frame-Options/X-Content-Type-Options/Referrer-Policy hiç yok → **clickjacking** (admin iframe'lenebilir), zayıf XSS savunma derinliği. **Fix:**
```js
async headers() {
  return [{ source: "/:path*", headers: [
    { key: "X-Frame-Options", value: "DENY" },
    { key: "X-Content-Type-Options", value: "nosniff" },
    { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
    { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
    { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
  ]}];
}
```

### 15. `register` kullanıcı enumerasyonu (YENİ)
`startRegistration` mevcut email'de `email-taken` dönüyor → saldırgan hangi email'lerin kayıtlı olduğunu öğrenir. (`forgot-password` enumerasyon-güvenli yapılmış ama `register` değil — tutarsız.) **Fix:** ya kayıtta da generic mesaj/akış, ya da en azından rate-limit (7).

---

## 🟢 DÜŞÜK
- **`dangerouslySetInnerHTML` (2):** `layout.js` themeCss (admin tema), `EmailTemplatesEditor` önizleme (admin) — tema/şablon değerlerini sanitize et.
- **Google Maps key** (`NEXT_PUBLIC_…`): Google Console'da HTTP-referrer kısıtlaması koy.
- **~213 sessiz `catch {}`:** ödeme/mail/stok civarında sessiz başarısızlık; kritik yollarda `console.error` ekle.
- **`addSubscriber` email doğrulaması** sadece `includes("@")` — zayıf (newsletter, düşük etki).
- **`MAIL_FROM` fallback** `onboarding@resend.dev` (deliverability).
- **`passwordOk` `===` ile karşılaştırıyor** (timing) — rate-limit var, düşük; yine de `timingSafeEqual` öner.

---

## ✅ DOĞRULANDI — SAĞLAM (dokunma, bozma)

Satır-satır okudum, bunlar **doğru** yapılmış:
- **IDOR yok:** `account/addresses` müşterinin kendi token-id'sine scope'lu; `orders/[id]` admin-only. Başkasının verisine erişim yok.
- **Mass-assignment yok:** `updateCustomer`/`updateContactInfo`/`addManualContact` **açık whitelist** (keyfi alan enjekte edilemez).
- **OAuth (Google) sağlam:** state/CSRF cookie + `verified_email` kontrolü + redirect origin-prefix'li (open-redirect yok); account linking verified-email ile güvenli.
- **Doğrulama kodu brute-force korumalı:** mantık katmanında **5 deneme limiti + TTL** (6 haneli kod kırılamaz).
- **E-posta güvenli:** Resend API (JSON gövde → header injection yok), HTML'e giren tüm user değerleri `esc()` ile kaçırılıyor.
- **order/invoice `viewToken` = `crypto.randomUUID`** (güçlü). Reset token `randomBytes(24)` + **1 saat TTL** + kullanınca temizleniyor; reset enumerasyon-güvenli.
- **Müşteri parolaları:** scrypt + per-user salt + `timingSafeEqual`.
- **Stripe/PayPal sunucu doğrulaması:** checkout + pay-link tutarları `quoteOrder`/`amountDueOf` ile **sunucudan** (pay-link tarafında #3 yok); confirm/capture'da provider'dan retrieve + orderId/custom eşleşmesi + status kontrolü.
- **SQL parametreli** (neon tagged templates) — SQL injection yok.
- **Finansal admin route'ları** (refund/mark-paid/bulk/manual/record-payment/void) hepsi `isAuthed()` + makul doğrulama; method/action whitelist'leri var.
- **forms/submit:** honeypot (`hp`) + timing (`elapsed`) anti-spam.

---

## Önerilen sıra (etki × kolaylık)

1. **#2 Next.js 14.2.25'e yükselt** (tek satır, KRİTİK) + Vercel'de misin teyit et.
2. **#3 contacts'a auth** (tek satır PROTECTED, sonra modeli ters çevir).
3. **#1 admin cookie imzası** (HMAC) — #2 bypass'ına karşı da kalıcı savunma.
4. **#4 server-side reprice** (e-ticaret için şart).
5. **#6 link'leri SITE_URL'e + #5 webhook secret zorunlu + #8 #10 fallback kaldır + #11 token + #14 header'lar** (env/küçük dokunuşlar, hızlı, hepsi kod yukarıda).
6. **#7 rate-limit** hassas/public uçlara + **#13 CSV neutralize** + **#12 cloudinary env** + **#15 register enum**.
7. **#9 eşzamanlılık** (sipariş/stok atomik) — hacim artmadan.
8. Küçükler (#16) + `npm audit` çalıştır (Next dışı bağımlılıklar).
