# Bankimun — TAM Görsel & Tasarımsal Denetim (eksiksiz liste)

> `app/globals.css` (2009 satır) + 67 JSX component **tek tek** tarandı (OrderDetail, ContactsView, NewOrderForm, InvoicesView, OrdersView, AppearanceEditor, InvoiceForm dahil okundu).
> Sayılar koddan ölçüldü. ⚠️ Kod-türevi olanlarda kesin görsel etkiyi ekranda da doğrula.
> En sonda **"kasıtlı / bug değil"** bölümü var — onlarla vakit kaybetme.

---

## ANA LİSTE (özet)

| # | Başlık | Önem | Kanıt |
|---|---|---|---|
| A1 | Radius ölçeği yok | 🟠 | 8 standart-dışı değer ~190× |
| A2 | Font-size ölçeği yok | 🟠 | **40 farklı boyut** |
| A3 | Spacing (padding/gap) ölçeği yok | 🟠 | 185 padding, 19 gap |
| A4 | Box-shadow ölçeği yok | 🟡 | 21 farklı gölge |
| A5 | Opacity (muted/disabled) ölçeği yok | 🟡 | 10 farklı değer |
| A6 | Border genişliği karışık | 🟡 | 1px/.5px/2px/1.5px |
| A7 | İkon boyutu karışık (özellikle X) | 🟡 | 12 genel · X kapatma **7 boyut** |
| A8 | Renk: token-dışı + grey zoo + inline | 🟠 | CSS 160 + inline 103 hardcoded |
| B1 | z-index sistemi yok (3 kopuk bant) | 🟠 | 15+ değer |
| B2 | Inline z-index hack'leri (tutarsız) | 🟡 | OrderDetail `zIndex:320` (8'in 2'sinde) |
| C1 | ~26 buton ailesi | 🟠 | aşağıda |
| C2 | Aynı rol farklı ölçü (toolbar/CTA) | 🟠 | radius 6/8/9/10, font 10–14 |
| C3 | Hover dili 5 farklı | 🟡 | border/renk/opacity karışık |
| C4 | Buton migration'ı yarım | 🟡 | inv-seg/ct-tbtn/bulkbtn hâlâ canlı |
| D1 | Kanonik `.modal` azınlıkta | 🟠 | 5 dosya vs ord-modal 9 + diğerleri |
| D2 | Modal genişlikleri inline, ölçek yok | 🟡 | 420/440/460/480/560 |
| D3 | Bazı modallarda `max-height`/scroll yok | 🟠 | kısa ekranda taşar |
| D4 | Modal dikey hizalama tutarsız | 🟡 | `.modal` üst, diğerleri orta |
| D5 | Backdrop/gölge modalden modale değişiyor | 🟡 | opaklık/blur/shadow |
| E1 | Menüler 4 farklı yöne açılıyor | 🟠 | right/left/full/rightward/up |
| E2 | Magic `top:34px/40px` offset'ler | 🟡 | trigger boyu değişince kayar |
| E3 | Menü radius'u karışık (10 vs 12) | 🟢 | ord-pop/s2-flyout 12 |
| E4 | Dropdown'lar portal kullanmıyor | 🟡 | kesilme + stacking riski |
| F1 | Drawer genişlikleri farklı | 🟡 | 340/420/440 |
| F2 | Drawer border + slide-mekanizması 3 farklı | 🟡 | transition vs 2 keyframe |
| G1 | Arka plan scroll kilidi YOK | 🟠 | body overflow set edilmiyor |
| G2 | Modalların çoğu Escape ile kapanmıyor | 🟠 | 2 dosya Escape dinliyor |
| G3 | Focus-trap yok | 🟡 | 0 component |
| G4 | Back-button davranışı tutarsız | 🟡 | sadece ContactsView pushState |
| H1 | Animasyon süreleri karışık | 🟢 | 10 farklı süre |
| H2 | Tekrarlanan keyframe'ler | 🟢 | 3 spinner, 2 slide, 2 pop |
| I1 | Responsive breakpoint kaosu | 🟠 | 10 kırılım + sözdizimi |
| J1 | Devasa component'ler | 🟡 | OrderDetail 1276/50 state, ContactsView 1200/57 |
| J2 | `money`/format helper'ı 8× tekrar | 🟡 | `lib/format.js` varken |
| J3 | 1401 inline style (token baypas) | 🟠 | 59 dosya |
| J4 | Başlık (`heading`) kullanımı tutarsız | 🟡 | 43 classed vs 77 ham h1-3 |
| K1 | ~170 tıklanabilir div/span, role yok | 🟠 | klavye/ekran-okuyucu |
| K2 | Kontrast: tx3/surf2 = 4.26 (<4.5) | 🟡 | küçük metin AA kalıyor |
| L1 | Storefront ayrı modal/toast/drawer | 🟡 | bk-modal/bk-toast z60 |

