# InvokeWorkflow

## Ne Yapar

Başka bir `.xaml` workflow dosyasını çalıştırır ve parametreler aracılığıyla veri alışverişi yapar. Büyük workflow'ları modüler yapıya bölmek ve ortak adımları paylaşmak için kullanılır.

---

## Parametreler

| Parametre | Açıklama |
|---|---|
| `WorkflowFileName` | Çalıştırılacak .xaml dosyasının yolu |
| `Arguments` | In / Out / InOut yönlü değişken eşleştirmeleri |

---

## Parametre Yönleri

```
In    → Ana workflow → Alt workflow'a değer gönder (salt okunur)
Out   → Alt workflow → Ana workflow'a değer döndür
InOut → Hem gönder hem geri al (iki yönlü)
```

**Kural:** Alt workflow'da **değiştirilecek** değişken `Out` veya `InOut` olmalı.

---

## Adım Adım Kullanım

1. Toolbox → **InvokeWorkflow** sürükle
2. `WorkflowFileName` → `...` butonu → `.xaml` dosyasını seç
3. `Arguments` → `+` ile parametre ekle:
   - Değişken adını yaz (alt workflow'daki adla eşleşmeli)
   - Yönü seç: In / Out / InOut
   - Değeri ana workflow değişkenine bağla
4. Alt workflow'u aç → `Arguments` bölümünde aynı parametre adlarının tanımlı olduğunu doğrula

---

## Sık Yapılan Hatalar

### 1. Yön yanlış — değer geri gelmiyor

```
❌ InvokeWorkflow Arguments: teklifNo [In]
   → Alt workflow teklifNo değerini ayarlıyor ama ana workflow'a dönmüyor

✅ InvokeWorkflow Arguments: teklifNo [Out]
   → Alt workflow'dan ana workflow'a değer gelir
```

### 2. Parametre adı uyuşmuyor

```
❌ Ana:  policeNo [In]
   Alt:  policeno (küçük harf) → bağlantı kurulamaz

✅ İki tarafta da tamamen aynı isim: policeNo
```

### 3. Alt workflow'da parametre tanımlı değil

```
Semptom: "Argument 'plaka' not found in workflow"
Çözüm:   Alt workflow'u aç → Variables/Arguments panelinde
          plaka adında, doğru yönde değişken oluştur
```

### 4. Büyük DataTable geçirmek

```
❌ 10.000 satırlık DataTable [InOut] → bellek ve performans sorunu

✅ WorkItem payload kullan — her robot kendi kaydını işler
   VEYA dosya yolunu geç, DataTable'ı alt workflow okusun
```

### 5. Alt workflow'un kendi TryCatch'i yok

```
❌ Alt workflow hata → ana workflow TryCatch yakalar, ama nereden hata geldiği belli değil

✅ Her alt workflow kendi TryCatch'ine sahip
   → Hatayı rethrow ederek ana workflow'a iletir
   → Log'da "GirisYap.xaml başarısız" nerede olduğu belli olur
```

---

## Sigorta Sektörü Örnekleri

### Modüler Teklif Akışı

**Ana workflow: TeklifOtomasyonu.xaml**

```
InvokeWorkflow: GirisYap.xaml
  Arguments:
    sistem   [In]  ← "Polisoft"
    oturum   [Out] → oturumDegiskeni

InvokeWorkflow: VeriDogrula.xaml
  Arguments:
    tcKimlik [In]  ← tcKimlikDegiskeni
    plaka    [In]  ← plakaDegiskeni
    hatalar  [Out] → hatalarDegiskeni

If (hatalar != "")
    InvokeWorkflow: HataRaporla.xaml
      Arguments:
        hataMetni [In] ← hatalar

InvokeWorkflow: TeklifAl.xaml
  Arguments:
    tcKimlik  [In]  ← tcKimlikDegiskeni
    plaka     [In]  ← plakaDegiskeni
    oturum    [In]  ← oturumDegiskeni
    teklifNo  [Out] → teklifNoDegiskeni
    prim      [Out] → primDegiskeni
```

### Alt Workflow Parametre Tanımı (GirisYap.xaml)

```
Variables/Arguments paneli:
  sistem   Direction=In   Type=String
  oturum   Direction=Out  Type=String

Sequence içi:
  [giriş adımları]
  Assign: oturum = "oturum-id-xyz"
```

### Ortak Alt Workflow: MailGonder.xaml

```
Arguments:
  alici   [In]  Type=String
  konu    [In]  Type=String
  icerik  [In]  Type=String
  basarili [Out] Type=Boolean

// Ana workflow'dan çağır:
InvokeWorkflow: MailGonder.xaml
  Arguments:
    alici   [In] ← "mudur@sirket.com"
    konu    [In] ← "Günlük Rapor"
    icerik  [In] ← raporMetni
    basarili [Out] → mailGonderildi
```

---

## Ne Zaman InvokeWorkflow Kullanılır?

| Durum | Karar |
|---|---|
| Aynı adımlar 2+ workflow'da tekrar ediyor | ✅ Ortak alt workflow |
| Tek workflow 200+ satırı geçiyor | ✅ Parçalara böl |
| Farklı uygulama açma/kapama | ✅ GirisYap / CikisYap alt workflow |
| Hata bildirimi her yerde aynı | ✅ HataRaporla alt workflow |
| Tek kullanımlık basit adım | ❌ Doğrudan ana workflow'a yaz |

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- Parametre bağlama arayüzü vanilla ile aynı görünüyor.
- `.xaml` dosya seçimi browser ile yapılıyor — relative path destekleniyor.
