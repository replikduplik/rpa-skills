---
name: openrpa-workflow
description: >
  Use when designing, building, debugging, or reviewing any OpenRPA robot workflow.
  Covers activity selection, error handling, retry logic, credential management,
  logging, and idempotency for desktop, SAP, Office, mail, Polisoft, Open insurance
  software, and insurance web portals. Triggers on: "hangi aktivite", "workflow hata",
  "robot çalışmıyor", "SAP otomasyon", "Polisoft robot", "sigorta sitesi", "Excel robotu",
  "mail gönder", "best practice", "teklif girişi", "poliçe", "hasar".
---

# OpenRPA Workflow Geliştirme

## Genel Mimari

OpenRPA = **Windows Workflow Foundation (WF)** + plugin tabanlı provider sistemi.
Her robot = `.xaml` dosyası. Yapı tercihi: **Sequence** (Flowchart yerine).

### Provider Seçim Tablosu

| Hedef | Provider |
|---|---|
| Windows masaüstü (Polisoft, Open, ERP) | `Windows` (UIAutomation) |
| SAP GUI | `Windows` — SAP selector sözdizimi |
| Chrome / Firefox (sigorta web portalleri) | `NativeMessaging (NM)` |
| Internet Explorer | `IE` |
| Java uygulamaları | `Java` |
| Excel / Word / Outlook | `Office` + COM Interop |
| Görüntü tabanlı son çare | `Image` |

---

## RPA Best Practice — Temel Kurallar

### 1. Exception Handling (Zorunlu)

Her ana blok **TryCatch** içinde olmalı. İstisnasız.

```
TryCatch
├── Try
│   └── [iş adımları]
├── Catch (Exception e)
│   ├── Log aktivitesi  → level: Error, message: e.Message + e.StackTrace
│   ├── Screenshot      → kanıt için ekran görüntüsü al
│   └── UpdateWorkitem  → state: "failed", error: e.Message
└── Finally
    └── CloseApplication  → temizlik (uygulama açık kalmasın)
```

### 2. Retry (Geçici Hata Toleransı)

Ağ hataları ve geçici ekran donmaları için:

```
retry_sayac = 0
While (retry_sayac < 3)
  TryCatch
    Try
      [kritik adım]
      Break  ← başarılıysa döngüden çık
    Catch
      retry_sayac = retry_sayac + 1
      Delay 2000ms
```

### 3. Credential Yönetimi (Güvenlik)

**Asla** username/password'ü workflow'a hardcode etme.

```csharp
// InvokeCode — OpenFlow'dan credential al
var cred = await global::OpenRPA.Interfaces.global.webSocketClient
    .GetCredential("polisoft-login");
string kullanici = cred.username;
string sifre     = cred.password;
```

OpenFlow UI → Credentials → yeni kayıt ekle → workflow'da bu adı kullan.

### 4. Logging (Denetim İzi)

```
Log aktivitesi
  message: "[{DateTime.Now}] [{WorkItemId}] Teklif No: {TeklifNo} işleniyor"
  level: Info
```

Her kritik adım öncesi/sonrası logla. Sigorta süreçlerinde yasal denetim için önemli.

### 5. Idempotency (Tekrar Çalıştırılabilirlik)

Aynı kaydı iki kez işlemeden önce kontrol et:

```
GetElement → "Teklif No" alanı
If (alan boş değil ve değer = beklenen)
  → zaten işlenmiş, WorkItem = "successful" yap, geç
Else
  → işleme devam et
```

### 6. Dinamik Bekleme (Fixed Delay Yerine)

```
// KÖTÜ
Delay 5000ms  ← körü körüne bekle

// İYİ — element görünene kadar bekle
GetElement
  selector: "yükleme bitti göstergesi"
  Timeout: 15000ms
  MinResults: 1
```

### 7. Veri Doğrulama

İşlem öncesi gelen veriyi doğrula:

