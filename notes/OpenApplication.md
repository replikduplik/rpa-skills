# OpenApplication

## Ne Yapar

Bir uygulamayı başlatır veya zaten açıksa odaklanır (focus verir). Workflow başında Polisoft, SAP, Excel veya herhangi bir masaüstü uygulamasını hazır hale getirmek için kullanılır.

---

## Parametreler

| Parametre | Açıklama | Varsayılan |
|---|---|---|
| `FileName` | Çalıştırılacak .exe yolu veya uygulama adı | (zorunlu) |
| `Arguments` | Komut satırı argümanları | — |
| `WorkingDirectory` | Çalışma dizini | — |
| `Timeout` | Uygulamanın açılması için ms bekleme | 10000 |
| `WaitForIdle` | Uygulama idle olana kadar bekle | true |
| `Selector` | Açılacak pencerenin selector'ı (var olanı bul) | — |

---

## Açık Uygulama vs Yeni Başlat

```
Selector parametresi verilirse:
  → Önce bu selector'a uyan pencere aranır
  → Bulunursa odaklanır (yeni açmaz)
  → Bulunmazsa FileName ile yeni başlatır

Selector verilmezse:
  → Her seferinde yeni instance açar
```

**Polisoft için:** Selector ver — birden fazla instance istemiyorsun.

---

## Adım Adım Kullanım

1. Toolbox → **OpenApplication** sürükle
2. `FileName` → uygulama yolunu gir
3. `Selector` → Spy ile ana pencereyi seç (varsa odaklan için)
4. `Timeout` → uygulamanın başlaması için gerçekçi süre ver
5. Arkasına `GetElement` ekle — uygulama gerçekten hazır mı kontrol et

---

## Sık Yapılan Hatalar

### 1. Timeout çok kısa

```
❌ Timeout=3000 → Polisoft 5-10 saniye açılıyor → hata
✅ Timeout=15000-30000 → gerçekçi açılış süresi
```

### 2. Selector'sız çoklu instance

```
❌ OpenApplication (Selector yok) → her çalıştırmada yeni Polisoft açılır
   → 5 çalıştırma = 5 Polisoft penceresi

✅ Selector: cls=TForm name=Polisoft*
   → Açıksa odaklanır, kapalıysa açar
```

### 3. Hemen ardından element aramak

```
❌ OpenApplication → GetElement (hemen)
   → Uygulama henüz tam yüklenmedi → "element not found"

✅ OpenApplication → GetElement (giriş ekranı veya menü)
   Timeout=20000 → Yüklenmesini bekler
```

### 4. Yanlış çalışma dizini

```
Semptom: Uygulama açılıyor ama dosya bulamıyor
Çözüm:  WorkingDirectory'yi exe'nin bulunduğu klasöre ayarla
```

---

## Sigorta Sektörü Örnekleri

### Polisoft Aç / Odaklan

```
OpenApplication
  FileName:        "C:\Polisoft\Polisoft.exe"
  Selector:        cls=TForm name=Polisoft*
  Timeout:         20000
  WaitForIdle:     true

// Ardından giriş ekranını bekle
GetElement → loginEkrani
  selector: cls=TForm name=*Login*
  Timeout: 15000
```

### SAP Aç

```
OpenApplication
  FileName:    "C:\Program Files\SAP\FrontEnd\SAPgui\saplogon.exe"
  Selector:    cls=SapWin name=*SAP*
  Timeout:     20000

GetElement → sapAnaEkran
  selector: cls=SapWin
  Timeout: 15000
```

### Excel Dosyası Aç

```
OpenApplication
  FileName:   "excel.exe"
  Arguments:  """C:\Data\policeListesi.xlsx"""
  Timeout:    10000

GetElement → excelPencere
  selector: cls=XLMAIN name=*policeListesi*
  Timeout: 10000
```

### Web Tarayıcı

```
// Chrome ile web portal aç
OpenApplication
  FileName:   "chrome.exe"
  Arguments:  "--new-window https://portal.sigortasirketi.com"
  Timeout:    5000

// Sayfa yüklenene kadar bekle
GetElement → girisButonu
  selector: css=#loginBtn
  Timeout: 30000
```

---

## Uygulama Kapatma

```
// OpenApplication'ın tersi — pencereyi kapat
ClickElement → cls=TButton name=Çıkış
// VEYA
TypeText → {Alt+F4}
// VEYA (InvokeCode)
System.Diagnostics.Process.GetProcessesByName("Polisoft")
    .FirstOrDefault()?.Kill();
```

**Dikkat:** `Kill()` veriyi kaydetmeden kapatır. Normal kapatma yolunu dene, çaresiz kalırsan Kill kullan.

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- Selector ile "var olanı bul" davranışı doğrulandı.
- WaitForIdle=true önerilir — uygulama tam yüklenmeden devam etmez.
