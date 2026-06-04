# ForEach ve BreakableLoop (Döngüler)

## Ne Yapar

Koleksiyon veya liste üzerinde tekrarlayan işlemler yapar. OpenRPA'da birden fazla döngü aktivitesi vardır — doğru seçim performansı ve doğruluğu doğrudan etkiler.

---

## Döngü Tipi Seçim Tablosu

| Aktivite | Ne Zaman |
|---|---|
| `ForEach` | Bilinen liste/koleksiyon üzerinde dolaş (DataTable, JArray, List) |
| `While` | Koşul sağlandığı sürece dön (WorkItem kuyruğu, sayfa sayfalama) |
| `BreakableLoop` | Break ile çıkılabilir For/While döngüsü |
| `DoWhile` | En az bir kez çalışması garantili döngü |

---

## ForEach

### Kullanım

```
ForEach (satir in policeTablosu.Rows)
├── Assign: policeNo = satir["PoliceNo"].ToString()
├── Assign: plaka    = satir["Plaka"].ToString()
└── [İşlemler]
```

### DataTable Üzerinde

```
ForEach Aktivitesi:
  TypeArgument: System.Data.DataRow
  Values:       policeTablosu.Rows  VEYA  policeTablosu.AsEnumerable()
  DisplayName:  "Her poliçe için"
  Item:         satir  (döngü değişkeni adı)
```

### JArray Üzerinde

```
ForEach:
  TypeArgument: Newtonsoft.Json.Linq.JToken
  Values:       teklifListesi  ← JArray
  Item:         teklif

  // İçinde:
  Assign: teklifNo = teklif["no"]?.ToString() ?? ""
```

### List<string> Üzerinde

```
ForEach:
  TypeArgument: String
  Values:       plakaListesi
  Item:         plaka
```

---

## BreakableLoop ile Break

```
BreakableLoop
└── While (True)
    ├── PopWorkitem → item
    │
    ├── If (item == null)
    │       Break  ← Kuyruk boş, döngüden çık
    │
    ├── TryCatch
    │   ├── Try: [WorkItem işle]
    │   └── Catch: UpdateWorkitem → failed
    │
    └── [Devam et]
```

**Break aktivitesi** — BreakableLoop veya ForEach içinde döngüyü anında sonlandırır.

**Continue aktivitesi** — Mevcut iterasyonu atla, bir sonrakine geç.

---

## While — Koşullu Döngü

```
While (kalanGun > 0 AND islenmedi)
├── [İşlem]
├── kalanGun = kalanGun - 1
└── [Koşul güncellemesi]
```

**Sonsuz döngü riski:** Her While döngüsünde bir güvenlik sayacı ekle:
```
Assign: sayac = 0
While (kosul)
├── sayac = sayac + 1
├── If (sayac > 1000)
│       Throw → "Sonsuz döngü tespit edildi"
└── [İşlem]
```

---

## Sık Yapılan Hatalar

### 1. ForEach içinde koleksiyonu değiştirmek

```
❌ ForEach satir in tablo.Rows
     tablo.Rows.Remove(satir)  ← koleksiyonu değiştiriyorsun → hata

✅ Silinecekleri ayrı listeye topla, döngü dışında sil
```

### 2. Büyük veri kümesinde ForEach

```
❌ ForEach satir in 100000RowTable.Rows
   → Her satır için ayrı aktivite çalışır → bellek şişer

✅ DataTable'ı toplu işle (InvokeCode → LINQ/SQL benzeri)
   VEYA sayfalandır (1000'er satır)
```

### 3. Break olmayan While(True)

```
❌ While (True)
     [WorkItem al, işle]
   → Kuyruk boşaldığında çıkış yolu yok

✅ While (True)
     PopWorkitem → item
     If (item == null) → Break
```

### 6. BreakableDoWhile'da `Or` ile Yazılan Yanlış Çıkış Koşulu

Bu özellikle görüntü tabanlı bekleme döngülerinde çok sık yapılan hata. `Or` kullanınca
koşulun bir parçası her zaman `True` kalabilir ve döngü hiç bitmez.

