# OpenRPA Asistan Ekosistemi — Tasarım Dokümanı

**Tarih:** 2026-05-13
**Durum:** Onaylandı (v2)

---

## Amaç

Claude'u, şirkete ait OpenRPA fork'u ile çalışırken gerçek zamanlı bir rehber haline getirmek.
Kullanıcı çalışırken:
- Ekranda ne gördüğünü anlatır → Claude yönlendirir
- Hata mesajı / log yapıştırır → Claude teşhis koyar
- Yazdığı workflow'u gösterir → Claude inceler ve öneri verir
- Fork farkı keşfeder → canlı belge güncellenir

Platform: Şirkete ait OpenRPA fork'u (yapısal olarak vanilla OpenRPA'ya yakın).
Sektör: Sigorta (Polisoft, Open sigorta yazılımı, sigorta web portalleri).

---

## Ekosistem Mimarisi

```
Kullanıcı prompt gönderir
         ↓
openrpa-master (ANA SKILL — router)
         ↓ prompt analiz eder
         ↓
┌─────────────────────────────────────┐
│  Hangi kaynak?                      │
│                                     │
│  skill    → openrpa-*/SKILL.md      │
│  aktivite → notes/*.md              │
│  hata     → openrpa-debug           │
│  review   → openrpa-review          │
│  C#       → openrpa-csharp          │
│  Polisoft → openrpa-polisoft-deep   │
│  fork farkı → fork-differences.md  │
│  yeni şablon → templates/           │
└─────────────────────────────────────┘
```

---

## Klasör Yapısı

```
rpa-skills/
│
├── openrpa-master/              ← ANA SKILL / Router
│   └── SKILL.md
│
├── openrpa-workflow/            ✅ mevcut
├── openrpa-selector/            ✅ mevcut
├── openrpa-invoke/              ✅ mevcut
├── openrpa-openflow/            ✅ mevcut
│
├── openrpa-debug/               🆕 Hata teşhis asistanı
│   └── SKILL.md
│
├── openrpa-review/              🆕 Workflow inceleme
│   └── SKILL.md
│
├── openrpa-csharp/              🆕 InvokeCode C# rehberi
│   └── SKILL.md
│
├── openrpa-polisoft-deep/       🆕 Polisoft modülleri
│   └── SKILL.md
│
├── notes/                       🆕 Aktivite başına kişisel rehber
│   ├── GetElement.md
│   ├── TypeText.md
│   ├── ClickElement.md
│   ├── InvokeCode.md
│   ├── TryCatch.md
│   ├── OpenApplication.md
│   ├── InvokeWorkflow.md
│   ├── Delay.md
│   └── ForEach-BreakableLoop.md
│
├── templates/                   🆕 Placeholder — case'lere göre büyür
│   └── README.md
│
├── fork-differences.md          🆕 Canlı fark belgesi
│
└── docs/superpowers/specs/
    └── 2026-05-13-openrpa-ecosystem-design.md
```

---

## Bileşen Detayları

### openrpa-master (Router Skill)

**Amacı:** Her oturumun ilk durağı. Prompt'u analiz edip doğru kaynağa yönlendirir.

**İçeriği:**
- Ekosistemin tam haritası (hangi skill ne işe yarar)
- Prompt → kaynak eşleme tablosu (routing rules)
- Yeni içerik ekleme rehberi (nereye ne eklenir)
- Güncelleme protokolü (fork farkı bulunca ne yapılır)
- Okunma önceliği (önce master, sonra ilgili skill/note)

**Routing örnekleri:**
```
"element not found"          → openrpa-debug + notes/GetElement.md
"InvokeCode'da tarih format" → openrpa-csharp
"Polisoft teklif ekranı"     → openrpa-polisoft-deep + openrpa-workflow
"workflow doğru mu?"         → openrpa-review
"queue nasıl kurulur"        → openrpa-openflow
"selector kırıldı"           → openrpa-selector + notes/GetElement.md
"fork'ta farklı çalıştı"     → fork-differences.md güncelle
"yeni aktivite keşfettim"    → notes/ altına yeni .md ekle
```

