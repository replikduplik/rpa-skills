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
| Chrome / Firefox — Epoch/X (YENİ STANDART) | `Playwright` aktiviteleri |
| Chrome / Firefox — Vanilla OpenRPA | `NativeMessaging (NM)` |
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
│   ├── HyperNodeInsertBusinessLog  → EntryState: "2" (Failed), Values: { "hata": ex.Message }
│   ├── CreateYouTrackIssue         → [KZ-12] ZORUNLU — her Catch'te olmalı
│   │     Summary: workflowAdi + " - " + ex.Message
│   │     Description: ex.Message + "\n" + ex.StackTrace
│   ├── [Müşteriye uygun Türkçe hata mesajı gönder — ham ex.Message gönderme! KZ-13]
│   └── UpdateWorkitem  → state: "failed"
└── Finally
    └── CloseApplication  → temizlik (uygulama açık kalmasın)
```

Detay: `notes/CreateYouTrackIssue.md`

### 2. Retry (Geçici Hata Toleransı)

**Önce hata tipini ayırt et:**
- **İş Hatası (BusinessException):** Geçersiz TC kimlik, boş poliçe no, müşteri bulunamadı → retry YAPMA, direkt `failed` yap
- **Sistem Hatası (SystemException):** Timeout, ağ kopması, SAP yavaşlığı → retry YAP

```
Retry karar ağacı:
Exception geldi
├── İş Hatası (BusinessException): geçersiz TC, boş poliçe, müşteri bulunamadı
│   → ThrowBusinessRuleException aktivitesi ile fırlat
│   → retry YAPMA, WorkItem = "failed"
└── Sistem Hatası (SystemException): timeout, ağ kopması, SAP yavaşlığı
    → InvokeWorkflow'u TryCatch içinde tekrar çağır (maks 3 deneme)
    → beklemeyi GetElement Timeout parametresi ile karşıla [KZ-06]
```

> ⚠️ **[KZ-06]** `Thread.Sleep` ve `Delay` aktivitesi **yasaktır**.
> Retry aralarındaki bekleme → GetElement/OpenUrl aktivitelerinin `Timeout` parametresi
> veya `WaitForDownload` gibi olay tabanlı aktivitelerle sağlanır.

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

### 6. Dinamik Bekleme — Delay Yerine Timeout [KZ-06]

> ⛔ `Delay` aktivitesi ve `Thread.Sleep` tüm workflow'larda **yasaktır** [KZ-06].
> Her bekleme ihtiyacı ilgili aktivitenin `Timeout` parametresi ile karşılanır.

```
// ❌ YASAK
Delay 5000ms

// ✅ DOĞRU — element görünene kadar bekle
GetElement
  selector: "yükleme bitti göstergesi"
  Timeout: 15000ms
  MinResults: 1

// ✅ İndirme bekleme
WaitForDownload
  Timeout: 30000ms

// ✅ SAP ekran geçişi
TypeText → "/nFB01" + "{Enter}"
GetElement → hedef ekran elementi (Timeout: 10000ms)
```

### 7. MetricLog — Her UI Etkileşiminde Zorunlu [KZ-07]

Her `GetElement`, `ClickElement`, `TypeText`, `GetText`, `OpenUrl` ve benzeri UI aktivitesi
bir `MetricLog` bloğu içine alınmalıdır.

```
MetricLog
  Name: "policeNo_GetElement"     ← [nesne]_[eylem] formatı
  ThrowIfError: false              ← [HYS-01] false olması normaldir
  Body:
    └── GetElement → poliçe no alanı (Timeout: 10000ms)
```

Detay: `notes/MetricLog.md`

---

### 8. Business Log [KZ-02, KZ-03, KZ-18]

Her kritik iş adımı `HyperNodeInsertBusinessLog` + `HyperNodeUpdateBusinessLog` çiftiyle izlenir.
İki temel kural:

- Her `Insert` mutlaka bir `Update` ile kapatılmalı — aksi hâlde kayıt sonsuza "InProcess" kalır.
- Birden fazla Insert kullanılıyorsa her biri **farklı Identifier** almalı — aynı Identifier ikinci kez kullanılırsa ilk kayıt üzerine yazılır.

```
HyperNodeInsertBusinessLog  Identifier: "ana_islem"  EntryState: "0"
[... iş adımları ...]
HyperNodeUpdateBusinessLog  Identifier: "ana_islem"  EntryState: "1"  ← aynı Identifier
```

Key isimlerinde tip prefixi kullanma: `str_musteri_adi` değil `musteri_adi` [KZ-02].

Detay: `notes/BusinessLog.md`

---

### 9. GetTextResource — Sabit Metin Yasağı [KZ-14]

Aktivite parametrelerinde, log mesajlarında veya mail içeriklerinde **sabit string kullanılamaz**.
Tüm serbest metinler `GetTextResource` aktivitesi ile merkezi kaynak dosyasından çekilmelidir.

```
❌ YANLIŞ — inline string [KZ-14 ihlali]
HyperNodeInsertBusinessLog
  Subject: "İşlem başarıyla tamamlandı"

