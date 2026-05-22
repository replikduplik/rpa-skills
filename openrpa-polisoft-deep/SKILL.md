---
name: openrpa-polisoft-deep
description: >
  Use for any Polisoft-specific automation: login, session management, teklif
  (quote), poliçe (policy), hasar (claim), zeyilname (endorsement), raporlama,
  grid reading, modal/popup handling. Triggers on: "Polisoft", "teklif ekranı",
  "poliçe sorgula", "hasar bildirimi", "zeyilname", "DBGrid", "Polisoft formu",
  "Delphi uygulama", "sigorta sistemi", "Open sigorta".
---

# Polisoft Otomasyon Rehberi

## Teknik Altyapı

Polisoft, Delphi/VCL tabanlı bir Windows uygulamasıdır.
- Provider: **Windows** (UIAutomation)
- Pencere sınıfı: `TForm` (her modül ayrı form)
- Alan sınıfları: `TEdit`, `TComboBox`, `TButton`, `TDBGrid`, `TLabel`
- Menü: `TMainMenu`, `TMenuItem`

---

## Selector Kuralları — Polisoft

```
Öncelik: cls + name > cls + idx > name alone

Doğru:
  cls=TEdit    name=PoliceNo        → TC kimlik alanı
  cls=TButton  name=Kaydet          → Kaydet butonu
  cls=TDBGrid  idx=1                → İlk grid

Kaçın:
  idx tek başına → pencere konumuna göre değişir
  name tek başına → farklı formlarda çakışabilir
```

---

## Oturum Yönetimi

### Giriş Akışı

```
Sequence: PoliceSoftGiris
├── OpenApplication
│     FileName: "C:\Polisoft\Polisoft.exe"
│     Timeout: 15000
│
├── GetElement → loginForm
│     selector: cls=TForm name=Login*
│     Timeout: 10000
│
├── TypeText → kullanıcı adı alanı
│     selector: cls=TEdit name=KullaniciAdi
│     Text: {kullanici}    ← OpenFlow Credentials'tan al
│
├── TypeText → şifre alanı
│     selector: cls=TEdit name=Sifre
│     Text: {sifre}        ← OpenFlow Credentials'tan al
│
├── ClickElement → Giriş butonu
│     selector: cls=TButton name=Giriş
│
└── GetElement → ana menü (giriş başarılı kontrolü)
      selector: cls=TMainMenu
      Timeout: 15000
```

### Keep-Alive (Idle Timeout Önleme)

```csharp
// InvokeCode — Polisoft'un idle timeout'unu sıfırla
// Her 10 dakikada bir F5 gönder (arka planda çalışır)
var form = /* Polisoft ana formu */;
System.Windows.Forms.SendKeys.SendWait("{F5}");
System.Threading.Thread.Sleep(500);
```

**Daha iyi yaklaşım:** Ana workflow döngüsü içinde aralıklı `GetElement` çağrısı — hem pencereyi canlı tutar hem de session kontrolü yapar.

### Session Kontrolü

```
GetElement → oturum_acik_mi
  selector: cls=TMainMenu
  Timeout: 3000
  MinResults: 0   ← hata fırlatma
  MaxResults: 1

If (oturum_acik_mi == null)
  InvokeWorkflow: PoliceSoftGiris.xaml
```

---

## Teklif Modülü

### Yeni Teklif Akışı

```
1. Menüden aç: Teklif → Yeni Teklif
   ClickElement → selector: cls=TMenuItem name=*Teklif*
   ClickElement → selector: cls=TMenuItem name=*Yeni*

2. Branş seç
   ClickElement → cls=TComboBox name=Brans
   ClickElement → cls=TMenuItem name={brans}  ← Kasko/Trafik/Konut vb.

3. Müşteri bilgileri doldur
   TypeText → cls=TEdit name=TCKimlik      → tcKimlik
   TypeText → cls=TEdit name=AracPlaka     → plaka
   TypeText → cls=TEdit name=ModelYil      → modelYil
   [Tab ile geç — bazı alanlar Tab'da doldurulur]

4. Teklif hesapla
   ClickElement → cls=TButton name=*Hesapla*
   GetElement → cls=TLabel name=PrimTutari   ← sonuç etiketi
   Timeout: 30000   ← hesaplama süresi değişken

5. Teklif no ve prim oku
   teklifNo = GetElement(cls=TEdit name=TeklifNo).Value
   prim     = GetElement(cls=TLabel name=PrimTutari).Value
```

