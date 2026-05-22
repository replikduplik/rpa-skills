# TypeText

## Ne Yapar

Bir UI elementine klavye girdisi gönderir. Metin yazma, özel tuş kombinasyonları (Tab, Enter, Ctrl+A) ve şifre girişi için kullanılır.

---

## Parametreler

| Parametre | Açıklama | Varsayılan |
|---|---|---|
| `Element` | Hedef element (GetElement çıktısı) | (zorunlu) |
| `Text` | Yazılacak metin veya tuş kombinasyonu | (zorunlu) |
| `Delay` | Tuşlar arası ms bekleme | 0 |
| `AnimateTyping` | Yavaş yazma animasyonu | false |
| `Password` | Güvenli mod (log'a yazmaz) | false |

---

## Adım Adım Kullanım

1. **GetElement** ile hedef alanı bul
2. **TypeText** sürükle → Element parametresine GetElement çıktısını bağla
3. Text alanına yazılacak değeri gir
4. Şifre alanıysa `Password=true` işaretle

---

## Özel Tuşlar

```
{Tab}       → Bir sonraki alana geç
{Enter}     → Onay / form gönder
{Escape}    → İptal / kapat
{F5}        → Yenile
{Delete}    → Sil
{Back}      → Geri al (Backspace)
{Home}      → Satır başı
{End}       → Satır sonu
{Ctrl+A}    → Tümünü seç
{Ctrl+C}    → Kopyala
{Ctrl+V}    → Yapıştır
{Alt+F4}    → Pencereyi kapat
{Down}      → Aşağı ok (dropdown seçimi)
{Up}        → Yukarı ok
```

### Tuş Kombinasyonu Örnekleri

```
{Ctrl+A}{Delete}    → Alanı temizle (önce tümünü seç, sonra sil)
{Tab}{Tab}          → İki alan atla
34{Tab}             → "34" yaz, sonra Tab
```

---

## Sık Yapılan Hatalar

### 1. Alanı temizlemeden üstüne yazmak

```
❌ Direkt TypeText → eski metin kalır, yeni metin eklenir
✅ Önce {Ctrl+A} → sonra metni yaz:
    TypeText → Text: "{Ctrl+A}" + yeniDeger
   VEYA
    ClickElement → (alana tıkla, focus ver)
    TypeText → Text: "{Ctrl+A}12345678901"
```

### 2. Şifreyi log'a yazmak

```
❌ Log aktivitesi: "Şifre: " + sifre   → GÜVENLİK İHLALİ
✅ TypeText → Password=true
   TypeText → Text: {credential'dan gelen sifre değişkeni}
   (log'a ***** olarak görünür)
```

### 3. Yanlış element'e yazmak

```
Semptom: Metin başka alana gidiyor
Çözüm:   GetElement → ClickElement → TypeText sırası
         (önce tıkla, odakla, sonra yaz)
```

### 4. Hızlı yazma — uygulama yetişemiyor

```
Semptom: Bazı karakterler atlanıyor
Çözüm:   Delay=50-100 (ms cinsinden tuşlar arası bekleme)
         AnimateTyping=true (daha yavaş ama güvenilir)
```

### 5. Dropdown seçimi için kullanmak

```
❌ TypeText → dropdown değeri   → çoğu zaman çalışmaz
✅ ClickElement → dropdown aç
   ClickElement → seçenek tıkla
   VEYA
   ClickElement → dropdown
   TypeText → {Down}{Down}{Enter}  (ok tuşları ile seç)
```

---

## Sigorta Sektörü Örnekleri

### Polisoft — TC Kimlik Girişi

```
GetElement → tcAlan
  selector: cls=TEdit name=TCKimlik
TypeText
  Element: tcAlan
  Text: "{Ctrl+A}" + tcKimlik  ← değişkenden al
```

### Polisoft — Tarih Alanı

```
TypeText
  Element: tarihAlani
  Text: bitisTarihi  ← "dd.MM.yyyy" formatında
  Delay: 50          ← tarih alanları hassas olabilir
```

### Web Portal — Arama Kutusu

```
GetElement → aramaKutusu
  selector: css=input[type='search']
TypeText
  Element: aramaKutusu
  Text: "{Ctrl+A}" + aramaMetni + "{Enter}"
```

### Giriş Formu (Kullanıcı Adı + Şifre)

```
TypeText
  Element: kullaniciAdiAlani
  Text: kullanici        ← OpenFlow Credentials'tan
TypeText
  Element: sifreAlani
  Text: sifre            ← OpenFlow Credentials'tan
  Password: true
ClickElement → girisButonu
```

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- `{Ctrl+A}` kombinasyonu Polisoft TEdit alanlarında çalışıyor.
- `Password=true` özelliği doğrulandı — log'a `*****` yazıyor.
