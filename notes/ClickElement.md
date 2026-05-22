# ClickElement

## Ne Yapar

Bir UI elementini tıklar. Buton, menü öğesi, checkbox, link ve diğer tıklanabilir elementler için kullanılır.

---

## Parametreler

| Parametre | Açıklama | Varsayılan |
|---|---|---|
| `Element` | Hedef element | (zorunlu) |
| `Button` | Left / Right / Middle | Left |
| `DoubleClick` | Çift tıklama | false |
| `VirtualClick` | Gerçek fare yerine sanal tıklama | false |
| `OffsetX / OffsetY` | Element merkezinden piksel kayması | 0 |

---

## VirtualClick vs Gerçek Tıklama

| | VirtualClick=false (gerçek) | VirtualClick=true (sanal) |
|---|---|---|
| Uygulama odakta olmalı | Evet | Hayır (arka planda çalışır) |
| Fare konumu değişir | Evet | Hayır |
| Paralel kullanım | Hayır | Evet |
| Bazı uygulamalarla uyumluluk | Yüksek | Düşük (Delphi/VCL'de sorun çıkabilir) |

**Kural:** Polisoft (Delphi) için VirtualClick=false. Web portallar için test et — çoğu VirtualClick destekler.

---

## Adım Adım Kullanım

1. **GetElement** ile elementi bul
2. **ClickElement** sürükle
3. Element parametresine GetElement çıktısını bağla
4. Gerekirse Button ve DoubleClick ayarla

**Kısa yol:** GetElement'in çıktı değişkenine gerek yoksa, ClickElement aktivitesine Selector doğrudan yazılabilir (dahili GetElement).

---

## Sık Yapılan Hatalar

### 1. Element odakta değil

```
Semptom: Tıklama çalışmıyor veya yanlış yere gidiyor
Çözüm:
    ClickElement → ana pencere (focus ver)
    ClickElement → hedef element
```

### 2. Element görünür ama tıklanamaz (disabled)

```
Semptom: ClickElement çalıştı ama hiçbir şey olmadı
Kontrol:
    GetElement → element.IsEnabled == false?
    → Önceki adım tamamlandı mı? (hesaplama, yükleme)
    → Bekleme ekle veya enabled olana kadar döngü
```

### 3. Dropdown menü kapanıyor

```
Semptom: ClickElement → dropdown açıldı → menü kapandı → seçenek tıklanamıyor
Çözüm:   GetElement → menü öğesi → ClickElement
         (önce elementi bul, sonra tıkla — ayrı iki aktivite)
```

### 4. Koordinat bazlı tıklama

```
Semptom: DBGrid'deki belirli bir hücreyi tıklamak istiyorum
Çözüm:   OffsetX ve OffsetY ile merkez koordinattan kaydır
         OffsetX = (sütunIndex * sütunGenişliği) - (gridGenişliği / 2)
```

### 5. Sağ tıklama menüsü

```
ClickElement
  Element: hedefEleman
  Button: Right
→ Bağlam menüsü açılır
GetElement → menü seçeneği
ClickElement → menü seçeneği
```

---

## Sigorta Sektörü Örnekleri

### Polisoft — Menü Navigasyonu

```
ClickElement
  selector: cls=TMenuItem name=*Teklif*
  VirtualClick: false

// Menü açıldı, alt menüyü bekle
GetElement → cls=TMenuItem name=*Yeni*
ClickElement
  selector: cls=TMenuItem name=*Yeni*
```

### Web Portal — Buton

```
GetElement → submitButon
  selector: css=button[type='submit']:not([disabled])
  Timeout: 10000

ClickElement
  Element: submitButon
```

### Checkbox İşaretleme

```
GetElement → onayKutusu
  selector: cls=TCheckBox name=OnayKutusu

// Durumu kontrol et
If (onayKutusu.IsChecked == false)
    ClickElement → onayKutusu
```

### Tablo Satırı Seçme (DBGrid)

```
// İlk satırı tıkla
ClickElement
  selector: cls=TDBGrid idx=1
  OffsetX: 0
  OffsetY: -80  ← ilk veri satırına kaydır (başlıktan sonra)

// Çift tıkla ile aç
ClickElement
  selector: cls=TDBGrid idx=1
  DoubleClick: true
  OffsetY: -80
```

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- VirtualClick davranışı test edilmeli — Polisoft için default (false) önerilir.