```
❌ YANLIŞ — Or operatörü sonsuz döngüye açık kapı bırakır
BreakableDoWhile
  Condition: [elem.Length = 0 Or sayac < 60]
  //          ^^^^^^^^^^^^^^^^^
  // GetElement görüntüyü hiç bulamazsa elem null kalır
  // Nothing.Length = 0 → True, sayac büyüse bile döngü bitmez

❌ Ayrıca: elem null iken elem.Length çağrısı → NullReferenceException!
```

**Doğru yaklaşım — null kontrol + And ile:**
```
✅ DOĞRU
BreakableDoWhile
  Condition: [(elem Is Nothing OrElse elem.Length = 0) And sayac < 60]
  //          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^
  //          Null-safe kontrol                        VE sayaç sınırı

  Gövde:
    GetElement → elem (Timeout="00:00:10")  ← sonsuz Timeout KULLANMA
    Assign: sayac = sayac + 1
    If [sayac >= 60] → Break  ← güvenlik çıkışı
```

**Özet kural:** BreakableDoWhile koşulunda döngüyü sürdüren parçalar `And` ile bağlanmalı,
`Or` ile değil. Şöyle düşün: koşul `True` olduğu sürece döngü devam eder — `Or`'un
herhangi bir tarafı `True` kalırsa çıkış olmaz.

### 4. ForEach içinde yanlış TypeArgument

```
❌ TypeArgument: Object
   → .PoliceNo gibi doğrudan erişim çalışmaz

✅ TypeArgument: System.Data.DataRow (DataTable için)
   TypeArgument: String (List<string> için)
```

### 5. Döngü içinde gereksiz Delay

```
❌ ForEach → [işlem] → Delay(1000)
   1000 kayıt × 1 saniye = 16 dakika ekstra bekleme

✅ Sadece gerektiğinde Delay ekle (API limit vb.)
```

---

## Sigorta Sektörü Örnekleri

### WorkItem Kuyruğu İşleme

```
GetWorkItemQueue → wiq
  WorkItemQueueName: "policeye-yenileme-kuyruğu"

BreakableLoop
└── While (True)
    ├── PopWorkitem → item
    │     wiq: wiq
    │
    ├── If (item == null) → Break
    │
    ├── Log → "İşleniyor: " + item.name
    │
    └── TryCatch
        ├── Try
        │   ├── [Polisoft işlemleri]
        │   └── UpdateWorkitem → successful
        └── Catch (Exception e)
            └── UpdateWorkitem → failed, error: e.Message
```

### Excel Listesini İşleme

```
// Excel'den policeTablosu geldi (DataTable)
Assign: basarili = 0
Assign: basarisiz = 0

ForEach (satir in policeTablosu.AsEnumerable())
├── TryCatch
│   ├── Try
│   │   ├── Assign: policeNo = satir["PoliceNo"].ToString()
│   │   ├── [İşlemler]
│   │   └── Assign: basarili = basarili + 1
│   └── Catch (Exception e)
│         ├── Log → "HATA " + policeNo + ": " + e.Message
│         └── Assign: basarisiz = basarisiz + 1
│
└── [Döngü biter]

Log → $"Tamamlandı: {basarili} başarılı, {basarisiz} başarısız"
```

### Sayfa Sayfalama (Web Portal)

```
Assign: sayfaNo = 1
Assign: devamEt = True

While (devamEt)
├── GetElement → kayitlar (mevcut sayfa)
│
├── ForEach kayit in kayitlar
│     [kayıt işle]
│
├── GetElement → sonrakiButon
│     MinResults: 0  ← son sayfada yok
│
├── If (sonrakiButon == null)
│       Assign: devamEt = False
│   Else
│       ClickElement → sonrakiButon
│       Assign: sayfaNo = sayfaNo + 1
│       GetElement → yeniSayfa (yüklenmesini bekle)
│
└── [Güvenlik: sayfaNo > 100 → break]
```

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- BreakableLoop ve Break aktiviteleri mevcut — doğrulandı.
- ForEach TypeArgument seçimi önemli — Object seçmekten kaçın.
