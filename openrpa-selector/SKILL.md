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

<!-- Grid/Liste satırı — idx ile -->
<ctrl type="DataItem" idx="2" />
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

Chrome/Firefox (Sigorta web portalleri)
  → NativeMessaging provider
  → id > data-testid > css > xpath önceliği

Office
  → Windows provider + Office COM
```
