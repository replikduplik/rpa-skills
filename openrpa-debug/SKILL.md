---
name: openrpa-debug
description: >
  Use when an OpenRPA workflow throws an error, produces wrong output, or stops
  unexpectedly. Triggers on: error messages, stack traces, "element not found",
  "timeout", "NullReferenceException", "selector kırıldı", "robot takıldı",
  "log analizi", "hata teşhis", "neden çalışmıyor", SAP script errors,
  web portal errors, InvokeCode exceptions, "robot durdu".
---

# OpenRPA Debug — Hata Teşhis Rehberi

## Teşhis Akışı

```
Hata mesajı / semptom
        ↓
1. Hata tipini tanımla (aşağıdaki tabloya bak)
2. Olası sebepleri sırala
3. Adım adım kontrol et
4. Çözüm uygula
5. Tekrar oluşup oluşmadığını test et
```

---

## Hata Tipi Tanımlama Tablosu

| Hata / Semptom | Hızlı Seksiyona Git |
|---|---|
| `Element not found` / `MinResults not met` | → Bölüm 1 |
| `Timeout` / robot dondu / beklemeye geçti | → Bölüm 2 |
| `NullReferenceException` | → Bölüm 3 |
| `COMException` / Office hataları | → Bölüm 4 |
| SAP script hataları | → Bölüm 5 |
| Web portal (NativeMessaging) hataları | → Bölüm 6 |
| Selector kırıldı — dün çalışıyordu | → Bölüm 7 |
| InvokeCode C# exception | → Bölüm 8 |
| Genel iş mantığı hatası (yanlış sonuç) | → Bölüm 9 |

---

## Bölüm 1 — Element Not Found / MinResults Not Met

### Semptomlar
- `Sequence failed — MinResults not met`
- `Could not find element matching selector`
- GetElement timeout doldu, aktivite durdu

### Olası Sebepler (sırayla kontrol et)

1. **Uygulama henüz yüklenmedi** — ekran açık ama element yoktu
2. **Selector yanlış/kırık** — uygulama güncellemesi selector'ı bozdu
3. **Pencere odağı yok** — uygulama minimize veya arka planda
4. **Yanlış provider** — Windows UI için NativeMessaging kullanıldı
5. **Dinamik element** — her açılışta farklı `idx`, `name` üretiliyor
6. **Modal/popup önde** — hedef element modalın arkasında

### Adım Adım Teşhis

```
[ ] 1. Workflow'u manuel çalıştır — element ekranda görünüyor mu?
[ ] 2. GetElement'te Highlight butonu → element renk alıyor mu?
        Hayır → Selector yanlış (Bölüm 7'ye git)
        Evet  → Timing sorunu (Bölüm 2'ye git)
[ ] 3. Timeout değerini artır (10000 → 30000 ms) → hâlâ hata mı?
[ ] 4. OpenApplication / SetFocus ile pencereyi öne al
[ ] 5. provider'ı kontrol et:
        Delphi/Win32 → Windows
        SAP          → SAP GUI Script
        Chrome/FF    → NativeMessaging
[ ] 6. GetElement öncesine Delay(2000) ekle → düzeldi mi?
        Evet → Gerçek çözüm: WaitForElement veya döngü kontrolü
```

### Çözüm Şablonu

```
TryCatch
├── Try
│   ├── GetWorkItem veya önceki adım
│   ├── [Uygulama hazır mı? — döngüyle bekle]
│   └── GetElement
│         selector: [daha stabil selector]
│         Timeout: 20000
│         MinResults: 1, MaxResults: 1
└── Catch (ElementNotFoundException e)
    ├── Log → "Element bulunamadı: " + e.Message
    └── UpdateWorkitem → state: "failed"
```

---

## Bölüm 2 — Timeout / Robot Dondu

### Semptomlar
- Workflow belirli bir noktada takılı kalıyor
- `The operation has timed out`
- Robot "Busy" görünüyor, hiç bitmiyor

### Olası Sebepler

1. **Sonsuz döngü** — While koşulu hiç false olmuyor
2. **Modal bekleniyor** — onay kutusu/popup cevap bekliyor
3. **Ağ gecikmesi** — web sayfası, SAP, veritabanı yavaş
4. **Deadlock** — InvokeCode içinde UI thread'e erişim
5. **OpenFlow bağlantısı kesildi** — WorkItem alınamıyor

### Teşhis Adımları

