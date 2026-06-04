---
name: openrpa-selector
description: >
  Use when writing, debugging, or fixing OpenRPA selectors for any application.
  Covers Windows UIAutomation, SAP GUI, Chrome/Firefox (NativeMessaging), IE, Office,
  Polisoft, Open insurance software, and insurance web portals. Triggers on:
  "element not found", "selector çalışmıyor", "element bulunamıyor", "Selector Window",
  "GetElement hata", "spy", "highlight", "selector kırıldı", "hangi selector",
  "Polisoft alanı bulamıyor", "web sitesi elementi".
---

# OpenRPA Selector Rehberi

Selector = OpenRPA'nın UI elementini tanımladığı XML tabanlı tanımlayıcı.
**Doğru selector = kararlı robot. Kırılgan selector = sürekli bakım.**

---

## Selector Anatomisi

```xml
<wnd app="polisoft.exe" cls="TfrmAna" />
<ctrl name="btnTeklif" type="Button" />
```

Her `<tag>` bir element veya üst kapsayıcıdır.

### Tanımlayıcı Öncelik Sırası (En İyiden En Kötüye)

```
1. automationid / id / data-testid   ← en kararlı, nadiren değişir
2. name + type kombinasyonu
3. css selector (web için)
4. innertext (metin değişirse kırılır)
5. idx (sıra değişirse kırılır)
6. title — dinamikse wildcard zorunlu
```

### Wildcard Kullanımı

```xml
title="Polisoft *"        ← "Polisoft 2024", "Polisoft Pro" vb. hepsine uyar
class="*primary*"         ← class içinde "primary" geçen her element
url="https://portal.*"    ← alt domain farklı olsa da çalışır
```

---

## Provider'a Göre Selector Sözdizimi

### Windows (UIAutomation) — Polisoft, Open, Desktop

```xml
<!-- Uygulama penceresi -->
<wnd app="polisoft.exe" cls="TfrmGiris" />

<!-- automationid varsa — en güvenilir -->
<ctrl automationid="edtKullaniciAdi" type="Edit" />

<!-- automationid yoksa — Delphi/VCL uygulamalar için -->
<ctrl name="Kullanıcı Adı" type="Edit" />
<ctrl name="Şifre" type="Edit" idx="0" />

<!-- Buton -->
<ctrl name="Giriş" type="Button" />
<ctrl name="btnGiris" type="Button" />

<!-- Grid/Liste satırı — idx KULLANMA, kırılır! -->
<!-- ❌ <ctrl type="DataItem" idx="2" />  → uygulama değişince bozulur -->
<!-- ✅ Bunun yerine name, automationid veya GetElement(MaxResults) ile tüm satırları al -->
<ctrl type="DataItem" name="PN-2024-98765" />   <!-- stabil: kayıt adı ile eşleş -->
```