---

### openrpa-debug

**Amacı:** Hata mesajı, log veya ekran tanımından hızlı teşhis.

**Kapsam:**
- Element Not Found, Timeout, NullReferenceException
- SAP Script hataları
- Web portal hataları (CAPTCHA, session timeout, dynamic content)
- Selector kırılmaları
- InvokeCode runtime hataları

**Yaklaşım:** Semptom → olası sebepler listesi → adım adım teşhis → çözüm

---

### openrpa-review

**Amacı:** Kullanıcının workflow adımlarını veya pseudo-xaml'ını inceleme.

**Kontrol listesi:**
- TryCatch eksik mi?
- Selector kalitesi (wildcard, automationid?)
- Idempotency var mı?
- Credential hardcode var mı?
- Modülerlik (tek workflow çok mu büyük?)
- Performans (gereksiz Delay, büyük DataTable?)
- Best practice uyumu

---

### openrpa-csharp

**Amacı:** InvokeCode aktivitesi için hazır C# kod blokları.

**Kapsam:**
- String/tarih/tip dönüşümleri, regex
- Excel COM Interop (hücre okuma/yazma, formül, sheet)
- DataTable filtreleme ve LINQ sorguları
- Dosya/klasör işlemleri, log yazma
- HTTP/REST çağrıları, JSON parse/serialize
- Sigorta sektörü örnekleri (TC kimlik doğrulama, plaka formatı, prim hesabı)

---

### openrpa-polisoft-deep

**Amacı:** Polisoft'un tüm modüllerine özel derinlemesine rehber.

**Kapsam:**
- Giriş ve oturum yönetimi (keep-alive, timeout)
- Teklif modülü — adım adım form doldurma
- Poliçe modülü — sorgulama, düzenleme
- Hasar modülü — bildirim girişi
- Zeyilname — değişiklik işlemleri
- Raporlama — liste çekme, export
- Grid/DBGrid okuma
- Modal/popup yönetimi

---

### notes/ Klasörü

Her OpenRPA aktivitesi için ayrı `.md` dosyası.

**Her dosyanın yapısı:**
```
# AktiviteAdı

## Ne Yapar
## Parametreler (önemli olanlar + ne zaman hangisi)
## Adım Adım Kullanım
## Sık Yapılan Hatalar
## Sigorta Sektörü Örnekleri
## Fork Notları (varsa)
```

**İlk oluşturulacak dosyalar:**
GetElement, TypeText, ClickElement, InvokeCode, TryCatch,
OpenApplication, InvokeWorkflow, Delay, ForEach-BreakableLoop

---

### templates/README.md

Klasörün amacını ve şablon ekleme formatını açıklar.
Gerçek şablonlar iş case'leri netleştikçe eklenir.

---

### fork-differences.md

Vanilla OpenRPA ile şirket fork'u arasında keşfedilen farkları biriktirir.

**Format:**
```
## [Tarih] Aktivite/Alan Adı
- Vanilla: ...
- Fork: ...
- Etki: ...
- İlgili skill güncellendi mi: Evet/Hayır
```

---

## Kararlar ve Gerekçeler

| Karar | Gerekçe |
|---|---|
| openrpa-master router skill | Tek giriş noktası — context karmaşası önlenir, hız artar |
| C# ayrı skill (notes'ta değil) | Skill'ler Claude tarafından otomatik tetiklenir; notes manuel referans |
| Templates placeholder başlar | Case'ler netleşmeden şablon yazmak tahmin oyunu |
| fork-differences.md canlı belge | Fork farkları önceden bilinmiyor, keşfedilecek |
| notes/ aktivite başına ayrı dosya | Tek büyük dosya yerine modüler — ihtiyaç duyulana git |

---

## Kapsam Dışı

- UiPath / Blue Prism / Power Automate
- Gerçek `.xaml` dosyası üretimi (şimdilik)
- Otomatik test framework'ü (sonraki aşama)