✅ DOĞRU — merkezi kaynak
GetTextResource
  key: "islem_basarili"
  → Result → mesajDegiskeni

HyperNodeInsertBusinessLog
  Subject: mesajDegiskeni
```

> **Not:** `GetTextResource` aktivitesi projenizde henüz yoksa, aktivitenin eklenmesi gerektiğini
> bulguda belirt. Kural, aktivitenin varlığından bağımsız olarak geçerlidir.

---

### 10. Başlangıç Doğrulaması (Precondition Check)

Workflow başında, herhangi bir iş adımından önce gerekli koşulları kontrol et.
Bu sayede hata, gerçek işlem başlamadan yakalanır ve daha hızlı teşhis edilir.

```
// === WORKFLOW BAŞLANGICINDA KONTROL ===
TryCatch
└── Try
    ├── GetRpaSetting Key: "pdf_klasoru_yolu"  → str_PdfKlasoru
    ├── GetRpaSetting Key: "excel_dosya_yolu"  → str_ExcelYolu
    │
    ├── // Klasör mevcut mu?
    │   If [Not System.IO.Directory.Exists(str_PdfKlasoru)]
    │     ThrowBusinessRuleException "PDF klasörü bulunamadı: " + str_PdfKlasoru
    │
    ├── // Klasörde işlenecek dosya var mı?
    │   If [System.IO.Directory.GetFiles(str_PdfKlasoru, "*.pdf").Length = 0]
    │     ThrowBusinessRuleException "PDF klasörü boş, işlenecek dosya yok"
    │
    ├── // Excel dosyası mevcut ve erişilebilir mi?
    │   If [Not System.IO.File.Exists(str_ExcelYolu)]
    │     ThrowBusinessRuleException "Excel dosyası bulunamadı: " + str_ExcelYolu
    │
    ├── // Config değerleri geçerli mi?
    │   If [String.IsNullOrWhiteSpace(str_PdfKlasoru)]
    │     ThrowBusinessRuleException "pdf_klasoru_yolu ayarı boş"
    │
    └── // Tüm kontroller geçti, iş akışına devam et
        HyperNodeInsertBusinessLog  ← ancak şimdi başlat
```

**Neden önemli?** Precondition check olmadan, hata 50. adımda yakalanır ve o ana kadar
yapılan tüm iş boşa gitmiş olabilir. Başlangıçta kontrol = erken hata = az kayıp.

---

### 11. Veri Doğrulama

İşlem öncesi gelen veriyi doğrula. `If` + `ThrowBusinessRuleException` tercih et:

```
// Kimlik doğrulama
If [String.IsNullOrWhiteSpace(str_TcKimlik) OrElse str_TcKimlik.Trim.Length <> 11]
  ThrowBusinessRuleException "Geçersiz TC Kimlik: " + str_TcKimlik

// Tutar doğrulama
If [Not IsNumeric(str_Prim) OrElse Convert.ToDecimal(str_Prim) <= 0]
  ThrowBusinessRuleException "Geçersiz prim tutarı: " + str_Prim
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
| `Delay` | ⛔ [KZ-06] Yasak — GetElement Timeout veya olay tabanlı bekleme kullan |
| `TryCatch` | Hata yakalama |
| `ForEach / BreakableLoop` | Döngü |
| `Foreach DataRow` | DataTable satır döngüsü (Epoch/X) |
| `Timeout` | Toplam süre sınırı sarmalayıcı (Epoch/X) |
| `Get and Update RPA Settings` | Ayar okuma/yazma (Epoch/X) |
| `EpochxWriteLine` | Debug output — Console.WriteLine yerine (Epoch/X) |
| `WriteLine` | ⛔ [KZ-08] Yasak — Console.WriteLine ile aynı kapsam, production'da kullanılmaz |
| `Match` | Regex tek eşleşme kontrolü — InvokeCode ile Regex.Match() yerine kullan [KZ-09] |
| `Matches` | Regex tüm eşleşmeler döngüsü — ForEach + Regex yerine kullan [KZ-09] |
| `Replace` | Regex ile metin değiştirme — InvokeCode ile Regex.Replace() yerine kullan [KZ-09] |
| `Comment Out` | Aktiviteyi geçici devre dışı bırak (Epoch/X) |