---

## A. TASARIM TOKEN'LARI (enforce edilmemiş) — kök sebep

DESIGN-STANDARD token tablosu var ama **CSS bu token'lara uymamış.** Boyut/aralık/renk hep ham değerle yazılmış. Bu, aşağıdaki tutarsızlıkların hepsinin anasıdır.

**A1 — Radius:** standart 10/14/16/999/6(chip) dururken: `8px`(53), `9px`(33), `7px`(24), `12px`(24), ayrıca 5/4/3/2/11/13/20 → **8 standart-dışı değer, ~190 kullanım**. Token: `--r-sm:10 --r-card:14 --r-modal:16 --r-pill:999 --r-chip:6`; 7/8/9/12'leri en yakına indir.

**A2 — Font-size: 40 farklı boyut** — 13(123), 12(83), 14(65), 11(55), 10(28), 12.5(23), 15(19), 13.5(18), 16(15), 11.5(12), 9/10.5/17/18/20/24/26/30… Radius'tan beter. Bir ölçek belirle (örn. 11/12/13/14/16/18/24) → `--fs-*` token, gerisini yuvarla.

**A3 — Spacing: 185 farklı padding değeri, 19 farklı gap** (gap: 8/10/6/12/9/4/5/14…). 4-8'lik bir step ölçeği (4/6/8/10/12/16/20/24) yok. Token + `gap`/`padding`'leri ona çek.

**A4 — Box-shadow: 21 farklı** gölge tanımı (`0 8px 24px`, `0 8px 28px`, `0 14px 38px`, `0 20px 60px`, `0 24px 70px`, `0 10px 30px`…). 3-4 seviyelik ölçek yeter (`--sh-menu`, `--sh-modal`, `--sh-pop`).

**A5 — Opacity (muted/disabled/hover): 10 değer** — .4/.45/.5/.6/.7/.85/.92/.95. Aynı "soluklaştırma" işi için .4 vs .45 vs .5. 2-3 token (`--o-muted:.6`, `--o-disabled:.4`).

**A6 — Border genişliği:** `1px`(128) / `.5px`(87) / `2px`(8) / `1.5px`(5) karışık — modallar 1px, card'lar .5px, bazıları 1.5/2. Tek hairline değeri seç (`.5px` veya `1px`).

**A7 — İkon boyutu: 12 farklı** (15/14/18/16/13/11/12/20/17/28/26/32). Standart `size=15` diyor. **Özellikle kapatma X'i 7 farklı boyutta:** `size={18}`(54×), 14(13×), 12(4×), 11(3×), 15(2×), 20(1×), 16(1×). Modal kapatma X'ini tek boyuta sabitle (öneri 16 veya 18).