```
[ ] 1. Log aktiviteleri ekle — tam olarak nerede takıldığını bul
[ ] 2. OpenRPA Designer'da "Stop" — nerede durduğunu gör
[ ] 3. Task Manager: CPU %100 mü? → sonsuz döngü
[ ] 4. Ekranda beklenmedik modal/popup var mı?
[ ] 5. Ağ testi: Manuel aynı işlem ne kadar sürüyor?
[ ] 6. Timeout değerleri gerçekçi mi? (web portal = 30s, SAP = 15s)
```

### Sonsuz Döngü Tespiti

```csharp
// InvokeCode içine sayaç ekle
iterasyon++;
if (iterasyon > 1000)
    throw new Exception($"Döngü limit aşıldı — son değer: {deger}");
```

---

## Bölüm 3 — NullReferenceException

### Semptomlar
- `Object reference not set to an instance of an object`
- `NullReferenceException` — stack trace InvokeCode satırına işaret ediyor

### Kök Neden Analizi

```csharp
// ❌ Hatalı — null kontrolü yok
string deger = item.payload["Alan"].ToString();

// ✅ Doğru — null-safe
string deger = item.payload["Alan"]?.ToString() ?? "";

// ❌ Hatalı — GetElement sonucu null olabilir
element.Click();

// ✅ Doğru
if (element != null)
    element.Click();
else
    throw new Exception("Element null — selector kontrol et");
```

### Kontrol Listesi

```
[ ] WorkItem payload'da beklenen alan var mı? (alan adı yazım hatası?)
[ ] GetElement çıktı değişkeni null mı? (MinResults=0 ise null gelebilir)
[ ] DataTable row'u mevcut mu? (Rows.Count > 0 kontrol ettik mi?)
[ ] Excel hücresi boş mu? (Range.Value null dönebilir)
[ ] InvokeWorkflow Out parametresi set edildi mi?
```

---

## Bölüm 4 — COMException / Office Hataları

### Sık Görülen Hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `RPC server is unavailable` | Excel/Word çökmüş | Process kill, yeniden aç |
| `HRESULT: 0x800AC472` | Excel meşgul (alert var) | DisplayAlerts=false |
| `Cannot access read-only document` | Dosya kilitli | Kilit kontrol, başka kullanıcı |
| `COMException: 0x80010001` | COM thread sorunu | InvokeCode'u STA thread'de çalıştır |

### Excel COM — Güvenli Temizleme

```csharp
// InvokeCode — her zaman Finally'de temizle
Excel.Application xlApp = null;
Excel.Workbook wb = null;
try
{
    xlApp = new Excel.Application();
    xlApp.DisplayAlerts = false;
    xlApp.Visible = false;
    wb = xlApp.Workbooks.Open(dosyaYolu);
    // ... işlemler
}
finally
{
    wb?.Close(false);
    xlApp?.Quit();
    if (wb   != null) System.Runtime.InteropServices.Marshal.ReleaseComObject(wb);
    if (xlApp!= null) System.Runtime.InteropServices.Marshal.ReleaseComObject(xlApp);
    wb    = null;
    xlApp = null;
    GC.Collect();
    GC.WaitForPendingFinalizers();
}
```

---

## Bölüm 5 — SAP Script Hataları

### Sık Hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `SAP GUI Scripting not enabled` | SAP ayarı kapalı | SAP → Options → Scripting → Enabled |
| `No active session` | SAP oturumu yok/kapandı | Oturum kontrol aktivitesi ekle |
| `Element not found` (SAP provider) | Transaction değişti | T-code/ekran adını doğrula |
| `Screen timeout` | SAP idle timeout | Keep-alive aktivitesi ekle |

### SAP Keep-Alive Şablonu

```
While (işlemDevam)
├── [İşlem aktiviteleri]
└── GetElement
      selector: SAP statusbar (her ekranda var)
      Timeout: 5000
      [Bulamazsa → oturum kapandı → yeniden giriş]
```

---

## Bölüm 6 — Web Portal / NativeMessaging Hataları

### Sık Hatalar

| Hata | Semptom | Çözüm |
|---|---|---|
| CAPTCHA | Element bulunamıyor | Manuel müdahale gerekiyor, log'a yaz |
| Session timeout | Giriş sayfasına yönlendirme | Session kontrol aktivitesi ekle |
| Dynamic content | Element yüklenmeden tıklama | WaitForElement, scroll |
| AJAX loading | Spinner görünürken tıklama | Spinner kaybolana kadar bekle |
| Certificate error | Chrome popup | Chrome policy ile bypass |

### AJAX Yükleme Bekleyici