**TypeText özel tuşlar:** `{Enter}`, `{Tab}`, `{F4}`, `{F8}`, `{Escape}`, `{Ctrl+a}`, `{Alt+F4}`

### TypeText vs Assign item.Value — Doğru Tercih

UI elementine değer yazmanın iki yolu var; hangisini kullanacağını şöyle belirle:

```
GetElement → elem (alanı bul)

❌ YANLIŞ — Assign ile doğrudan atama
   Assign: elem.Value = "TC123456789"
   Sorun: TypeText aktivitesinin bypass edilmesi MetricLog'u da devre dışı bırakır,
          bazı uygulamalarda change event tetiklenmez, platform best practice değil

✅ DOĞRU — TypeText ile yaz
   MetricLog Name: "tcField_TypeText"
   └── TypeText: Text="TC123456789"
   Avantaj: MetricLog ile ölçülebilir, change event tetiklenir, test edilebilir
```

**İstisna:** Değeri okumak (yazmak değil) için `item.Value` kullanmak doğrudur:
```
GetElement → elem
Assign: okunanDeger = elem.Value   ← okuma için OK
```

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
GetElement      → sonuç satırı (Timeout: 10000ms)  ← Delay yerine Timeout kullan [KZ-06]
  MinResults: 1
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
GetElement      → hedef ekran yüklendi mi? (Timeout: 10000ms)  ← Delay yasak [KZ-06]
```

### Business Log — HyperNode [KZ-02, KZ-03, KZ-18]

İş akışının kritik adımları `HyperNodeInsertBusinessLog` + `HyperNodeUpdateBusinessLog` ile izlenir.

**Temel kullanım:**
```
HyperNodeInsertBusinessLog
  Identifier: "islem_basla"       ← ConcurrentDictionary anahtarı
  EntryState: "0"                 ← 0=InProcess, 1=Succeeded, 2=Failed
  Role: "0"                       ← 0=Parent (yeni groupId), 1=Child (mevcut groupId)
  GroupId: "police_grubu"
  Subject: GetTextResource("islem_basladi")
  Values:
    policeNo: policeNoValue        ← [KZ-02] "str_policeNo" YAZMA! tip prefixi yasak
    musteriAdi: musteriAdiValue
  IsAsync: true
  IfErrorThrow: false              ← [HYS-01] false olması normaldir

[... iş adımları ...]

HyperNodeUpdateBusinessLog
  Identifier: "islem_basla"       ← Insert ile aynı Identifier
  EntryState: "1"                 ← 1=Succeeded
  Values:                         ← [HYS-03] boş Values normaldir
    sonuc: sonucDegeri
```

**[KZ-02] Key isimlerinde tip prefixi kullanma:**
```
❌ str_musteri_adi, int_kayit_sayisi, dt_islem_tarihi
✅ musteri_adi, kayit_sayisi, islem_tarihi
```

**[KZ-03] Business log ID'si GUID ile üretilir:**
Aktivite kendi GUID'ini otomatik oluşturur. `Identifier` alanına
`DateTime.Now.ToString(...)` gibi bir şey **yazma** — bu alan sadece bir anahtar etiketidir.

**Kritik: Her Insert mutlaka Update ile kapatılmalı:**
```
❌ Insert var, Update yok → kayıt sonsuza "InProcess" kalır
✅ Her Insert → Try sonunda veya Catch'te Update çağrılmalı

❌ Aynı Identifier iki Insert'te → ilk kayıt üzerine yazılır, güncellenemiyor
✅ Her Insert için farklı Identifier:
     str_LogID_Ana    = Guid.NewGuid().ToString()
     str_LogID_PdfDon = Guid.NewGuid().ToString()