**A8 — Renk (en dağınık):**
- CSS'te **160 hardcoded hex** (var 1260). Token DEĞERİ elle yazılmış: `#3c2a80`(brand)6×, `#cdb2f0`(acc2)5×, `#2a1f4d`(bd)10×, `#b67bd2`(acc)3×, `#fac775`(amber)4×, `#241a45`(surf2)3× → token değişirse kalır.
- Token-OLMAYAN renkler: **`#1a1030` 27×** (kaçak koyu zemin), `#1a1a2e` 6×, `#5b9bff` 4×.
- Inline style içinde **103 hardcoded renk** — özellikle **token'da olmayan grey zoo:** `#999`(8), `#555`(8), `#888`(6), `#666`(4), `#445`(4), `#777`(2). Bunlar `tx2`/`tx3` olmalı.

---

## B. KATMAN / Z-INDEX

**B1 — Sistem yok, 3 kopuk bant:**
- Düşük (5–60): `catmenu`(5), `rowmenu`/`bulkmenu`/`trk-cmenu`(20), `ov-trk-menu`(30), `ord-menu`/`ord-pop`(51), `tsel-menu`/`cp-more-menu`/`inv-fby-menu`/`ct-addfield-menu`(50), `ct-create-menu`(60), storefront `bk-drawer`(50)/`bk-toast`(60)/`s2-flyout`(50)/`s2-flyout2`(60)
- Orta (200–230): `ct-drawer`/`ct-push`(200), `md-bg`(210), `cp-lblpick-menu`(216), `md-box`(230)
- Yüksek (1000–3100): `modal-bg`/`ord-modal-wrap`/`uppanel`/`mlib-overlay`(1000), `inv-modal-bg`/`ct-edit-wrap`/`mlib-confirm`(1100), `bk-modal-ov`(2000), `admin-toast`(3100)

Aynı iş (açılır menü) için `catmenu` **z5** ile `cp-lblpick-menu` **z216** arası. `bk-toast` z60 storefront modalının (z2000) altında kalır. Tek token ölçeği: `--z-dropdown:1000 / --z-sticky:1100 / --z-drawer:1200 / --z-modal:1300 / --z-popover:1400 / --z-toast:1500` (portal'lı dropdown'lar modaldan yüksek olmalı).

**B2 — Inline z-index hack'leri:** `OrderDetail.jsx`'te 8 modaldan **2'si** `style={{ zIndex: 320 }}` ile override (payModal:636, share:690), diğer 6'sı default 1000 → aynı dosyada modal katmanları tutarsız; z320 olanlar başka bir z1000 overlay'in altında kalabilir.

---

## C. BUTONLAR

**C1 — ~26 aile:** `btn` · `icon-btn`/`iconbtn` · `linklike` · `seg`/`seg-btn` · `auth-btn` · `ck-gate-btn` · `no-custbtn` · `et-btn` · `apf-btn` · `pagebtn` · `pg-headbtn` · `inv-seg` · `inv-filter-btn`/`inv-fby-btn` · `ct-tbtn`/`ct-bulk-btn` · `bulkbtn` · `eb-seg`/`eb-seg-full`/`eb-ebtn` · `et-sbtn` · `ord-quickbtn`/`ord-itembtn` · `ov-trk-btn`/`trk-qbtn`/`acc-trackbtn` · `imgfield-btn` · `cp-more-btn` · `sub-statusbtn`.