```
// Spinner'ın kaybolmasını bekle
GetElement
  selector: css=[class='loading-spinner'][style='display:none']
  Timeout: 30000
  MinResults: 1
```

### Session Kontrolü Şablonu

```
GetElement → giriş butonu veya kullanıcı adı alanı
If (loginElement != null)
    InvokeWorkflow: GirisYap.xaml
```

---

## Bölüm 7 — Selector Kırıldı

### "Dün Çalışıyordu" Durumu

```
[ ] 1. Uygulama güncellemesi oldu mu? (versiyon değişikliği)
[ ] 2. Kullanıcı arayüzü değişti mi? (yeni alan, farklı pencere adı)
[ ] 3. Ekran çözünürlüğü/DPI değişti mi? (idx bazlı selector bozulur)
[ ] 4. Dil/locale değişti mi? (name bazlı selector bozulur)
[ ] 5. Windows tema güncellemesi mi?
```

### Yeniden Selector Yazma Adımları

```
1. GetElement aktivitesine çift tıkla
2. "Highlight" → element hâlâ turuncu renk alıyor mu?
   Hayır → "Spy" ile yeniden seç
3. Selector editörünü aç — hangi özellik değişti?
4. Stabil selector hiyerarşisine geç:
   automationid > name+type > css > xpath > idx
5. Wildcard kullan (değişen kısım için):
   name='Polisoft*' veya css=[id^='btn_']
```

→ Daha fazla detay için: `openrpa-selector` skill + `notes/GetElement.md`

---

## Bölüm 8 — InvokeCode C# Exception

### Hata Okuma Yöntemi

```
Exception mesajı: [hatanın kendisi]
Stack trace:
  at OpenRPA.InvokeCode... line XX   ← kendi kodun
  at System.Linq...                  ← .NET iç kütüphane
```

**Satır numarasını bul → InvokeCode içinde o satıra git.**

### Sık C# Hataları

| Hata | Sebep | Çözüm |
|---|---|---|
| `FormatException` | DateTime.Parse yanlış format | `DateTime.ParseExact("31.01.2025", "dd.MM.yyyy", null)` |
| `InvalidCastException` | Yanlış tip dönüşümü | `Convert.ToInt32()` yerine `int.Parse()` veya `as int?` |
| `KeyNotFoundException` | Dictionary/JObject'te olmayan key | `.ContainsKey()` kontrolü ekle |
| `IndexOutOfRange` | DataTable/array boş | `Rows.Count > 0` kontrolü |
| `ArgumentNullException` | Null string'e işlem | `?.` null-conditional kullan |

→ Daha fazla C# kodu için: `openrpa-csharp` skill

---

## Bölüm 9 — Yanlış Sonuç (İş Mantığı Hatası)

### Yaklaşım: Bisection Debug

```
1. Workflow'u ortadan böl
2. Her yarının çıktısını Log aktivitesiyle yaz
3. Hatalı yarıyı tekrar böl
4. Sorunlu aktiviteyi bul
```

### Log Stratejisi

```
Log → "BAŞLADI: " + DateTime.Now
Log → "Veri: " + policeNo + " | " + plaka
[aktivite]
Log → "Sonuç: " + sonucDeger
```

Çıktı OpenRPA log panelinde görünür (View → Output).

### Değişken İzleme

InvokeCode içinde:
```csharp
// Geçici debug çıktısı
Console.WriteLine($"DEBUG policeNo={policeNo}, brans={brans}");
```

---

## Hızlı Kontrol Listesi — Her Hata İçin

```
[ ] Tam hata mesajını kopyala (mesaj + stack trace)
[ ] Hangi aktivitede durdu? (kırmızı hata simgesi)
[ ] Son başarılı adım neydi? (log'dan bak)
[ ] Değişkenler beklenen değerde mi? (InvokeCode Console.WriteLine)
[ ] TryCatch var mı? Yoksa ekle, sonra tekrar çalıştır
[ ] Ortam aynı mı? (uygulama açık, giriş yapılmış, pencere odakta)
[ ] Aynı hata her seferinde mi oluyor? (intermittent mi?)
```

---

## Log Seviyeleri

| Seviye | Ne Zaman |
|---|---|
| `Information` | Normal akış adımları |
| `Warning` | Beklenmedik ama devam eden durum |
| `Error` | İşlem başarısız — Catch bloğunda |
| `Debug` | Geliştirme sırasında değişken değerleri |

**Production'da Debug logları kapat** — performans etkiler.
