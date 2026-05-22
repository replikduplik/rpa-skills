---
name: openrpa-csharp
description: >
  Use when writing C# code inside OpenRPA InvokeCode activity. Triggers on:
  "InvokeCode", "C# kodu", "nasıl yazarım C#", "string dönüşüm", "tarih format",
  "Excel COM", "DataTable", "LINQ", "dosya okuma", "HTTP istek", "JSON parse",
  "REST API", "regex", "tip dönüşümü", "TC kimlik", "prim hesabı".
---

# OpenRPA — InvokeCode C# Rehberi

## InvokeCode Temel Kurallar

```
1. async/await desteklenir — await kullanabilirsin
2. Değişken yönleri:
     In    → workflow'dan koda gelen
     Out   → koddan workflow'a giden
     InOut → her iki yön
3. using direktifi yok — tam namespace kullan:
     System.DateTime.Now  (DateTime.Now değil)
     System.Text.StringBuilder
4. Hata → workflow TryCatch'e fırlatır (throw kullan)
5. Console.WriteLine → OpenRPA Output panelinde görünür (debug için)
```

---

## 1. String ve Tarih İşlemleri

### String Temizleme

```csharp
// Boşluk ve gereksiz karakter temizleme
string temiz = ham.Trim().Replace("\r\n", " ").Replace("  ", " ");

// Sayısal string → sadece rakamlar
string sadeceSayi = System.Text.RegularExpressions.Regex.Replace(ham, @"[^\d]", "");

// Büyük/küçük harf normalize
string normalize = System.Globalization.CultureInfo
    .GetCultureInfo("tr-TR").TextInfo.ToTitleCase(ham.ToLower());
```

### Tarih Dönüşümleri

```csharp
// String → DateTime (Türkçe format)
System.DateTime tarih = System.DateTime.ParseExact(
    "31.01.2025", "dd.MM.yyyy",
    System.Globalization.CultureInfo.InvariantCulture);

// DateTime → string
string goruntule = tarih.ToString("dd.MM.yyyy");
string iso       = tarih.ToString("yyyy-MM-dd");

// Gün farkı
int kalanGun = (bitisTarihi - System.DateTime.Today).Days;

// Ay sonu bulma
System.DateTime aySonu = new System.DateTime(tarih.Year, tarih.Month,
    System.DateTime.DaysInMonth(tarih.Year, tarih.Month));

// Null-safe parse
System.DateTime? güvenliTarih = null;
System.DateTime temp;
if (System.DateTime.TryParseExact(hamDeger, "dd.MM.yyyy",
    System.Globalization.CultureInfo.InvariantCulture,
    System.Globalization.DateTimeStyles.None, out temp))
    güvenliTarih = temp;
```

### Regex — Sigorta Alanları

```csharp
// TC Kimlik doğrulama (basit format)
bool tcGecerli = System.Text.RegularExpressions.Regex.IsMatch(tcKimlik, @"^\d{11}$")
    && tcKimlik[0] != '0';

// Plaka format (34ABC123 veya 34 ABC 123)
string plakaTemiz = System.Text.RegularExpressions.Regex.Replace(plaka, @"\s", "").ToUpper();
bool plakaGecerli = System.Text.RegularExpressions.Regex.IsMatch(plakaTemiz, @"^\d{2}[A-Z]{1,3}\d{2,4}$");

// Para birimi string → decimal
string primStr = "4.250,00 TL";
decimal prim = decimal.Parse(
    System.Text.RegularExpressions.Regex.Replace(primStr, @"[^\d,]", "").Replace(",", "."),
    System.Globalization.CultureInfo.InvariantCulture);
```

---

## 2. Excel COM Interop

### Hücre Okuma

```csharp
// Excel'i aç ve oku
Microsoft.Office.Interop.Excel.Application xlApp = null;
Microsoft.Office.Interop.Excel.Workbook wb = null;
Microsoft.Office.Interop.Excel.Worksheet ws = null;
var tablo = new System.Data.DataTable();

try
{
    xlApp = new Microsoft.Office.Interop.Excel.Application();
    xlApp.Visible = false;
    xlApp.DisplayAlerts = false;
    wb = xlApp.Workbooks.Open(dosyaYolu);
    ws = (Microsoft.Office.Interop.Excel.Worksheet)wb.Sheets[1];

    int sonSatir = ws.Cells[ws.Rows.Count, 1].End(
        Microsoft.Office.Interop.Excel.XlDirection.xlUp).Row;

    // Başlık satırından sütunları al
    for (int j = 1; j <= ws.UsedRange.Columns.Count; j++)
        tablo.Columns.Add(ws.Cells[1, j].Value?.ToString() ?? $"Col{j}");

    // Veri satırları (2. satırdan başla)
    for (int i = 2; i <= sonSatir; i++)
    {
        var row = tablo.NewRow();
        for (int j = 1; j <= tablo.Columns.Count; j++)
            row[j - 1] = ws.Cells[i, j].Value?.ToString() ?? "";
        tablo.Rows.Add(row);
    }

    excelDataTable = tablo;  // Out değişkeni
}
finally
{
    wb?.Close(false);
    xlApp?.Quit();
    if (ws   != null) System.Runtime.InteropServices.Marshal.ReleaseComObject(ws);
    if (wb   != null) System.Runtime.InteropServices.Marshal.ReleaseComObject(wb);
    if (xlApp!= null) System.Runtime.InteropServices.Marshal.ReleaseComObject(xlApp);
    GC.Collect();
    GC.WaitForPendingFinalizers();
}
```

