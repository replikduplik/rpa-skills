# Delay

## Ne Yapar

Workflow'u belirtilen süre kadar durdurur. Sabit bekleme süresi ekler.

---

## Parametreler

| Parametre | Açıklama | Varsayılan |
|---|---|---|
| `Milliseconds` | Bekleme süresi (ms) | 1000 |

---

## Ne Zaman Kullanılır, Ne Zaman Kullanılmaz

### ✅ Kullan

```
1. Kullanıcı etkileşimi gerektiren durum:
   ClickElement → Delay(500) → TypeText
   (bazı uygulamalar click sonrası focus almak için zaman ister)

2. API rate limiting:
   foreach kayıt → HTTP istek → Delay(1000) → sonraki istek

3. Animasyon/transition beklemesi:
   ClickElement (tab değiştir) → Delay(300) → GetElement

4. Debug sırasında yavaşlatma:
   (Production'a almadan önce kaldır)
```

### ❌ Kullanma

```
1. "Uygulama yüklenene kadar bekle" için:
   ❌ Delay(3000)
   ✅ GetElement → Timeout=15000  (element hazır olduğunda devam eder)

2. "Hesaplama bitene kadar bekle" için:
   ❌ Delay(5000)
   ✅ GetElement → sonuç elementi → Timeout=30000

3. "Sayfa yüklenene kadar bekle" için:
   ❌ Delay(2000)
   ✅ GetElement → css=.page-loaded veya spinner kaybolana kadar bekle

4. Loop içinde sabit bekleme:
   ❌ While True → [işlem] → Delay(1000)
   ✅ GetElement ile koşul kontrolü
```

---

## Neden Delay Kötü Bir Alışkanlık?

| Sorun | Açıklama |
|---|---|
| **Yavaş** | 3 saniyelik Delay, element 500ms'de hazır olsa bile 3 saniye bekler |
| **Kırılgan** | Sistem yavaşlayınca Delay yetersiz kalır → hata |
| **Tahmin** | "Kaç ms beklemeliyim?" sorusu gün geçtikçe cevabı değişir |
| **Bakım maliyeti** | Her ortamda farklı Delay değeri gerekebilir |

**Kural:** Delay gördüğünde soğukkanlılıkla sor: *"Burada ne bekliyorum? GetElement ile bekleyebilir miyim?"*

---

## Delay Yerine GetElement ile Bekleme

```
// Hesaplama sonucunu bekle
GetElement → sonucAlani
  selector: cls=TLabel name=PrimTutari
  Timeout:  30000   ← hesaplama ne kadar sürerse sürsün bekler

// Spinner kaybolmasını bekle
GetElement → yuklemeBitti
  selector: css=.spinner[style*='display: none']
  Timeout:  20000

// Modal kapanmasını bekle
GetElement → modalYok
  selector: cls=TForm name=*Onay*
  Timeout:  10000
  MinResults: 0  ← yoksa null döner
// Sonra: null olana kadar döngü
```

---

## Kabul Edilebilir Delay Değerleri

| Durum | Değer | Gerekçe |
|---|---|---|
| Click sonrası focus | 200-500 ms | UI tepki süresi |
| API rate limit | 1000-2000 ms | Sunucu koruması |
| Animasyon geçişi | 300-500 ms | Görsel geçiş |
| Tab/panel değişimi | 300-800 ms | DOM yeniden yükleme |

**500ms'den büyük sabit Delay → GetElement ile değiştirilmeli.**

---

## Sigorta Sektörü Örnekleri

### Kabul Edilebilir Kullanım

```
// Polisoft — alan değeri hesaplandıktan sonra Tab
TypeText → plaka
Delay: 300  ← Polisoft alanı hesaplarken kısa bekleme
TypeText → {Tab}
```

```
// API rate limit — toplu teklif alma
ForEach arac in aracListesi
    HTTP → TeklifAl(arac)
    Delay: 1500  ← API kotası aşmamak için
```

### Kötü Kullanım → Düzeltme

```
❌ Kötü:
OpenApplication → Polisoft
Delay: 5000  ← "açılması için bekle"
GetElement → giriş ekranı

✅ Düzeltilmiş:
OpenApplication → Polisoft
GetElement → giriş ekranı
  Timeout: 20000  ← açılana kadar bekler
```

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- Delay davranışı vanilla ile aynı.
- Minimum Delay değeri test edilmedi (teorik olarak 0ms kabul eder).