### Teklif Kaydetme

```
ClickElement → cls=TButton name=Kaydet
GetElement   → cls=TForm name=*Bilgi*     ← onay dialogu
ClickElement → cls=TButton name=Tamam
```

---

## Poliçe Modülü

### Poliçe Sorgulama

```
1. Menüden aç: Poliçe → Sorgula
2. Arama kriteri gir:
   TypeText → cls=TEdit name=PoliceNo  → policeNo
   ClickElement → cls=TButton name=Ara
3. Sonuç listesinden seç:
   GetElement → cls=TDBGrid idx=1
   [DBGrid okuma — aşağıdaki bölüme bak]
4. Çift tıkla veya Aç:
   ClickElement → cls=TButton name=*Aç*
```

### DBGrid Okuma

```csharp
// InvokeCode — Polisoft TDBGrid'den veri oku
// grid = GetElement ile alınan TDBGrid elementi

// Yöntem 1: Satır satır dolaş (klayve navigasyonu)
// Her satır için Home tuşu, ardından her sütun için Tab

// Yöntem 2: Satır seç ve değer oku (selector ile)
// Gridde belirli bir değeri içeren satırı bul

// Grid satırı navigasyonu
ClickElement(grid);
SendKeys("{Home}");  // İlk satıra git

for (int i = 0; i < satirSayisi; i++)
{
    // Her sütunu oku
    string deger = GetCellValue(grid, i, sutunIndex);
    // ... işle
    SendKeys("{Down}");  // Sonraki satır
}
```

**Pratik yaklaşım:** DBGrid yerine Polisoft'un export özelliğini kullan:
```
ClickElement → cls=TButton name=*Excel*  veya
ClickElement → cls=TButton name=*Export*
→ Excel dosyasını oku (openrpa-csharp skill → Excel COM bölümü)
```

---

## Hasar Modülü

### Yeni Hasar Bildirimi

```
Sequence: HasarBildirimGir
├── Menü: Hasar → Yeni Bildirim
│
├── Poliçe bağla
│   TypeText → cls=TEdit name=PoliceNo → policeNo
│   ClickElement → cls=TButton name=*Getir*
│   GetElement → cls=TLabel name=SigortaAdi (doğrulama)
│
├── Hasar bilgileri
│   TypeText → cls=TEdit name=HasarTarihi → hasarTarihi (dd.MM.yyyy)
│   TypeText → cls=TEdit name=HasarYeri   → hasarYeri
│   ClickElement → cls=TComboBox name=HasarTipi
│   ClickElement → cls=TMenuItem name={hasarTipi}
│   TypeText → cls=TEdit name=HasarAciklama → aciklama
│
├── İhbar kaydı oluştur
│   ClickElement → cls=TButton name=*Kaydet*
│   GetElement → cls=TEdit name=IhbarNo (sonuç)
│   ihbarNo = [okunan değer]
│
└── Doğrula
    If (ihbarNo == "" || ihbarNo == null)
        throw new Exception("İhbar kaydı oluşturulamadı")
```

---

## Zeyilname (Değişiklik) Modülü

