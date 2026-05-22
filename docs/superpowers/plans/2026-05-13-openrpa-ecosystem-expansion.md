# OpenRPA Ekosistemi Genişletme — Implementation Planı

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** OpenRPA fork kullanıcısı için Claude'u gerçek zamanlı RPA asistanına dönüştüren eksiksiz bir skill + rehber ekosistemi oluşturmak.

**Architecture:** Tek giriş noktası olarak `openrpa-master` router skill; altında özelleşmiş skill'ler, aktivite bazlı notes, canlı fork belgesi ve şablon kütüphanesi placeholder'ı.

**Tech Stack:** Markdown (SKILL.md, .md), yazılı içerik — kod yok, test = bölüm kontrol listesi.

---

## Dosya Haritası

```
rpa-skills/
├── openrpa-master/SKILL.md          ← Task 2
├── openrpa-debug/SKILL.md           ← Task 3
├── openrpa-review/SKILL.md          ← Task 4
├── openrpa-csharp/SKILL.md          ← Task 5
├── openrpa-polisoft-deep/SKILL.md   ← Task 6
├── notes/
│   ├── GetElement.md                ← Task 7
│   ├── TypeText.md                  ← Task 8
│   ├── ClickElement.md              ← Task 9
│   ├── InvokeCode.md                ← Task 10
│   ├── TryCatch.md                  ← Task 11
│   ├── OpenApplication.md           ← Task 12
│   ├── InvokeWorkflow.md            ← Task 13
│   ├── Delay.md                     ← Task 14
│   └── ForEach-BreakableLoop.md     ← Task 15
├── templates/README.md              ← Task 16
└── fork-differences.md              ← Task 17
```

---

## Task 1: Dizin Yapısını Oluştur

**Files:**
- Create: `openrpa-master/` (dizin)
- Create: `openrpa-debug/` (dizin)
- Create: `openrpa-review/` (dizin)
- Create: `openrpa-csharp/` (dizin)
- Create: `openrpa-polisoft-deep/` (dizin)
- Create: `notes/` (dizin)
- Create: `templates/` (dizin)

- [ ] **Adım 1: Dizinleri oluştur**

```powershell
$base = "C:\Users\cagda\OneDrive\Masaüstü\Project\rpa-skills"
New-Item -ItemType Directory -Force `
  "$base\openrpa-master", `
  "$base\openrpa-debug", `
  "$base\openrpa-review", `
  "$base\openrpa-csharp", `
  "$base\openrpa-polisoft-deep", `
  "$base\notes", `
  "$base\templates"
```

- [ ] **Adım 2: Doğrula**

Tüm dizinler oluştu mu kontrol et. 7 dizin görünmeli.

- [ ] **Adım 3: Commit**

```bash
git add -A
git commit -m "chore: create openrpa ecosystem directory structure"
```

---

## Task 2: openrpa-master/SKILL.md (Router Skill)

**Files:**
- Create: `openrpa-master/SKILL.md`

- [ ] **Adım 1: Dosyayı yaz**

Zorunlu bölümler:

```
---
name: openrpa-master
description: >
  Use FIRST for any OpenRPA question, error, workflow review, or RPA task.
  Routes to the correct skill or note based on the prompt. Use when the user
  asks anything about OpenRPA, robot building, automation, Polisoft, SAP,
  selectors, InvokeCode, C#, errors, workflow design, or fork differences.
---

# OpenRPA Master — Ekosistem Rehberi ve Router

## Bu Skill Ne Yapar
[Ekosistemin giriş noktası olduğunu açıkla]