```csharp
// InvokeCode
if (string.IsNullOrWhiteSpace(tcKimlik) || tcKimlik.Length != 11)
    throw new Exception($"Geçersiz TC Kimlik: '{tcKimlik}'");
if (!decimal.TryParse(prim, out decimal primTutar) || primTutar <= 0)
    throw new Exception($"Geçersiz prim tutarı: '{prim}'");
```

---

## Temel Aktiviteler

| Aktivite | Kullanım |
|---|---|
| `OpenApplication` | Uygulama aç / odakla / konumlandır |
| `CloseApplication` | Finally bloğunda temizlik |
| `GetElement` | UI element bul (RPA'nın çekirdeği) |
| `ClickElement` | Tıkla (Virtual veya gerçek) |
| `Click Element and Verify` | Tıkla + doğrula + retry (Epoch/X) |
| `TypeText` | Klavye girdisi |
| `InvokeCode` | C# satır içi kod |
| `InvokePowerShell` | PowerShell script |
| `InvokeWorkflow` | Alt workflow çağır |
| `Delay` | Bekleme (sadece zorunluysa — Epoch/X'te minimize et) |
| `TryCatch` | Hata yakalama |
| `ForEach / BreakableLoop` | Döngü |
| `Foreach DataRow` | DataTable satır döngüsü (Epoch/X) |
| `Timeout` | Toplam süre sınırı sarmalayıcı (Epoch/X) |
| `Get and Update RPA Settings` | Ayar okuma/yazma (Epoch/X) |
| `EpochxWriteLine` | Debug output — Console.WriteLine yerine (Epoch/X) |
| `Comment Out` | Aktiviteyi geçici devre dışı bırak (Epoch/X) |

**TypeText özel tuşlar:** `{Enter}`, `{Tab}`, `{F4}`, `{F8}`, `{Escape}`, `{Ctrl+a}`, `{Alt+F4}`

---

## Epoch/X — Playwright ile Web Otomasyon

Epoch/X'te web otomasyon için NativeMessaging yerine **Microsoft.Playwright** tabanlı aktiviteler kullanılır.

### Playwright Aktiviteleri

```
Playwright Open URL    → tarayıcı aç + URL'ye git
Playwright Click       → elemente tıkla
Playwright Fill        → alana değer gir (TypeText yerine)
Playwright Get         → element değerini oku
Playwright Exists      → element var mı? (bool döner)
Playwright Close Tab   → mevcut sekmeyi kapat
Playwright Close Browser → tüm tarayıcıyı kapat
```

### Playwright Selector Sözdizimi

```csharp
// CSS selector (önerilen)
"css=input#tcKimlik"
"css=button.btn-primary"
"css=.premium-result span.amount"

// XPath — KRİTİK: tek tırnak kullan, backslash değil!
"xpath=//input[@id='Username']"       // ✅ DOĞRU
"xpath=//input[@id=\'Username\']"     // ❌ YANLIŞ — parse hatası

// Görünen metin
"text=Giriş Yap"

// Placeholder
"placeholder=TC Kimlik Numarası"
```

### Playwright Web Portal Giriş Şablonu

```
Playwright Open URL
  url: "https://portal.sigorta.com/login"

Playwright Fill
  selector: "css=input[name='username']"
  value: {kullanici}

Playwright Fill
  selector: "css=input[name='password']"
  value: {sifre}

Playwright Click
  selector: "css=button[type='submit']"

Playwright Exists
  selector: "css=.dashboard-header"
  Timeout: 15000ms   ← giriş başarılı doğrulaması
```

### CAPTCHA Sorunu (Sigorta Portalleri)

Bazı sigorta portalleri (Garanti BBVA, bazı sigorta siteleri) CAPTCHA içeriyor.
Mevcut yaklaşımlar:

```
1. Google Lens / OCR tabanlı çözüm (deneysel, yanlış okuma riski)
2. Manuel müdahale: robot durur, operatör CAPTCHA'yı çözer, devam eder
3. CAPTCHA API servisi entegrasyonu (planlamada)
```

**Dikkat:** CAPTCHA çözümü başarısız olursa hesap kilitlenebilir.
Log'a yaz, WorkItem'ı "manual_intervention" state'ine çek, bildirim gönder.

### Playwright vs NM — Hangisi Ne Zaman

```
Playwright kullan:
  → Yeni Epoch/X workflow yazarken (standart)
  → SPA (React/Angular) siteler
  → Dinamik DOM, AJAX ağır siteler

NM GetElement kullan:
  → Eski workflow'u korurken
  → Playwright toolbox'ta yoksa
```

---

## Sigorta Uygulamaları — Platform Rehberi

### Polisoft (Desktop)

Polisoft, Windows tabanlı sigorta yönetim yazılımıdır. UIAutomation provider kullanılır.

**Giriş akışı:**
```
OpenApplication → selector: app="polisoft.exe" veya cls="TfrmGiris"
GetElement      → kullanıcı adı alanı
TypeText        → kullanici + "{Tab}"
GetElement      → şifre alanı
TypeText        → sifre + "{Enter}"
GetElement      → ana menünün yüklenmesini bekle (Timeout: 15000ms)
```

**Teklif modülü — veri girişi:**
```
OpenApplication → Polisoft ana ekranı
GetElement      → "Teklif" menüsü
ClickElement    → menüye tıkla
GetElement      → "Yeni Teklif" veya "Teklif Giriş" formu yüklendi mi?
TypeText        → TC Kimlik alanına
TypeText        → "{Tab}" → diğer alanlara geç
[Her alan için GetElement + TypeText tekrarla]
GetElement      → "Kaydet" / "Hesapla" butonu
ClickElement    → kaydet
GetElement      → başarı mesajı veya teklif numarası
```

**Poliçe sorgulama:**
```
GetElement      → poliçe no alanı
TypeText        → policeno + "{Enter}"
Delay           → 1000ms (sonuç yüklensin)
GetElement (MaxResults=1) → sonuç satırı
```

**Kritik ipuçları:**
- Polisoft ekranları arası geçişte **mutlaka** GetElement ile yükleme bekle (Delay değil)
- Modal dialog açılırsa önce onu kapat (GetElement + ClickElement "Tamam")
- Oturum zaman aşımı: 30 dakikada bir KeepAlive GetElement yap

### Open (Sigorta Yazılımı)

Open, benzer Windows tabanlı sigorta uygulamasıdır. UIAutomation provider.

```
OpenApplication → selector: app="open.exe" veya pencere başlığı ile
GetElement      → login formu
TypeText        → kullanici + "{Tab}" + sifre + "{Enter}"
GetElement      → dashboard yüklendi (Timeout: 20000ms)
```

**Hasar girişi:**
```
GetElement      → "Hasar" / "Hasar Bildirimi" menüsü
ClickElement    →
GetElement      → hasar giriş formu
TypeText        → poliçe no + "{Tab}"
GetElement      → müşteri bilgisi otomatik doldu (bekleme)
[Hasar detaylarını doldur]
GetElement      → "Kaydet" butonu
ClickElement    →
GetElement      → "Hasar No: XXXXX" onay mesajı → hasar no'yu oku
```

### Sigorta Web Portalleri (Chrome/Firefox)

Allianz, Axa, Mapfre, Anadolu Sigorta, Generali vb. web portalleri.

**NativeMessaging provider — giriş:**
```
OpenApplication → selector: browser="chrome", url="https://portal.allianz.com.tr/*"
GetElement      → id="username" veya name="username"
TypeText        → kullanici + "{Tab}"
GetElement      → id="password"
TypeText        → sifre + "{Enter}"
GetElement      → dashboard elementi (giriş başarısını doğrula, Timeout:20000)
```

**Teklif alma — web portal:**
```
GetElement      → "Yeni Teklif" / "New Quote" butonu
ClickElement    →
GetElement      → araç tipi / sigorta branşı dropdown → tag="select", id="branchType"
[SelectOption aktivitesi kullan — DropDown için]
GetElement      → TC/VKN alanı
TypeText        → kimlik + "{Tab}"
GetElement      → otomatik doldurulan müşteri bilgisi alanı (Timeout:10000)
[Devam butonu, sonraki adımlar]
GetElement      → prim tutarı elementi → değeri oku ve kaydet
```

**Web scraping — teklif fiyatı karşılaştırma:**
```
ForEach sigorta_sirket in sirket_listesi
  OpenApplication → sigorta_sirket.url
  [Giriş yap]
  [Araç bilgisi gir]
  GetElement      → prim alanı → değeri oku
  UpdateWorkitem  → payload'a prim ekle
  CloseApplication
```

---

## Otomasyon Türüne Göre Rehberler

### SAP Otomasyonu

SAP GUI Script açık olmalı: Tools → Options → Accessibility & Scripting → Enable scripting.

```
OpenApplication → cls="SAP_FRONTEND_SESSION"
GetElement      → T-kodu: name="wnd[0]/tbar[0]/okcd"
TypeText        → "/nFB01" + "{Enter}"
Delay           → 500ms
GetElement      → hedef ekran yüklendi mi?
```

### Office — Excel

```csharp
// InvokeCode — Excel COM ile toplu okuma
var xl = (Microsoft.Office.Interop.Excel.Application)
    System.Runtime.InteropServices.Marshal.GetActiveObject("Excel.Application");
var ws = (Microsoft.Office.Interop.Excel.Worksheet)xl.ActiveSheet;
int sonSatir = ws.UsedRange.Rows.Count;
var tablo = new System.Data.DataTable();
// ... sütun ve satırları oku
ciktiDataTable = tablo;
```

### Mail — Outlook

```powershell
# InvokePowerShell — SMTP ile güvenilir mail
Send-MailMessage `
  -To "mudur@sirket.com" `
  -From "robot@sirket.com" `
  -Subject "Günlük Teklif Raporu - $(Get-Date -Format 'dd.MM.yyyy')" `
  -Body $mailGovde `
  -Attachments "C:\Raporlar\rapor.xlsx" `
  -SmtpServer "mail.sirket.com" -Port 587 -UseSsl
```

---

## Modüler Workflow Tasarımı (Sigorta Örneği)

```
Ana: SigortaTeklifIsle.xaml
├── InvokeWorkflow: KimlikDogrula.xaml     [In: tcKimlik | Out: musteriAdi]
├── InvokeWorkflow: PoliceSorgula.xaml     [In: plaka   | Out: mevcutPolice]
├── If (mevcutPolice boş değil)
│   └── InvokeWorkflow: YenilemeYap.xaml  [In: police, musteriAdi]
└── Else
    └── InvokeWorkflow: YeniTeklifAl.xaml [In: aracBilgi, musteriAdi | Out: teklifNo]
        └── InvokeWorkflow: Bildir.xaml   [In: teklifNo, musteriAdi]
```

Her modül bağımsız test edilebilir ve farklı ana workflow'lardan çağrılabilir.

---

## Sık Yapılan Hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Element bulunamıyor | Timeout kısa veya uygulama yavaş | Timeout 10000–20000ms yap |
| Polisoft oturum kapandı | Keep-alive yok | Her 25 dakikada KeepAlive GetElement |
| Web portal CAPTCHA | Bot koruması | Manuel adım ekle veya 2Captcha entegrasyonu |
| Credentials hardcoded | Güvenlik riski | OpenFlow Credentials kullan |
| Aynı kayıt iki kez girildi | Idempotency yok | İşlem öncesi kayıt var mı kontrol et |
| SAP ekran değişmedi | Delay eksik | Ekran geçişi sonrası GetElement ile bekle |
| Designer açık production'da | RAM şişiyor | Production'da designer'ı kapat |