### Excel'e Yazma

```csharp
// Yeni satır ekle veya mevcut güncelle
ws.Cells[sonSatir + 1, 1].Value = policeNo;
ws.Cells[sonSatir + 1, 2].Value = teklifNo;
ws.Cells[sonSatir + 1, 3].Value = System.DateTime.Now.ToString("dd.MM.yyyy HH:mm");
ws.Cells[sonSatir + 1, 4].Value = prim;
ws.Cells[sonSatir + 1, 4].NumberFormat = "#,##0.00";  // Para format

wb.Save();
```

---

## 3. DataTable Filtreleme ve LINQ

### Temel Filtre

```csharp
// DataTable → List ile LINQ
var aktifPoliceler = policeTablosu.AsEnumerable()
    .Where(r => r["Durum"].ToString() == "Aktif"
             && System.DateTime.Parse(r["BitisTarihi"].ToString()) <= System.DateTime.Today.AddDays(30))
    .Select(r => new {
        PoliceNo    = r["PoliceNo"].ToString(),
        Plaka       = r["Plaka"].ToString(),
        BitisTarihi = r["BitisTarihi"].ToString()
    })
    .ToList();

sonucSayisi = aktifPoliceler.Count;  // Out değişkeni
```

### DataTable Kopyalama ve Filtreleme

```csharp
// Yeni DataTable — sadece belirli sütunlar
System.Data.DataTable filtreli = policeTablosu.Clone();
// Clone sadece yapıyı kopyalar, veriyi değil

foreach (System.Data.DataRow row in policeTablosu.Rows)
{
    if (row["Brans"].ToString() == "Kasko")
        filtreli.ImportRow(row);
}

kaskoTablosu = filtreli;  // Out değişkeni
```

### Gruplama ve Toplama

```csharp
// Brans bazında prim toplamı
var bransPrim = policeTablosu.AsEnumerable()
    .GroupBy(r => r["Brans"].ToString())
    .Select(g => new {
        Brans = g.Key,
        ToplamPrim = g.Sum(r => decimal.Parse(r["Prim"].ToString())),
        AdetPolic  = g.Count()
    })
    .OrderByDescending(x => x.ToplamPrim)
    .ToList();

foreach (var b in bransPrim)
    Console.WriteLine($"{b.Brans}: {b.AdetPolic} poliçe, {b.ToplamPrim:N2} TL");
```

---

## 4. Dosya ve Klasör İşlemleri

### Dosya Okuma / Yazma

```csharp
// Metin okuma
string icerik = System.IO.File.ReadAllText(dosyaYolu, System.Text.Encoding.UTF8);

// Satır satır okuma
string[] satirlar = System.IO.File.ReadAllLines(dosyaYolu, System.Text.Encoding.UTF8);

// Log dosyasına yazma (ekle)
string logSatiri = $"[{System.DateTime.Now:yyyy-MM-dd HH:mm:ss}] {mesaj}";
System.IO.File.AppendAllText(logDosyasi, logSatiri + System.Environment.NewLine,
    System.Text.Encoding.UTF8);

// Dosya var mı kontrol
bool varMi = System.IO.File.Exists(dosyaYolu);
```

### Klasör Yönetimi

```csharp
// Klasör oluştur (varsa hata vermez)
string arsivKlasor = System.IO.Path.Combine(@"C:\RPA\Arsiv",
    System.DateTime.Now.ToString("yyyy-MM"));
System.IO.Directory.CreateDirectory(arsivKlasor);

// Dosyayı taşı
string hedef = System.IO.Path.Combine(arsivKlasor,
    System.IO.Path.GetFileName(kaynakDosya));
System.IO.File.Move(kaynakDosya, hedef);

// Klasördeki son dosyayı bul
string enYeniDosya = System.IO.Directory.GetFiles(@"C:\İndirilenler", "*.pdf")
    .OrderByDescending(f => System.IO.File.GetLastWriteTime(f))
    .FirstOrDefault();
```

---

## 5. HTTP / REST Çağrıları

### GET İsteği