```
Sequence: ZeyilnameOlustur
├── Poliçeyi aç (Poliçe Modülü → Sorgulama)
│
├── Değişiklik menüsü
│   ClickElement → cls=TButton name=*Zeyilname* veya
│   ClickElement → cls=TMenuItem name=*Değişiklik*
│
├── Değişiklik tipini seç
│   ClickElement → cls=TComboBox name=ZeyilnameTipi
│   ClickElement → cls=TMenuItem name={tip}
│   [Plaka değişikliği / Adres değişikliği / Ek teminat vb.]
│
├── Yeni değerleri gir
│   [Tipe göre farklı alanlar]
│
└── Onayla ve kaydet
    ClickElement → cls=TButton name=*Hesapla*    ← fark primi hesapla
    GetElement   → cls=TLabel name=FarkPrim       ← fark prim göster
    ClickElement → cls=TButton name=*Onayla*
    zeyilnameNo  = GetElement(cls=TEdit name=ZeyilnameNo).Value
```

---

## Raporlama

### Excel Export (Tercih Edilen Yöntem)

```
1. Rapor menüsüne git
   ClickElement → cls=TMenuItem name=*Rapor*
   ClickElement → cls=TMenuItem name={raporAdi}

2. Filtre parametrelerini gir
   TypeText → cls=TEdit name=BaslangicTarihi → baslangic
   TypeText → cls=TEdit name=BitisTarihi     → bitis
   ClickElement → cls=TComboBox name=Brans
   ClickElement → cls=TMenuItem name={brans}

3. Raporu oluştur
   ClickElement → cls=TButton name=*Listele*
   GetElement → cls=TDBGrid (raporun yüklendiğini bekle)
   Timeout: 30000

4. Excel'e aktar
   ClickElement → cls=TButton name=*Excel*
   [Dosya kaydet dialogunu yönet]
   GetElement → cls=TForm name=*Kaydet* → SaveDialog
   TypeText   → cls=TEdit name=FileName → hedefDosyaYolu
   ClickElement → cls=TButton name=Kaydet
```

---

## Modal ve Popup Yönetimi

### Onay Dialogları

```
// Polisoft çeşitli onay dialogları açar
// Pattern: GetElement → dialog varlığını kontrol et → yanıtla

GetElement → onay_dialog
  selector: cls=TForm name=*Uyarı*  veya  cls=TForm name=*Onay*
  Timeout: 5000
  MinResults: 0  ← zorunlu değil

If (onay_dialog != null)
    ClickElement → cls=TButton name=Evet  (veya Tamam / Hayır)
```

### Hata Dialogları

```
GetElement → hata_dialog
  selector: cls=TForm name=*Hata*
  Timeout: 3000
  MinResults: 0

If (hata_dialog != null)
{
    hataMesaji = GetElement(cls=TLabel).InnerText
    ClickElement → cls=TButton name=Tamam
    throw new Exception("Polisoft hatası: " + hataMesaji)
}
```

---

## Sık Sorunlar ve Çözümler

| Sorun | Sebep | Çözüm |
|---|---|---|
| Giriş başarısız | Yanlış credential veya kilitli | OpenFlow Credential kontrol, hesap kilidi kaldır |
| Alan boş kalıyor | Tab gerekiyor, Enter değil | TypeText sonrası `{Tab}` veya `{Enter}` gönder |
| DBGrid okunmuyor | Grid provider desteği | Export → Excel yöntemini kullan |
| Modal engel | Önceki işlem tamamlanmadı | Her adımda modal kontrolü ekle |
| Session kapandı | Idle timeout | Keep-alive veya döngü başında session kontrolü |
| Tarih formatı hatası | Polisoft dd.MM.yyyy bekliyor | Tüm tarihleri `dd.MM.yyyy` olarak gönder |
| Yavaş hesaplama | Sunucu yoğunluğu | GetElement Timeout=30000-60000 kullan |

---

## Fork Notları

> Bu bölüm keşfedilen farkları biriktirir.
> İlk fark bulununca `fork-differences.md`'ye de ekle.

- [ ] Polisoft versiyonu: ___
- [ ] Menü yapısı vanilla Polisoft'tan farklı mı?
- [ ] Selector'lar beklenenden farklı çalışıyor mu?