## Ekosistem Haritası
[Tüm skill'lerin ve notes dosyalarının tablosu — ne işe yarar]

## Routing Kuralları
[Semptom/konu → hangi skill/note tablosu]
Örnekler:
  "element not found / timeout"     → openrpa-debug + notes/GetElement.md
  "InvokeCode C# kodu"              → openrpa-csharp
  "Polisoft ekranı / modülü"        → openrpa-polisoft-deep + openrpa-workflow
  "workflow doğru mu / review"      → openrpa-review
  "selector / element bulamıyor"    → openrpa-selector + notes/GetElement.md
  "queue / workitem / orkestrasyon" → openrpa-openflow
  "invoke / tetikle / PowerShell"   → openrpa-invoke
  "hangi aktivite / nasıl yapılır"  → openrpa-workflow + ilgili notes/
  "fork farkı keşfettim"            → fork-differences.md güncelle
  "yeni aktivite / nasıl çalışır"   → notes/ altına yeni .md ekle
  "şablon / template"               → templates/README.md

## Dosyaya Ekleme Kuralları
[Ne zaman yeni skill, ne zaman notes, ne zaman template eklenir]

## Güncelleme Protokolü
[Fork farkı bulununca ne yapılır, hangi dosya güncellenir]

## Okunma Önceliği
[Her soruda: önce master → sonra ilgili skill → sonra ilgili note]
```

- [ ] **Adım 2: Bölüm kontrolü**

✅ frontmatter: name + description (Use when... ile başlıyor)
✅ Ekosistem haritası tablosu var
✅ Routing tablosu var (semptom → hedef)
✅ Ekleme/güncelleme kuralları var
✅ Okunma önceliği açık

- [ ] **Adım 3: Commit**

```bash
git add openrpa-master/SKILL.md
git commit -m "feat: add openrpa-master router skill"
```

---

## Task 3: openrpa-debug/SKILL.md

**Files:**
- Create: `openrpa-debug/SKILL.md`

- [ ] **Adım 1: Dosyayı yaz**

Zorunlu bölümler:

```
---
name: openrpa-debug
description: >
  Use when an OpenRPA robot throws an error, behaves unexpectedly, or when
  diagnosing element not found, timeout, NullReference, SAP errors, web portal
  failures, or selector breaks. Triggers on: "hata aldım", "element not found",
  "timeout", "çalışmıyor", "NullReference", "selector kırıldı", "robot durdu".
---

# OpenRPA Debug — Hata Teşhis Rehberi

## Hata Tipi Tanıma
[Hata mesajı → kategori tablosu]

## Element Not Found
Semptom → Teşhis adımları (6 adım) → Çözümler

## Timeout Hataları
Semptom → Sebepler → Çözümler

## NullReferenceException
Neden oluşur → Nerede aranır → Çözüm

## SAP Hataları
SAP Script kapalı / ekran değişimi / oturum kapalı → Her biri için çözüm

## Web Portal Hataları
CAPTCHA / Session timeout / Dynamic content / CORS → Çözümler

## InvokeCode Runtime Hataları
Cast hatası / null değer / COM exception → Çözümler

## Teşhis Akışı (Genel)
[Adım adım teşhis şeması — hangi hata türünde ne bakılır]

## Log Okuma Rehberi
[OpenRPA log formatı, nereden okunur, neye bakılır]
```

- [ ] **Adım 2: Bölüm kontrolü**

✅ Her hata kategorisi ayrı başlık
✅ Her kategori: semptom + sebep + çözüm
✅ Sigorta uygulamalarına özel notlar var (Polisoft, web portal)
✅ description "Use when..." ile başlıyor

- [ ] **Adım 3: Commit**

```bash
git add openrpa-debug/SKILL.md
git commit -m "feat: add openrpa-debug diagnostic skill"
```

---

## Task 4: openrpa-review/SKILL.md

**Files:**
- Create: `openrpa-review/SKILL.md`

- [ ] **Adım 1: Dosyayı yaz**

Zorunlu bölümler:

```
---
name: openrpa-review
description: >
  Use when reviewing an OpenRPA workflow for correctness, best practices,
  security, or maintainability. Triggers on: "workflow doğru mu", "bak bakar
  mısın", "eksik bir şey var mı", "incele", "best practice", "review".
---

# OpenRPA Workflow Review

## Review Kontrol Listesi
Her workflow'da kontrol edilecekler (checkbox formatında):

### Hata Yönetimi
- [ ] Her ana blok TryCatch ile sarılı mı?
- [ ] Catch bloğunda log var mı?
- [ ] Finally'de temizlik (CloseApplication) var mı?
- [ ] Retry logic var mı (geçici hatalar için)?

### Güvenlik
- [ ] Credential hardcode var mı? (varsa → OpenFlow Credentials'a taşı)
- [ ] Log aktivitesine şifre yazılıyor mu?
- [ ] Kişisel veri (TC, IBAN) log'a düşüyor mu?

### Selector Kalitesi
- [ ] automationid kullanılıyor mu (varsa)?
- [ ] Dinamik title/name'lerde wildcard var mı?
- [ ] idx kullanımı gerçekten gerekli mi?

### Modülerlik
- [ ] Workflow 200 satırı geçiyor mu? (geçiyorsa böl)
- [ ] Aynı blok tekrarlanıyor mu? (InvokeWorkflow ile modüle al)

### Performans
- [ ] Gereksiz fixed Delay var mı? (GetElement ile değiştir)
- [ ] Büyük DataTable argüman olarak mı geçiyor?
- [ ] Loop içinde büyük nesne biriktiriliyorsa temizleniyor mu?

### Idempotency
- [ ] Aynı kayıt iki kez işlenirse ne olur?
- [ ] İşlem öncesi kayıt var mı kontrolü yapılıyor mu?

## Yaygın Sorunlar ve Düzeltmeleri
[Her sorun için before/after örneği]

## Öncelik Sırası
[Kritik (güvenlik/veri) > Önemli (hata yönetimi) > İyi olur (performans)]
```

- [ ] **Adım 2: Bölüm kontrolü**

✅ Kontrol listesi checkbox formatında
✅ Güvenlik bölümü var (KVKK kritik)
✅ Before/after örnekleri var
✅ Öncelik sırası net

- [ ] **Adım 3: Commit**

```bash
git add openrpa-review/SKILL.md
git commit -m "feat: add openrpa-review workflow review skill"
```

---

## Task 5: openrpa-csharp/SKILL.md

**Files:**
- Create: `openrpa-csharp/SKILL.md`

- [ ] **Adım 1: Dosyayı yaz**

Zorunlu bölümler + hazır kod blokları:

```
---
name: openrpa-csharp
description: >
  Use when writing C# code inside OpenRPA InvokeCode activity. Covers string,
  date, type conversions, regex, Excel COM Interop, DataTable/LINQ, file I/O,
  HTTP/REST calls, JSON parsing. Triggers on: "InvokeCode", "C# kodu", "nasıl
  yazılır", "Excel COM", "DataTable filtre", "JSON parse", "HTTP istek".
---

# OpenRPA — InvokeCode C# Rehberi

## InvokeCode Temelleri
- Değişken yönleri (In/Out/InOut)
- using namespace ekleme
- async/await (OpenRPA'da dikkat noktası)

## String ve Tarih İşlemleri
[Hazır kod blokları:]
- Trim/ToUpper/Replace/Split
- DateTime parse (Türkçe format: "dd.MM.yyyy")
- Tarih karşılaştırma ve fark hesaplama
- TC Kimlik format kontrolü
- Plaka format kontrolü
- Para formatı (1.234,56 → decimal)
- Regex ile veri çekme

## Tip Dönüşümleri
- string → int, decimal, DateTime (null-safe)
- object → string (null-safe)
- DataRow değerlerine güvenli erişim

## DataTable ve LINQ
- DataTable'ı filtrele (LINQ)
- DataTable'dan değer oku
- DataTable sırala
- İki DataTable birleştir
- DataTable → List<Dictionary>

## Excel COM Interop
- Aktif Excel'den veri oku
- Belirli sheet + hücreye yaz
- UsedRange ile son satır bul
- Hücreyi formülle doldur
- Excel'i kaydet ve kapat

## Dosya ve Klasör İşlemleri
- Dosya var mı kontrol
- Dosyayı kopyala/taşı/sil
- Klasördeki dosyaları listele
- Log dosyasına yaz (append)
- Geçici dosya oluştur

## HTTP ve REST API
- GET isteği + JSON parse
- POST isteği (JSON body)
- Bearer token ile kimlik doğrulama
- Timeout ve retry
- Sigorta API örneği (teklif sorgulama)

## JSON İşlemleri
- JSON string → JObject
- JObject'ten değer oku (null-safe)
- Nesneyi JSON'a dönüştür
- JSON array döngüsü

## Sigorta Sektörü Hazır Fonksiyonlar
- TC Kimlik doğrulama algoritması
- Prim tutarını formatla
- Poliçe bitiş tarihi hesapla (1 yıl sonrası)
- IBAN format kontrolü
```

- [ ] **Adım 2: Bölüm kontrolü**

✅ Her bölümde çalışır hazır kod blokları
✅ Sigorta sektörü örnekleri var
✅ Null-safe erişim gösterilmiş
✅ Türkçe tarih/para format örnekleri var

- [ ] **Adım 3: Commit**

```bash
git add openrpa-csharp/SKILL.md
git commit -m "feat: add openrpa-csharp InvokeCode reference skill"
```

---

## Task 6: openrpa-polisoft-deep/SKILL.md

**Files:**
- Create: `openrpa-polisoft-deep/SKILL.md`

- [ ] **Adım 1: Dosyayı yaz**

Zorunlu bölümler:

```
---
name: openrpa-polisoft-deep
description: >
  Use when automating Polisoft insurance software — login, quotation (teklif),
  policy (poliçe), claim (hasar), endorsement (zeyilname), or reporting modules.
  Triggers on: "Polisoft", "teklif", "poliçe", "hasar", "zeyilname", "sigorta
  programı", "DBGrid", "TfrmTeklif".
---

# Polisoft Otomasyon Rehberi

## Teknik Altyapı
- Delphi/VCL tabanlı Windows uygulaması
- Provider: Windows UIAutomation
- Selector sözdizimi: cls=TForm/TEdit/TButton/TDBGrid
- component name = Delphi Object Inspector adı

## Giriş ve Oturum Yönetimi
- Login akışı (adım adım + selector)
- Oturum zaman aşımı (varsayılan 30 dk)
- Keep-alive stratejisi (her 25 dk KeepAlive GetElement)
- Çoklu oturum açma engelini aşma

## Teklif Modülü
- Yeni teklif açma
- Müşteri bilgisi arama / seçme
- TC/VKN girişi ve otomatik doldurma bekleme
- Araç bilgisi girişi (plaka, marka, model, yıl)
- Branş seçimi (Trafik, Kasko, Konut, DASK, Sağlık)
- Prim hesapla → sonucu oku
- Teklif kaydet → teklif no'yu al
- Selector örnekleri: edtTCKimlik, edtPlaka, btnHesapla, lblTeklifNo

## Poliçe Modülü
- Poliçe sorgulama (no ile, plaka ile, müşteri ile)
- Poliçe detaylarını okuma
- Poliçe düzenleme

## Hasar Modülü
- Yeni hasar bildirimi açma
- Poliçe bağlama
- Hasar detayları girişi
- Hasar no alma ve kaydetme

## Zeyilname Modülü
- Zeyilname türleri
- Değişiklik girişi akışı

## Raporlama Modülü
- Liste / rapor çekme
- Tarih aralığı filtreleme
- Export (Excel/PDF)
- Grid'den veri okuma

## Grid (TDBGrid) Yönetimi
- Satırları GetElement ile toplu alma (MaxResults)
- Satır + sütun erişimi
- Kaydırma (scroll) gerektiren grid'ler
- Tıklama ile satır seçme

## Modal / Popup Yönetimi
- Onay dialogları (Evet/Hayır/Tamam)
- Uyarı popup'ları
- Pick/PickBranch ile bekleme stratejisi

## Yaygın Polisoft Hataları
- "Kayıt bulunamadı" → sebep + çözüm
- "Oturum zaman aşımı" → çözüm
- "Başka kullanıcı tarafından kilitleniyor" → çözüm
- Grid boş görünüyor → sebep + çözüm

## Fork Notları
[Şirket fork'unda Polisoft entegrasyonuna özel farklılıklar — keşfedildikçe eklenir]
```

- [ ] **Adım 2: Bölüm kontrolü**

✅ Her modül ayrı başlık
✅ Selector örnekleri somut (edtTCKimlik vb.)
✅ Yaygın hatalar ve çözümleri var
✅ Fork notları bölümü placeholder olarak var

- [ ] **Adım 3: Commit**

```bash
git add openrpa-polisoft-deep/SKILL.md
git commit -m "feat: add openrpa-polisoft-deep insurance software skill"
```

---

## Task 7–15: notes/ Aktivite Dosyaları

Her dosya aynı şablonu izler. Aşağıdaki 9 dosya sırasıyla yazılır.

**Her dosyanın şablonu:**
```markdown
# AktiviteAdı

## Ne Yapar
[1-2 cümle özet]

## Parametreler
| Parametre | Tip | Açıklama | Ne Zaman Kullanılır |
|---|---|---|---|

## Adım Adım Kullanım
[Toolbox'tan sürükle → yapılandır → test et]

## Sık Yapılan Hatalar
| Hata | Sebep | Çözüm |
|---|---|---|

## Sigorta Sektörü Örnekleri
[Polisoft / web portal / SAP örneği]

## Fork Notları
[Şirket fork'unda farklı davranışlar — keşfedildikçe eklenir]
```

### Task 7: notes/GetElement.md

- [ ] Yaz: Timeout/MinResults/MaxResults parametreleri, selector bağlantısı, varlık kontrolü (MinResults=0), MaxResults ile toplu alma, element null kontrolü
- [ ] Kontrol: Parametre tablosu ✅, Hata tablosu ✅, Sigorta örneği (Polisoft form alanı) ✅
- [ ] Commit: `git commit -m "docs: add GetElement activity note"`

### Task 8: notes/TypeText.md

- [ ] Yaz: Özel tuş sözdizimi ({Enter}/{Tab}/{F4}/{Ctrl+a}), VirtualClick farkı, gecikme parametresi, SAP alanına yazma, web input'a yazma, şifre alanı
- [ ] Kontrol: Tüm özel tuşlar listelenmiş ✅, SAP örneği ✅, şifre alanı güvenlik notu ✅
- [ ] Commit: `git commit -m "docs: add TypeText activity note"`

### Task 9: notes/ClickElement.md

- [ ] Yaz: VirtualClick=True vs False farkı, OffsetX/OffsetY kullanımı, GetElement içinde kullanım zorunluluğu, double-click, right-click
- [ ] Kontrol: Virtual vs Real click farkı net ✅, OffsetX/Y örneği ✅
- [ ] Commit: `git commit -m "docs: add ClickElement activity note"`

### Task 10: notes/InvokeCode.md

- [ ] Yaz: C# vs VB.NET seçimi, In/Out/InOut değişken yönleri, using namespace ekleme, async dikkat noktası, compile hataları nasıl okunur, openrpa-csharp skill'ine yönlendirme
- [ ] Kontrol: Değişken yönleri tablosu ✅, async uyarısı ✅, csharp skill cross-ref ✅
- [ ] Commit: `git commit -m "docs: add InvokeCode activity note"`

### Task 11: notes/TryCatch.md

- [ ] Yaz: Try/Catch/Finally yapısı, birden fazla Catch (farklı exception tipi), exception değişkenine erişim (e.Message, e.StackTrace), Finally garantisi, iç içe TryCatch, ne zaman rethrow yapılır
- [ ] Kontrol: Finally garantisi açıklanmış ✅, e.Message/e.StackTrace gösterilmiş ✅
- [ ] Commit: `git commit -m "docs: add TryCatch activity note"`

### Task 12: notes/OpenApplication.md

- [ ] Yaz: Uygulama açma vs odaklanma farkı, pencere konumlandırma, Polisoft/SAP/Excel örnek selector'ları, uygulama zaten açıksa ne olur, başlatma zaman aşımı
- [ ] Kontrol: Polisoft selector örneği ✅, SAP örneği ✅, zaman aşımı notu ✅
- [ ] Commit: `git commit -m "docs: add OpenApplication activity note"`

### Task 13: notes/InvokeWorkflow.md

- [ ] Yaz: In/Out/InOut fark tablosu, WorkflowFileName seçimi, bağımsız çalışabilir alt workflow tasarımı, büyük DataTable geçirme uyarısı, openrpa-invoke skill'ine yönlendirme
- [ ] Kontrol: Parametre yönleri tablosu ✅, DataTable uyarısı ✅
- [ ] Commit: `git commit -m "docs: add InvokeWorkflow activity note"`

### Task 14: notes/Delay.md

- [ ] Yaz: Ne zaman kullanılır vs kullanılmaz, ms cinsinden değerler, GetElement ile dinamik bekleme (önerilen alternatif), SAP'ta ne kadar beklenmeli, ne zaman Delay kaçınılmazdır
- [ ] Kontrol: "Delay yerine GetElement kullan" mesajı net ✅, SAP önerisi ✅
- [ ] Commit: `git commit -m "docs: add Delay activity note"`

### Task 15: notes/ForEach-BreakableLoop.md

- [ ] Yaz: ForEach vs BreakableLoop farkı, index/Total kullanımı, Break ile döngüden çıkış, Continue, döngü içinde TryCatch, büyük koleksiyonda bellek yönetimi, WorkItem queue ile karşılaştırma
- [ ] Kontrol: ForEach vs BreakableLoop farkı net ✅, Break örneği ✅, bellek uyarısı ✅
- [ ] Commit: `git commit -m "docs: add ForEach-BreakableLoop activity note"`

---

## Task 16: templates/README.md

**Files:**
- Create: `templates/README.md`

- [ ] **Adım 1: Dosyayı yaz**

```markdown
# Workflow Şablonları

Bu klasör, sık kullanılan sigorta otomasyon senaryoları için
hazır workflow şablonlarını içerir.

## Bu Klasör Ne İçin?

Gerçek iş case'leri netleştikçe buraya şablon eklenir.
Her şablon: adım adım akış + pseudo-xaml + dikkat noktaları + selector örnekleri.

## Şablon Ekleme Formatı

Yeni bir şablon eklerken:
1. Dosya adı: `[süreç-adı].md` (küçük harf, tire ile)
   Örn: `policeye-yenileme.md`, `hasar-bildirimi.md`
2. İçerik yapısı:
   - ## Amaç (ne otomatize ediyor)
   - ## Ön Koşullar (hangi uygulamalar açık olmalı, hangi veriler hazır)
   - ## Akış Diyagramı (adım adım)
   - ## Pseudo-Xaml (workflow yapısı)
   - ## Selector Örnekleri
   - ## Dikkat Noktaları
   - ## Fork Notları

## Mevcut Şablonlar

*(Henüz eklenmedi — iş case'leri netleştikçe buraya eklenecek)*
```

- [ ] **Adım 2: Doğrula**

✅ Ekleme formatı açık
✅ Dosya adlandırma kuralı var
✅ "Henüz yok" durumu dürüstçe belirtilmiş

- [ ] **Adım 3: Commit**

```bash
git add templates/README.md
git commit -m "docs: add templates placeholder with contribution guide"
```

---

## Task 17: fork-differences.md

**Files:**
- Create: `fork-differences.md`

- [ ] **Adım 1: Dosyayı yaz**

```markdown
# Fork Farklılıkları — Canlı Belge

Şirket OpenRPA fork'u ile vanilla OpenRPA arasında keşfedilen farklar.
Çalışırken bir fark bulununca buraya eklenir.

## Nasıl Kullanılır

1. Bir farklılık keşset (aktivite farklı davranıyor, yeni özellik var vb.)
2. Aşağıdaki formatta buraya ekle
3. İlgili skill veya note güncellemesi gerekiyorsa belirt
4. openrpa-master routing kuralları etkileniyorsa onu da güncelle

## Fark Ekleme Formatı

### [YYYY-MM-DD] Alan/Aktivite Adı
- **Vanilla OpenRPA:** ...
- **Şirket Fork'u:** ...
- **Etki:** Hangi workflow'ları/skill'leri etkiler
- **İlgili Skill Güncellendi mi:** Evet / Hayır / Bekliyor

---

## Keşfedilen Farklar

*(Henüz keşfedilen fark yok — ilk farkı bulunca buraya ekle)*
```

- [ ] **Adım 2: Doğrula**

✅ Ekleme formatı net
✅ "Henüz yok" başlangıç durumu var
✅ Güncelleme protokolü açık

- [ ] **Adım 3: Commit**

```bash
git add fork-differences.md
git commit -m "docs: add fork-differences live tracking document"
```

---

## Self-Review Notları

**Spec coverage:**
- ✅ openrpa-master (router) → Task 2
- ✅ openrpa-debug → Task 3
- ✅ openrpa-review → Task 4
- ✅ openrpa-csharp → Task 5
- ✅ openrpa-polisoft-deep → Task 6
- ✅ 9 notes dosyası → Task 7-15
- ✅ templates/README.md → Task 16
- ✅ fork-differences.md → Task 17

**Placeholder taraması:** Yok — her task somut içerik gereksinimleri içeriyor.

**Tip tutarlılığı:** Tüm dosyalar Markdown, şablon tutarlı.