```

Detay: `notes/BusinessLog.md`

---

### Office — Excel

```csharp
// InvokeCode — Excel COM ile toplu okuma + ZORUNLU cleanup
Microsoft.Office.Interop.Excel.Application xl = null;
Microsoft.Office.Interop.Excel.Workbook wb = null;
Microsoft.Office.Interop.Excel.Worksheet ws = null;
try
{
    xl = (Microsoft.Office.Interop.Excel.Application)
        System.Runtime.InteropServices.Marshal.GetActiveObject("Excel.Application");
    wb = xl.ActiveWorkbook;
    ws = (Microsoft.Office.Interop.Excel.Worksheet)xl.ActiveSheet;
    int sonSatir = ws.UsedRange.Rows.Count;
    var tablo = new System.Data.DataTable();
    // ... sütun ve satırları oku
    ciktiDataTable = tablo;
}
finally
{
    // ⚠️ COM nesnelerini mutlaka serbest bırak — yoksa EXCEL.EXE arka planda kalır
    // ve sonraki çalışmada "dosya kilitli" hatası alırsın
    if (ws != null) System.Runtime.InteropServices.Marshal.ReleaseComObject(ws);
    if (wb != null) System.Runtime.InteropServices.Marshal.ReleaseComObject(wb);
    if (xl != null)
    {
        xl.Quit();
        System.Runtime.InteropServices.Marshal.ReleaseComObject(xl);
    }
    ws = null; wb = null; xl = null;
    GC.Collect();
    GC.WaitForPendingFinalizers();
}
```

### Mail — Outlook / Graph API / SendEmail

**Epoch/X'te mail gönderme seçenekleri:**
```
SendEmail          → HyperNode API üzerinden (basit, tercih edilen)
NewMailItem        → Outlook COM ile (Outlook kurulu gerekir)
SendOutlookMailGraph → Microsoft Graph API (OAuth, Outlook gerektirmez)
```

**[KZ-11] Veri Gizliliği — Zorunlu:**
E-posta içeriğinde kişisel veri, kimlik bilgisi veya finansal veri varsa
gönderim öncesinde maskelenmeli veya şifrelenmelidir.

```
❌ YANLIŞ — ham kişisel veri [KZ-11]
SendEmail
  Body: "TC: " + tcKimlik + " Banka Hesabı: " + ibanNo

✅ DOĞRU — maskelenmiş veri
SendEmail
  Body: "TC: ***" + tcKimlik.Substring(7) + " IBAN: TR**...***" + ibanNo.Substring(20)
```

**Graph API parametreleri GetRpaSetting ile alınmalı [KZ-01]:**
```
TenantId     ← GetRpaSetting("graph_tenant_id")
ClientId     ← GetRpaSetting("graph_client_id")
ClientSecret ← GetRpaSetting("graph_client_secret")
UserEmail    ← GetRpaSetting("robot_email")
```

**Klasik SMTP (PowerShell):**
```powershell
# InvokePowerShell — SMTP ile mail (Epoch/X'te InvokeCode yasak [KZ-09])
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
| SAP ekran değişmedi | Delay kullanımı | Ekran geçişi sonrası GetElement Timeout ile bekle [KZ-06] |
| Designer açık production'da | RAM şişiyor | Production'da designer'ı kapat |

---

## ⚠️ Platform Uyarıları

### DeleteAllRows — KULLANMA (Bilinen Bug)

`DeleteAllRows` aktivitesinin döngü koşulunda hata var (`i >= count` hiçbir zaman doğru olmaz).
Sonuç: aktivite çalışıyor gibi görünür ama **hiçbir satırı silmez**.

```
❌ YANLIŞ
DeleteAllRows DataTable: myTable   ← satırları silmez, sessizce geçer

✅ DOĞRU — alternatifler
myTable = new DataTable()          ← yeni boş tablo
// veya
myTable.Clear()                    ← tüm satırları sil, yapıyı koru
```

### ExecuteNonQuery / ExecuteQuery — CommandTimeout Karışıklığı

`CommandTimeout` parametresi **milisaniye** cinsinden belirtilir, ancak platform bunu
**saniye** olarak işler. Varsayılan `30000` → aslında ~8 saatlik bir zaman aşımı anlamına gelir.

```
❌ Yanıltıcı
CommandTimeout: 30000   ← 30000 ms gibi görünür, saniye olarak yorumlanır (~8 saat!)

✅ Makul değer
CommandTimeout: 30      ← 30 saniye
```

### UpdateFromDataTable (Database) — SQL Injection Riski

`TableName` parametresi doğrudan SQL sorgusuna ekleniyor. Dış kaynaklı veya kullanıcı
kontrolünde bir string **asla** bu parametreye geçirilmemeli.

```
✅ Güvenli kullanım
TableName: "POLICE_LOG"   ← sabit string veya GetRpaSetting ile alınmış değer
```
