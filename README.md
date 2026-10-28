# GNC Randevu

**İşletmeler için müşteri randevu platformu** — GNC ERP ekosisteminden ayrıştırılmış, bağımsız çalışan müşteri randevu sitesi. Port **3002**.

Müşteriler işletmeleri keşfeder, hizmet seçer, uygun saate rezervasyon yapar; üyelik ile randevularını yönetir, değerlendirme yazar, bekleme listesine girer, damga kartlarını takip eder. İşletmeler kendi web sitelerine `iframe` widget ile aynı randevu akışını gömer.

## Mimari

```
Tarayıcı ──▶ randevu (3002, bu repo)  ──server-side fetch──▶  GNC ERP API (3001)
   same-origin /api/*                       /api/public/*
   HTML sayfalar + cache                     PostgreSQL (gnc_erp)
```

- **Veri katmanı GNC ERP PostgreSQL'idir (API üzerinden).** Bu servisin kendi veritabanı yoktur — bilinçli mimari karardır: randevu, üyelik, değerlendirme, bekleme listesi ve damga kartı verilerinin tek kaynağı GNC ERP'tir.
- Tarayıcıya yalnızca bu servisin **same-origin** `/api/*` uçları açılır (CORS yok, CRM adresi tarayıcıya sızdırılmaz).
- Sunucu tarafında kısa süreli **bellek cache'i**: işletme listesi 60 sn, slotlar 10 sn, işletme detayı 60 sn, değerlendirmeler 15 sn.
- **Zero-dep**: tek çalışma zamanı **Bun**; npm bağımlılığı yoktur. Build adımı yoktur — HTML/CSS/JS şablonları `views.ts` içinde gömülüdür, statik varlıklar (`assets/`) startup'ta belleğe okunur.
- **Sandbox gateway desteği**: sayfa `?XTransformPort=` ile açıldığında tüm göreli iç linklere bu parametre eklenir (click-time yakalayıcı + MutationObserver ile dinamik linkler dahil). Parametre yoksa script hiçbir şey yapmaz — üretimde zararsızdır.

## Özellikler

- 🏪 **İşletme keşfi** — arama (isim/adres/hizmet, Türkçe-duyarlı küçük harf), tür ve şehir filtresi, puan rozetleri
- 📅 **Randevu akışı** — hizmet → tarih → uygun saat; "ilk uygun saat" kısayolu; misafir (kod ile) ve üye randevusu
- 📆 **ICS takvim dosyası** — "Takvime ekle" (`?code=` misafir, `?id=` + Bearer üye)
- 👤 **Üyelik** — kayıt/giriş, randevularım (iptal/yeniden planla), hesap silme (KVKK)
- ⭐ **Değerlendirmeler** — puan + yorum (yönetici onaylı), dağılım özeti
- ⏳ **Bekleme listesi** — dolu saate bekleme kaydı + "Bekleme Durumum" sorgusu
- 🎟️ **Damga kartı** — sadakat programı kartları (aktif program olan işletmeler)
- 🗺️ **Harita** — Leaflet (self-hosted, CDN yok) ile işletme konumu
- 🌐 **Alt alan adı vitrinleri** — `sik-kuafor.gncinc.online` şeklinde işletme başına otomatik sayfa
- 🔗 **Siteye göm widget** — `/{widget}/:slug` iframe kaynağı (`?embed=1` kompakt mod)
- 🍉 Self-hosted varlıklar, cache-busting sürüm parametresi, 404 önerileri (hizmet adı eşleşmeli)

## API uçları

