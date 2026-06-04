# GetElement

## Ne Yapar

Ekranda bir UI elementini bulur ve referans olarak döndürür. Tüm sonraki etkileşimler (tıklama, yazma, okuma) bu referans üzerinden gerçekleşir. **OpenRPA'nın en temel aktivitesi.**

---

## Parametreler

| Parametre | Açıklama | Varsayılan |
|---|---|---|
| `Selector` | Elementi tanımlayan kural | (zorunlu) |
| `Timeout` | ms cinsinden bekleme süresi | 5000 |
| `MinResults` | En az kaç element bulunmalı | 1 |
| `MaxResults` | En fazla kaç element | 1 |
| `Çıktı değişkeni` | Bulunan element/ler | — |

### MinResults / MaxResults — Ne Zaman Ne Kullanılır?

```
Tek element bekliyorum (alan, buton):
  MinResults=1, MaxResults=1

Varlığını kontrol etmek istiyorum (hata yok mu?):
  MinResults=0, MaxResults=1
  → Bulunamazsa null döner, exception fırlatmaz

Liste / grid (kaç satır bilinmiyor):
  MinResults=0, MaxResults=100
  → GetElements (çoğul) ile kullan
```

---

## Adım Adım Kullanım

1. Toolbox → **GetElement** sürükle
2. Selector alanına tıkla → **Spy** butonu
3. Hedef uygulamaya geç → elementi tıkla
4. Selector editörde dönen değeri incele — stabil mi?
5. **Highlight** butonuyla doğrula (element turuncu renk almalı)
6. Timeout'u uygulamaya göre ayarla:
   - Hızlı Windows uygulama: 5000
   - SAP / web portal: 15000-30000
   - Hesaplama bekleme: 30000-60000

---

## Selector Kalite Hiyerarşisi

```
En stabil              En kırılgan
      ↓                      ↓
automationid → name+cls → css/id → xpath → idx
```

**Polisoft için:**
```
cls=TEdit name=PoliceNo    ✅
cls=TEdit idx=3            ⚠️ pencere boyutu değişince kırılır
```

**Web için:**
```
id=btnSubmit               ✅
css=[data-testid=submit]   ✅
xpath=//button[3]          ⚠️
```

---

## Sık Yapılan Hatalar

### 1. Timeout çok kısa

```
❌ Timeout=500 → Ağ gecikmesinde hep başarısız
✅ Web/SAP için minimum Timeout=15000
```

### 2. MinResults=0 kullanınca null kontrolü atlamak

`MinResults=0` ile `GetElement` çalıştırılırsa element bulunamadığında aktivite `Nothing`
(null) döner, exception fırlatmaz. Ama sonraki adımda `.Length`, `.Value` gibi property'ye
doğrudan erişmek `NullReferenceException` üretir.

```
❌ MinResults=0 → element null döner → If [elements.Length = 0] → NullReferenceException!

✅ MinResults=0 → null-safe kontrol:
    If [elements Is Nothing OrElse elements.Length = 0]
    //  ^^^^^^^^^^^^^^^^^^^^
    //  Önce null mu diye sor, sonra uzunluğa bak
```

**Sık görülen örüntü — login kontrol:**
```
GetElement
  Selector: [giriş başarılı elementi]
  MinResults: 0
  → isLogin değişkenine yaz

If [isLogin Is Nothing OrElse isLogin.Length = 0]
  Then → Giriş başarısız, tekrar dene
  Else → Giriş başarılı, devam et
```

Tek element için (`OutArgument<IElement>` tipi):
```
If [loginElem Is Nothing]       ← array değil, direkt null kontrolü
```

### 3. `Timeout="{x:Null}"` — Sonsuz Bekleme Tuzağı

XAML'da `Timeout="{x:Null}"` görüyorsan bu aktivite **sonsuza kadar bekler**. Platform
null'ı "sınırsız bekle" olarak yorumlar. Element hiç gelmeyecekse robot takılır.

Özellikle `BreakableDoWhile` veya `BreakableWhile` içinde kullanıldığında döngüyü de
sonsuzlaştırır — dışarıdan müdahale edilene kadar robot kilitli kalır.

```
❌ GetElement Timeout="{x:Null}"   ← sonsuz bekleme, robot kilitlenebilir

✅ GetElement Timeout="00:00:10"   ← 10 saniye sonra ElementNotFoundException fırlatır
   veya
✅ GetElement Timeout="00:00:30"   ← yavaş sayfalar için 30 saniye
```

**Nasıl kontrol ederim?** XAML dosyasında `x:Null` ara — her biri potansiyel risk noktası.

### 4. Dinamik idx kullanımı

```
❌ selector: idx=5 → UI değişince kırılır
✅ cls=TButton name=Kaydet → stabil
```

### 4. Pencere odakta değil

```
Semptom: Highlight çalışıyor ama tıklama yanlış yere gidiyor
Çözüm:   GetElement öncesine ClickElement(ana pencere) ekle
         veya OpenApplication ile focus ver
```

### 5. Yanlış provider

```
Chrome sayfasını Windows provider ile arıyorsun
→ Selector'da provider=NativeMessaging olmalı
SAP ekranını Windows provider ile arıyorsun
→ Selector'da provider=SAP olmalı
```

---

## Sigorta Sektörü Örnekleri

### Polisoft — TC Kimlik Alanı

```
Aktivite:  GetElement
Selector:  cls=TEdit name=TCKimlik
Timeout:   5000
MinResults: 1
```

### Web Portal — Prim Sonucu

```
Aktivite:  GetElement
Selector:  css=#sonucPrim .prim-tutari
Timeout:   30000   ← hesaplama bekler
MinResults: 1
```

### Hata Dialogu Kontrolü

```
Aktivite:  GetElement
Selector:  cls=TForm name=*Hata*
Timeout:   3000
MinResults: 0   ← yoksa null, exception değil
MaxResults: 1
```

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- Timeout davranışı vanilla ile aynı görünüyor.
- MinResults=0 ile null dönüşü doğrulandı.