**Delphi/VCL uygulamalar (Polisoft genellikle Delphi):**
- `cls` değerleri `TForm`, `TEdit`, `TButton`, `TDBGrid` gibi başlar
- `name` çoğu zaman component adıdır (Delphi Object Inspector'dan görülür)
- `automationid` çoğu Delphi uygulamasında yoktur — `name` + `type` kullan

```xml
<!-- Polisoft teklif formu örneği -->
<wnd app="polisoft.exe" cls="TfrmTeklif" title="Teklif Giriş*" />
<ctrl cls="TEdit" name="edtTCKimlik" />
<ctrl cls="TEdit" name="edtPlaka" />
<ctrl cls="TButton" name="btnHesapla" />
<ctrl cls="TDBGrid" name="dbgTeklifler" />
```

### SAP GUI

SAP GUI Script etkin olmalı: Tools → Options → Accessibility & Scripting → Enable scripting.

```xml
<wnd app="saplogon.exe" cls="SAP_FRONTEND_SESSION" />

<!-- T-kodu alanı -->
<ctrl name="wnd[0]/tbar[0]/okcd" type="SAPTextField" />

<!-- Giriş ekranı kullanıcı alanı -->
<ctrl name="wnd[0]/usr/txtRSYST-BNAME" type="SAPTextField" />
<ctrl name="wnd[0]/usr/pwdRSYST-BCODE" type="SAPPasswordField" />

<!-- Execute / F8 butonu -->
<ctrl name="wnd[0]/tbar[1]/btn[8]" type="SAPButton" />

<!-- Grid 3. satır, 2. sütun -->
<ctrl name="wnd[0]/usr/tblGRID/txtFELD[1,2]" type="SAPTextField" />
```

**SAP element yolu nasıl bulunur:**
- SAP'ta elementi sağ tıkla → "Technical Info" → component path'i kopyala
- Veya Selector Window → Spy → SAP elementine tıkla

### Chrome / Firefox — Sigorta Web Portalleri

```xml
<!-- Tarayıcı + sayfa -->
<nativeapp browser="chrome" url="https://portal.allianz.com.tr/*" />

<!-- id ile (en iyi) -->
<ctrl tag="input" id="tcKimlikNo" />

<!-- name ile -->
<ctrl tag="input" name="vehiclePlate" />

<!-- class ile (kısmi eşleme) -->
<ctrl tag="button" class="*btn-primary*" />

<!-- Görünen metin ile -->
<ctrl tag="button" innertext="Teklif Al" />
<ctrl tag="a" innertext="Giriş Yap" />

<!-- Dropdown (select) -->
<ctrl tag="select" id="branchType" />   ← SelectOption aktivitesi ile kullan

<!-- data-* özelliği -->
<ctrl tag="button" data-testid="submit-quote" />

<!-- CSS selector — karmaşık yapı için -->
<ctrl css="table.premium-table tbody tr:nth-child(1) td.amount" />

<!-- XPath — son çare -->
<ctrl xpath="//button[contains(@class,'submit') and normalize-space()='Hesapla']" />
```

**Sigorta portalı öncelik sırası:** `id` > `data-testid` > `name` > `css` > `innertext` > `xpath`

**Dinamik sayfa için bekleme:**
```
GetElement
  css: "div.loading-spinner"
  MinResults: 0
  Timeout: 500ms

GetElement
  id: "premiumResult"
  Timeout: 20000ms   ← prim hesabı yavaş olabilir
```

### Internet Explorer

```xml
<ie url="https://eski-portal.sigortasirketi.com/*" />
<ctrl tag="input" id="txtTCKimlik" />
<ctrl tag="a" innertext="Devam" />
```

### Office Selectorları

```xml
<!-- Excel -->
<wnd app="EXCEL.EXE" title="* - Excel" />
<ctrl name="A1" type="Cell" />

<!-- Outlook -->
<wnd app="OUTLOOK.EXE" cls="rctrl_renwnd32" />
<ctrl name="Yeni E-posta" type="Button" />
```

---

## Selector Window Kullanımı

1. GetElement → **"Open Selector"** tıkla
2. **Spy** → ekranda hedef elementi tıkla → selector otomatik oluşur
3. **Highlight** → doğru elementi vurguluyor mu?
4. **Test** → kaç element bulduğunu gör (genelde 1 olmalı)
5. Gereksiz özellikleri kaldır → dinamik değerlere wildcard ekle

---

## Kırık Selector — Teşhis Adımları

**"Element Not Found" hatası alıyorum:**

```
1. Uygulama açık mı? Doğru pencere odakta mı?
   → OpenApplication çalışıyor mu kontrol et

2. Yükleme bekleniyor mu?
   → Timeout artır: 3000 → 15000ms

3. Selector doğru mu?
   → Selector Window → Highlight → buluyor mu?

4. Başlık / ID dinamik mi?
   → title="Polisoft 2024.3" → title="Polisoft *" yap

5. Birden fazla eşleşme mi?
   → idx="0" ekle ya da daha ayırt edici özellik bul

6. Element gizli / scroll gerekiyor mu?
   → ScrollIntoView aktivitesi ekle

7. Modal dialog açık mı?
   → Önce dialog'u kapat (GetElement + ClickElement "Tamam"/"Kapat")
```

### Yaygın Kırılma Sebepleri

| Sebep | Çözüm |
|---|---|
| Polisoft güncellendi | Yeniden spy et |
| Web portal DOM değişti | id/data-testid güncelle |
| Pencere başlığı değişti | Wildcard ekle |
| SAP ekranı değişti | Technical Info ile yeni yolu bul |
| Dil/bölge değişti | name yerine automationid kullan |
| Ekran çözünürlüğü farklı | Image selector yerine UI selector kullan |

---

## ⚠️ Selector'da Asla Bulunmaması Gerekenler

### Hardcoded Oturum Parametreleri (Session ID, Nonce, State)

Web portal selector'larının URL kısmına oturum bazlı dinamik parametreler gömülmesi
son derece yaygın bir hatadır. Robot ilk açılışta çalışır, ikinci çalışmada tamamen kırılır.

```xml
❌ YANLIŞ — oturum parametreleri sabit yazılmış
<nativeapp browser="chrome"
  url="https://portal.dogasigorta.com/login?state=0da6e6d4db864bb3&nonce=e09f2fb1c9" />
<ctrl xpath="//div[@id='HasarAra']//tr[@hasarIslemId='160420261449h3tr']/td[18]" />

Sorun:
- state= ve nonce= her oturumda farklı → ikinci çalışmada element bulunamaz
- hasarIslemId= sabit bir işlem kimliği → robot her seferinde aynı kayda gider

✅ DOĞRU — dinamik parametreler URL'den çıkarılmış
<nativeapp browser="chrome" url="https://portal.dogasigorta.com/login*" />
<ctrl xpath="//div[@id='HasarAra']//tr[td[contains(@class,'hasar-id')]]/td[18]" />

hasarIslemId değeri varsa dinamik değişkenden alınmalı:
  selector: "...hasarIslemId=[out_HasarIslemId]..."
```

**Kontrol listesi — selector URL'lerine bak:**

| Parametre türü | Örnek | Risk |
|---|---|---|
| OAuth state | `state=abc123` | Her oturumda farklı → her seferinde kırılır |
| Nonce | `nonce=xyz789` | Her oturumda farklı |
| İşlem ID | `hasarIslemId=160420` | Sabit → robot hep aynı kayda gider |
| Session token | `sessionId=aBc123` | Sabit → oturum süresi dolarsa kırılır |

**Kural:** Selector URL'lerinde `?` karakterinden sonraki tüm parametreleri sorgula.
Değer sabit ve anlamlıysa (işlem kimliği gibi) → dinamik değişkenden al.
Değer rastgele/oturum bazlıysa → URL'den tamamen çıkar, wildcard kullan.

---

## Sigorta Uygulaması Özel Durumlar

### Polisoft Grid'den Veri Okuma

```xml
<wnd app="polisoft.exe" cls="TfrmTeklifListesi" />
<ctrl cls="TDBGrid" name="dbgTeklifler" />
```

```
GetElement (MaxResults=200) → grid satırlarını al
ForEach item in items
  GetElement (içinde) → belirli sütun
  → değeri al
```

### Web Portalda Prim Sonucu Okuma

```
GetElement → css=".premium-amount" veya id="totalPremium"
  Timeout: 25000ms  ← hesaplama yavaş olabilir
→ element.Value veya element.Text ile prim değerini al
```

### Popup / Modal Yönetimi (Sigorta Portalleri)

```
Pick
├── PickBranch (Trigger: GetElement → "Hata mesajı" popup)
│   └── ClickElement → "Kapat" / "Tamam"
│       Throw new Exception("Portal hata verdi: " + hata_metni)
└── PickBranch (Trigger: GetElement → prim sonuç alanı)
    └── [prim değerini oku, devam et]
```

---

## Hızlı Başvuru

```
Polisoft / Open (Desktop Delphi/VCL)
  → Windows provider
  → cls ile TForm/TEdit/TButton/TDBGrid
  → name = Delphi component adı

SAP
  → Windows provider
  → name = "wnd[0]/usr/..." yolu

Chrome/Firefox — Epoch/X (Playwright) [YENİ STANDART]
  → Playwright aktiviteleri
  → "css=..." > "xpath=..." > "text=..."
  → XPath'te tek tırnak kullan: xpath=//input[@id='x'] (backslash değil!)

Chrome/Firefox — NM GetElement [ESKİ, dönüşüm planlanıyor]
  → NativeMessaging provider
  → id > data-testid > css > xpath önceliği

Office
  → Windows provider + Office COM
```

---

## Epoch/X — Image Selector Sorunları

### Import Sonrası Image Kaybı

Workflow 2.0'a import edildiğinde image tanımları silinir. Bu bilinen bir platform sorunudur.

**Geçici Çözüm — Base64 Dönüşümü:**

```csharp
// 1. Image dosyasını base64'e çevir (InvokeCode ile)
string base64Image = System.Convert.ToBase64String(
    System.IO.File.ReadAllBytes(@"C:\resimler\hedef_element.png"));

// 2. Eski XAML'den image ID'sini al (metin editörü ile aç)
// 3. XAML içindeki ID referansını base64 string ile değiştir
```

**Image.GetElement Yeni Parametreler (Epoch/X):**

| Parametre | Açıklama |
|---|---|
| `retry` | Başarısız olursa tekrar dene |
| `retrydelaycount` | Yeniden deneme aralığı (ms) |
| `cache` | Önceki eşleşmeyi önbellekle |
| `matchmode` | `accurate` / `pyramid` / `pyramidfast` |
| `tracklastmatch` | Son eşleşme konumunu izle |

**Öneri:** `matchmode=pyramid` hız/doğruluk dengesi için iyi başlangıç noktası.