```csharp
// Senkron HTTP GET (async/await ile)
using (var http = new System.Net.Http.HttpClient())
{
    http.Timeout = System.TimeSpan.FromSeconds(30);
    http.DefaultRequestHeaders.Add("Authorization", $"Bearer {apiToken}");

    var yanit = await http.GetAsync($"https://api.sirket.com/police/{policeNo}");
    yanit.EnsureSuccessStatusCode();

    string json = await yanit.Content.ReadAsStringAsync();
    jsonYanit = json;  // Out değişkeni
}
```

### POST İsteği — JSON

```csharp
using (var http = new System.Net.Http.HttpClient())
{
    var icerik = new System.Net.Http.StringContent(
        Newtonsoft.Json.JsonConvert.SerializeObject(new {
            TCKimlik = tcKimlik,
            Plaka    = plaka,
            Brans    = brans
        }),
        System.Text.Encoding.UTF8,
        "application/json");

    var yanit = await http.PostAsync("https://api.sirket.com/teklif", icerik);
    yanit.EnsureSuccessStatusCode();
    sonucJson = await yanit.Content.ReadAsStringAsync();
}
```

---

## 6. JSON Parse ve Serialize

```csharp
// JSON string → JObject (dinamik erişim)
var jo = Newtonsoft.Json.Linq.JObject.Parse(jsonString);
string policeNo  = jo["data"]?["policeNo"]?.ToString() ?? "";
decimal prim     = jo["data"]?["prim"]?.Value<decimal>() ?? 0m;

// JArray döngüsü
var liste = Newtonsoft.Json.Linq.JArray.Parse(jsonArrayString);
foreach (var item in liste)
{
    string ad = item["ad"]?.ToString();
    Console.WriteLine(ad);
}

// Nesne → JSON string
string json = Newtonsoft.Json.JsonConvert.SerializeObject(new {
    PoliceNo  = policeNo,
    IslemZamani = System.DateTime.Now,
    Prim      = 4250.00m
}, Newtonsoft.Json.Formatting.Indented);

// WorkItem payload'dan okuma (null-safe)
string alan1 = item.payload["Alan1"]?.ToString() ?? "";
int    sayi  = item.payload["Sayi"]?.Value<int>() ?? 0;
```

---

## 7. Sigorta Sektörü — Hazır Fonksiyonlar

### TC Kimlik Algoritması (11 Hane Kontrol)

```csharp
bool TcGecerliMi(string tc)
{
    if (tc == null || tc.Length != 11 || tc[0] == '0') return false;
    if (!System.Text.RegularExpressions.Regex.IsMatch(tc, @"^\d{11}$")) return false;

    int[] d = tc.Select(c => int.Parse(c.ToString())).ToArray();
    int toplam1 = (d[0]+d[2]+d[4]+d[6]+d[8]) * 7 - (d[1]+d[3]+d[5]+d[7]);
    if (((toplam1 % 10) + 10) % 10 != d[9]) return false;

    int toplam2 = d.Take(10).Sum();
    return toplam2 % 10 == d[10];
}

tcGecerli = TcGecerliMi(tcKimlikDegiskeni);  // Out: bool
```

### Plaka Normalize

```csharp
// "34 ABC 123" → "34ABC123"
string PlakaNormalize(string plaka)
{
    return System.Text.RegularExpressions.Regex
        .Replace(plaka ?? "", @"[\s\-\.]", "")
        .ToUpperInvariant();
}
normalPlaka = PlakaNormalize(hamPlaka);
```

### Poliçe Vade Hesabı

```csharp
System.DateTime baslangic = System.DateTime.ParseExact(
    baslangicStr, "dd.MM.yyyy",
    System.Globalization.CultureInfo.InvariantCulture);

System.DateTime bitis = baslangic.AddYears(1).AddDays(-1);

int kalanGun = (bitis - System.DateTime.Today).Days;
bool yenilemeGerekli = kalanGun <= 30 && kalanGun >= 0;

bitisTarihiStr     = bitis.ToString("dd.MM.yyyy");
kalanGunSayisi     = kalanGun;
yenilemeGerekliMi  = yenilemeGerekli;
```

---

## Sık Hatalar ve Çözümler

| Hata | Sebep | Çözüm |
|---|---|---|
| `FormatException` | Tarih/sayı format uyumsuzluğu | `TryParse` + `ParseExact` kullan |
| `NullReferenceException` | `?.` olmadan null erişim | Her erişimde `?.` ve `?? ""` ekle |
| `InvalidCastException` | Yanlış tip cast | `Convert.ToX()` yerine `Parse()` |
| `COMException` Excel | COM object serbest bırakılmadı | `Marshal.ReleaseComObject` + `GC.Collect()` |
| `TaskCanceledException` | HTTP timeout | `http.Timeout` artır, retry ekle |
| `KeyNotFoundException` | JObject'te olmayan key | `jo.ContainsKey("alan")` kontrol et |