**C2 — Aynı rol, farklı ölçü:**
- *Toolbar:* `.btn.sm` (6×10, r10) vs `inv-seg`/`inv-filter-btn` (8×13, r9, surf2) vs `ct-tbtn` (8×12, r9, **surf**). Orders/Inventory/Contacts toolbar'ları yan yana farklı.
- *Primary CTA:* `.btn`(r10/13/600) · `et-btn`(r8/12/700) · `apf-btn`(r6/10/700/**Chakra Petch**) · `bk-hero-cta`(r10/14/600). Radius 6→10, font 10→14, weight 600/700.

**C3 — Hover 5 dialekt:** `.btn`→zemin acc · `ct-tbtn`/`bulkbtn`→border acc+renk tx · `ord-quickbtn`/`imgfield-btn`→border acc+renk acc2 · `seg-btn`→border tx3 · `acc-trackbtn`(.92)/`sub-statusbtn`(.75)→opacity. Tek hover dili seç; opacity-hover'ları bırak.

**C4 — Migration yarım** (DESIGN-STANDARD P1 "tamam" diyor ama JSX'te canlı): `linklike`20× · `auth-btn`9× · `eb-seg`8× · `ord-quickbtn`8× · `pagebtn`7× · `bulkbtn`5× · `ct-tbtn`5× · `ct-bulk-btn`4× · `trk-qbtn`4× · `inv-seg`3× · `ck-gate-btn`3× · `inv-filter-btn`2× · `et-btn`2× · `apf-btn`2×.

---

## D. MODALLAR

**D1 — Kanonik `.modal` azınlıkta:** sadece **5 dosya** `.modal` kullanıyor; `ord-modal` **9 dosyada**, ayrıca `inv-modal`, `ct-edit-modal`/`md-bg`, storefront `bk-modal`(2), `md-box`, `apm-modal`. 6+ ayrı modal sistemi.

**D2 — Genişlik inline, ölçek yok:** OrderDetail modalları `style={{ width: 420/440/460/480 }}`, refund 560 (`ord-refmodal`). 5 farklı genişlik, class değil inline.

**D3 — Bazı modallarda `max-height`/scroll yok** → kısa ekranda alttaki butonlara ulaşılamaz. OrderDetail'de payLink(998)/editInfo(1036) `maxHeight:88-90vh+overflow` var AMA payModal(636)/share(690)/markPaid(1068)/overpay(1233)/cancel(1251) **yok**. Baz `.ord-modal`'da da `max-height` yok.

**D4 — Dikey hizalama tutarsız:** `.modal`→`flex-start` (üstten), `.ord-modal`/`.inv-modal`/`.ct-edit`→`center`. Farklı modallar ekranda farklı konumda açılıyor.

**D5 — Backdrop/gölge değişken:** `.modal`/`.ord-modal` rgba(8,5,20,.6)+blur3, `.bk-modal-ov` rgba(11,8,22,.66) blur yok; gölge `0 20px 60px` vs `0 24px 70px` vs yok.

---

## E. AÇILIR MENÜLER / DROPDOWN ("sağa açılır ıvır zıvır")

**E1 — 4 farklı açılış yönü:** right-anchor (`right:0`: rowmenu/bulkmenu/ord-menu/ct-create-menu/cp-more-menu) · left-anchor (`left:0`: ord-pop/cp-lblpick/inv-fby/ct-addfield) · full-width (`left:0;right:0`: tsel-menu/trk-cmenu) · **sağa flyout** (`left:100%`: s2-flyout/s2-flyout2, storefront alt-menü) · **yukarı** (`bottom:100%`: ct-addfield-menu). Hangi menünün nereye açıldığı öngörülemez.

**E2 — Magic offset'ler:** `rowmenu` `top:34px`, `catmenu` `top:40px;right:8px` — trigger yüksekliğine sabitlenmiş sihirli sayılar; buton boyu değişirse menü kayar. Diğerleri `top:calc(100% + 4/5/6px)` (o da 4/5/6 karışık).

**E3 — Radius karışık:** menülerin çoğu r10 ama `ord-pop`/`s2-flyout` r12.

**E4 — Portal yok:** `AdminSelect` + `ThemedSelect` menüyü inline render ediyor (`tsel-menu`), drop-up var ama `overflow:hidden` kapsayıcıda kesilir + B1'deki stacking'e açık. `createPortal` + tek overlay-root + `--z-popover` öner.

---

## F. DRAWER (sağ slide-in paneller)

**F1 — Genişlik:** `bk-drawer` min(420) (store) · `ct-drawer`/`inv-drawer`/`cpn-fdrawer` 340 · `cp-drawer` **440**. Standart canonical 340; 420/440 migrate.

**F2 — Border + animasyon 3 farklı:** border-left `.5px`(bk) vs `1px`(ct/inv); slide mekanizması üç ayrı: `bk-drawer` transform-transition (.28s ease), `ct-drawer` `@keyframes ctslide`(.2s), `inv-drawer` `@keyframes inv-slide`(.2s) — ikisi birebir aynı işi yapan **çift keyframe**.

---

## G. "AÇILIR-KAPANIR" DAVRANIŞ (en görünür UX)

**✅ Var:** backdrop-tıkla-kapan (20 component), dropdown dışına-tıkla-kapan, Drawer Escape.

**G1 — Arka plan scroll kilidi YOK.** `AdminScrollGuard` sadece `overscroll-behavior-x:none` ayarlıyor; `body{overflow:hidden}` yok → **modal/drawer açıkken arka sayfa kayıyor.**

**G2 — Modalların çoğu Escape ile kapanmıyor** (tüm projede 2 dosya Escape dinliyor; kanonik `.modal` formlarının çoğu yanıt vermiyor).

**G3 — Focus-trap yok (0 component).** 22 `autoFocus` var ama modal açıkken Tab arka plana kaçıyor, kapanınca focus geri dönmüyor (WCAG 2.4.3/2.1.2).

**G4 — Back-button davranışı tutarsız.** Sadece `ContactsView` `history.pushState` ile geri/swipe'ı kilitliyor; diğer modallarda geri tuşu modalı atlayıp sayfadan çıkarıyor. App genelinde tutarsız.

> **Tek çözüm:** ortak `<Modal>` component'i — backdrop+Escape+scroll-lock+focus-trap+`max-height:88vh`+iç scroll+tek width prop+tek backdrop/gölge/hizalama. D1–D5 ve G1–G4 birden kapanır.

---

## H. ANİMASYON

**H1 — 10 farklı süre:** .12(8)/.14(11)/.15(25)/.2(7)/.24/.26/.28/.3/.4/.55. Aynı micro-etkileşim için .12/.14/.15 üçlüsü. 2 token yeter (`--t-fast:.14`, `--t-mid:.24`). (Easing nispeten temiz: çoğu `ease` + 2 cubic-bezier.)

**H2 — Tekrarlanan keyframe'ler:** spinner **3×** (`bk-spin`/`ct-spin`/`inv-rot`), drawer slide **2×** (`ctslide`/`inv-slide`), pop-in **2×** (`inv-pop`/`mdpop`). Aynı animasyonun farklı isimlerle kopyaları — tek isimde topla.

---

## I. RESPONSIVE

**I1 — 10 breakpoint + sözdizimi karışık:** 560/640/720/760/820/860/880/900/1100/1200; `@media(max-width…)` (boşluksuz) vs `@media (max-width…)` (boşluklu) ikisi de var. Komşu bölümler farklı genişlikte reflow ediyor. 3'lü set belirle (600/900/1200).

---

## J. KOD / YAPI (görseli dolaylı bozan)

**J1 — Devasa component'ler:** `OrderDetail` 1276 satır / **50 useState** / 8 inline modal; `ContactsView` 1200 / **57 useState** / 21 overlay bloğu; `NewOrderForm` 573/39. Modallar hep bu dosyaların içinde inline tanımlı → D bölümündeki tutarsızlığın sebebi. Parçala (Modal'ları ayır).

**J2 — `money`/format helper'ı 8× tekrar:** `lib/format.js` `money` export ediyor AMA OrdersView/OrderDetail/NewOrderForm/OrderFormBits/AdminChargeModal/PayPage/InvoicesView kendi lokal `const money` yazmış (OrderDetail'de hem `money` hem `_m` — aynı dosyada iki kopya). Para biçimi değişirse 8+ yer. Hepsi `lib/format`'tan import etsin.

**J3 — 1401 inline style / 59 dosya** (OrderDetail 179, NewOrderForm 87, InvoiceForm 78, InvoicesView 74…). CSS class sistemini baypas eden tek-seferlik renk/spacing/size'ların ana yuvası. Tekrar edenleri (A8 grey'leri, sabit width'ler) class/token'a çek.

**J4 — Başlık tutarsız:** `className="heading"` 43× ama ham `<h1/h2/h3>` 77×. Standart "sayfa başlığı = `heading`" diyor; 77 ham başlık ad-hoc/parent-selektörle stilleniyor → tipografi tutarsız.

**Diğer:** liste tabloları 13 dosyada `<table>` — ortak `table` stilini paylaşıp paylaşmadıklarını doğrula. `key={index}` 28×. TODO/HACK notu 5 (düşük).

---

## K. ERİŞİLEBİLİRLİK

**K1 — ~170 tıklanabilir `div/span/td/tr`** ama `role=` **0**, sadece 19 `aria-label`. Klavyeyle gezilemez/çalıştırılamaz, ekran okuyucu "buton" demez. Özellikle storefront (müşteri) önemli. `<button>`'a çevir ya da `role="button" tabIndex={0} onKeyDown(Enter/Space)`.

**K2 — Kontrast (WCAG):** token'ların çoğu temiz (beyaz 16–18, tx2 ~8, acc2 ~9, acc ~5.5 — AA geçer). Tek istisna: **`tx3` (#8b7ab8) `surf2` (#241a45) üstünde 4.26** → küçük metin için AA(4.5) altında (büyük metin/UI için sorun yok). En koyu zeminde yardımcı metni biraz aç ya da büyük metinle sınırla.

---

## L. STOREFRONT-ÖZEL

- **Kendi modal/toast/drawer'ı:** `bk-modal` (r14 + **kırmızı başlık** + Chakra Petch), `bk-toast` (**z60** düşük — modal arkasında kalır), `bk-drawer` (z50, transform-transition). Admin'inkilerden ayrı. NavGuard'ın admin confirm'i de hâlâ storefront `bk-modal` stilinde (DESIGN-STANDARD §9 işaretli).
- **`s2-flyout` sağa açılan alt-menü** (`left:100%`) — admin'de eşi yok; ayrı bir açılış paradigması.
- Storefront `<img>` (next/image değil), tıklanabilir div'ler (K1) — müşteri tarafında daha kritik.

---

## M. KASITLI / BUG DEĞİL (vakit kaybetme)

- **İkili token sistemi** `--bg/--surf` (admin) vs `--m-*` (storefront/tema): merchant `AppearanceEditor` ile temalanıyor — tasarımsal, doğru.
- **`apf-`/`apm-`/AppearanceEditor**'ın `--m-*` + Chakra Petch kullanması: admin chrome'u değil, **storefront tema önizlemesi** — "tutarsız buton" sanıp düzeltme.
- **`et-` (email template) + invoice/PDF (`InvoiceDocument`) hardcoded hex'ler:** e-posta istemcileri/PDF CSS değişkeni desteklemez — orada hardcoded **doğru**, token'a zorlama.
- **Açık renkler** (`#faf9ff`, `#f4f4f7`, `#eee`, `#666` gibi): büyük ihtimalle invoice/print/light bağlamı — A8'e dahil etmeden önce bağlamı kontrol et.

---

## ÖNERİLEN SIRA (en çok kazandıran → en az)

1. **Tek `<Modal>` component'i** → D1–D5 + G1–G4 (en görünür UX + en çok tutarsızlık).
2. **Token dosyası** (`tokens.css`): z-index (B1) + radius (A1) + font-size (A2) + spacing (A3) + shadow (A4) + opacity (A5) + animasyon (H1). Toplu bul-değiştir.
3. **Dropdown'ları portal'la** (E4) + menü açılış yönü/offset'i standartlaştır (E1–E3).
4. **Tek Drawer component'i** (F1–F2) + tekrarlanan keyframe'leri birleştir (H2).
5. **Buton ailelerini bitir** (C1–C4) — DESIGN-STANDARD planına göre, hover dilini birle.
6. **Inline style avı** (J3) + `lib/format` import (J2) + `heading` class (J4) + ikon/X boyutu (A7).
7. **Erişilebilirlik** (K1) — storefront öncelikli.
8. **Renk temizliği** (A8) — M'deki bağlamları ayırarak.
