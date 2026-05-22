# InvokeCode

## Ne Yapar

Workflow içinde C# kodu çalıştırır. Yerleşik aktivitelerle yapılamayan işlemler, veri dönüşümleri, hesaplamalar, harici API çağrıları ve Excel COM işlemleri için kullanılır.

---

## Parametreler

| Parametre | Açıklama |
|---|---|
| `Code` | Çalıştırılacak C# kodu |
| `Arguments` | Değişken bağlamaları (In / Out / InOut) |
| `ImportNamespaces` | Ek using direktifleri |

---

## Değişken Yönleri

```
In    → Workflow → InvokeCode (okuma)
Out   → InvokeCode → Workflow (yazma)
InOut → Her iki yön (hem oku hem yaz)
```

**Önemli:** Bir değişkene **yazmak** istiyorsan `Out` veya `InOut` olmalı.

### Bağlama Örneği

```
Arguments:
  policeNo     [In]  ← workflow değişkeninden oku
  teklifNo     [Out] → workflow değişkenine yaz
  musteriData  [InOut] ← oku ve güncelle → geri yaz
```

---

## Kod Yazma Kuralları

```csharp
// 1. using direktifi YOK — tam namespace kullan
System.DateTime.Now          // ✅
DateTime.Now                 // ❌ (using System; olmadan çalışmaz)

// 2. async/await desteklenir
var result = await httpClient.GetAsync(url);

// 3. Yerel fonksiyon tanımlayabilirsin
string Temizle(string s) => s?.Trim() ?? "";
string temiz = Temizle(hamVeri);

// 4. throw ile hata fırlat → TryCatch yakalar
if (string.IsNullOrEmpty(policeNo))
    throw new System.Exception("PoliceNo boş");

// 5. Console.WriteLine → OpenRPA Output panelinde görünür
Console.WriteLine($"DEBUG: policeNo={policeNo}");
```

---

## Adım Adım Kullanım

1. Toolbox → **InvokeCode** sürükle
2. Arguments → değişkenleri ekle ve yönlerini seç
3. Code editörüne C# kodunu yaz
4. Çalıştır ve Output panelini izle
5. Hata varsa Catch bloğu yakalayacak (TryCatch içindeyse)

---

## Sık Yapılan Hatalar

### 1. Out değişkeni set etmemek

```csharp
// ❌ teklifNo Out olarak bağlandı ama hiç set edilmedi
// → Workflow'da teklifNo null kalır

// ✅ Her durumda set et
teklifNo = hesaplananDeger ?? "";
```

### 2. using yetersizliği

```csharp
// ❌ Compile hatası
DateTime.Now
List<string>

// ✅ Tam namespace
System.DateTime.Now
System.Collections.Generic.List<string>

// VEYA ImportNamespaces'e ekle:
// System
// System.Collections.Generic
```

### 3. Null reference

```csharp
// ❌ Crash
string deger = item.payload["Alan"].ToString();

// ✅ Null-safe
string deger = item.payload["Alan"]?.ToString() ?? "";
```

### 4. COM nesnesi serbest bırakmamak

```csharp
// ❌ Excel process'i takılı kalır
xlApp.Quit();

// ✅ Her zaman Marshal.ReleaseComObject + GC.Collect
System.Runtime.InteropServices.Marshal.ReleaseComObject(wb);
System.Runtime.InteropServices.Marshal.ReleaseComObject(xlApp);
GC.Collect();
GC.WaitForPendingFinalizers();
```

### 5. Yanlış tarih formatı

```csharp
// ❌ Locale'e göre değişir
System.DateTime.Parse("31.01.2025");

// ✅ Explicit format
System.DateTime.ParseExact("31.01.2025", "dd.MM.yyyy",
    System.Globalization.CultureInfo.InvariantCulture);
```

---

## Sigorta Sektörü Örnekleri

### WorkItem Payload Okuma

```csharp
// Güvenli payload okuma — tüm alanlar null-safe
string policeNo    = item.payload["PoliceNo"]?.ToString() ?? "";
string plaka       = item.payload["Plaka"]?.ToString() ?? "";
string brans       = item.payload["Brans"]?.ToString() ?? "Trafik";
string bitisDeger  = item.payload["BitisTarihi"]?.ToString() ?? "";

System.DateTime bitis = System.DateTime.MinValue;
if (!string.IsNullOrEmpty(bitisDeger))
    System.DateTime.TryParseExact(bitisDeger, "dd.MM.yyyy",
        System.Globalization.CultureInfo.InvariantCulture,
        System.Globalization.DateTimeStyles.None, out bitis);

// Doğrulama
if (string.IsNullOrEmpty(policeNo))
    throw new System.Exception("PoliceNo boş — WorkItem geçersiz");
```

### Excel'den DataTable Yükleme

```csharp
// → openrpa-csharp skill → Bölüm 2: Excel COM Interop
```

### TC Kimlik Doğrulama

```csharp
// TC 11 hane, ilk rakam 0 olamaz, algoritma kontrolü
bool TcGecerliMi(string tc)
{
    if (tc == null || tc.Length != 11 || tc[0] == '0') return false;
    if (!System.Text.RegularExpressions.Regex.IsMatch(tc, @"^\d{11}$")) return false;
    int[] d = tc.Select(c => int.Parse(c.ToString())).ToArray();
    int t1 = (d[0]+d[2]+d[4]+d[6]+d[8]) * 7 - (d[1]+d[3]+d[5]+d[7]);
    if (((t1 % 10) + 10) % 10 != d[9]) return false;
    return d.Take(10).Sum() % 10 == d[10];
}

tcGecerli = TcGecerliMi(tcKimlik);
```

---

## Ne Zaman InvokeCode Kullanma

| Durum | Öneri |
|---|---|
| Basit string birleştirme | Assign aktivitesi yeterli |
| Değer atama | Assign |
| Koşul kontrolü | If aktivitesi |
| Element tıklama / yazma | ClickElement / TypeText |
| Workflow çağırma | InvokeWorkflow |

InvokeCode için: karmaşık veri işleme, API çağrısı, Excel, regex, tip dönüşümü.

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- async/await desteği doğrulandı.
- `Console.WriteLine` Output panelinde görünüyor.