| Metot | Uç | Açıklama |
|-------|----|----------|
| GET | `/healthz` | Sağlık kontrolü (`{ok, service, port, crmBase, subdomainMode}`) |
| GET | `/` | Ana sayfa — işletme listesi |
| GET | `/uyelik` | Üyelik sayfası (giriş/kayıt/randevularım) |
| GET | `/isletme/:slug` | İşletme detay sayfası (`?embed=1` gömülü mod, `?q=` 404 öneri terimi) |
| GET | `/widget/:slug` | iframe widget kaynağı (detay sayfasının çerçevesiz hâli) |
| GET | `/api/businesses` | İşletme listesi — `?q=&type=&city=` filtreleri; şehir listesi döner |
| GET | `/api/businesses/:slug` | İşletme detayı + hizmetleri |
| GET | `/api/businesses/:slug/slots?date=&serviceId=` | Uygun saatler (+ `nextAvailable` kısayolu) |
| POST | `/api/book` | Randevu oluştur (misafir/üye; doğrulama sunucu tarafında) |
| GET/POST/DELETE | `/api/member/*` | Üyelik proxy'si — CRM `/api/public/member/*` uçlarını same-origin yapar (Bearer iletilir) |
| POST | `/api/reviews` | Değerlendirme gönder (üye token'ı varsa iletilir) |
| GET | `/api/reviews/providers/:slug?limit=` | İşletme değerlendirmeleri + özet |
| GET/POST | `/api/waitlist` | Bekleme listesi (gerçek IP `x-forwarded-for` ile taşınır — rate limit) |
| GET | `/api/stampcard` | Damga kartları (Bearer) |
| GET | `/api/appointments/ics?code=` veya `?id=` | ICS takvim dosyası (misafir kodu ya da üye token) |
| GET | `/assets/*` | Gömülü varlıklar (style.css, JS, Leaflet, görseller) |

> Sunucu, bu uçların arkasında GNC ERP'ten şu kamu API'lerini çağırır:
> `/api/public/providers`, `/api/public/providers/:slug(/availability|/reviews)`, `/api/public/appointments(/ics)`, `/api/public/member/*`, `/api/public/reviews`, `/api/public/waitlist`, `/api/public/stampcard`.

## Geliştirme

```bash
# Geliştirme (hot reload; ERP API 3001'de ayakta olmalı)
bun run dev

# Üretim
bun run start          # → http://localhost:3002
```

Gereksinim: [Bun](https://bun.sh) 1.x — bağımlılık kurulumu yoktur (`bun install` gerekmez).

## Ortam değişkenleri

| Değişken | Varsayılan | Açıklama |
|----------|-----------|----------|
| `CRM_BASE_URL` | `http://localhost:3001` | GNC ERP (CRM) API taban adresi — tüm veri fetch'leri bu adrese gider |
| `BASE_DOMAIN` | *(boş → kapalı)* | Alt alan adı modu: `{slug}.BASE_DOMAIN` Host başlığı işletme vitrinine, `randevu.BASE_DOMAIN` ana listeye gider |

Örnek dosya: `.env.example` → `cp .env.example .env`

## Deployment

Port **3002** (hardcode — gateway kuralı). Üretim örneği:

```bash
CRM_BASE_URL=http://localhost:3001 BASE_DOMAIN=gncinc.online bun run start
```

### Nginx (randevu.gncinc.online + {slug}.gncinc.online wildcard)

```nginx
# randevu.gncinc.online — ana işletme listesi
server {
    listen 443 ssl http2;
    server_name randevu.gncinc.online;
    location / {
        proxy_pass http://127.0.0.1:3002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# {slug}.gncinc.online — her işletme için otomatik vitrin (wildcard)
# NOT: catch-all, diğer alt alan adlarını (www, crm, …) da yakalar; sunucu
# rezerve slug'ları (www, crm, api, …) güvenle reddeder, ancak mevcut alt
# alan adı bloklarından ÖNCE gelmemesine dikkat edin.
server {
    listen 443 ssl http2;
    server_name *.gncinc.online;
    location / {
        proxy_pass http://127.0.0.1:3002;
        proxy_set_header Host $host;               # alt alan adı tespiti için ZORUNLU
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Sertifika: `*.gncinc.online` wildcard TLS. Servis `x-forwarded-for` değerini CRM rate-limit'ine taşır; nginx'te iletmeyi unutmayın.

## Kapsam dışı

- GitHub push bu repo için ayrı plan aşamasında yapılır (W3).
- Veri şeması/migrasyonlar GNC ERP reposundadır; bu repo yalnızca sunum katmanıdır.

---

*GNC Inc — mini-services ekosistemi.* 🍉
