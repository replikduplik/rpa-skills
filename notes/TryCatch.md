# TryCatch

## Ne Yapar

Hataları yakalar ve workflow'un çökmeden devam etmesini sağlar. **Her kritik işlem bloğu TryCatch içinde olmalıdır.** OpenRPA'da hata yönetiminin temel taşı.

---

## Yapısı

```
TryCatch
├── Try        ← Normal akış buraya gider
├── Catches    ← Hata tiplerine göre bloklar
│   ├── Catch (Exception e)       ← Tüm hatalar
│   ├── Catch (IOException e)     ← Sadece dosya hataları
│   └── Catch (TimeoutException e) ← Sadece timeout
└── Finally    ← Her durumda çalışır (opsiyonel)
```

---

## Parametreler

| Alan | Açıklama |
|---|---|
| `Try` | Korunan kod bloğu |
| `Catches` | Bir veya daha fazla Catch bloğu + hata tipi |
| `Finally` | Her zaman çalışır (temizleme için) |
| `e` | Yakalanan exception nesnesi (`e.Message`, `e.StackTrace`) |

---

## Adım Adım Kullanım

1. Toolbox → **TryCatch** sürükle
2. **Try** bölümüne korumak istediğin aktiviteleri ekle
3. Catches kısmında `+` ile yeni Catch ekle → hata tipini seç
4. Catch bloğuna şunları ekle:
   - `Log` (Error seviye, `e.Message`)
   - `UpdateWorkitem` (varsa, state: "failed")
   - Gerekirse `Rethrow` veya recovery aktivitesi
5. **Finally** bölümü: COM nesnesi temizleme, uygulama kapatma

---

## Catch Tipi Seçimi

```
Exception            → Tüm hataları yakala (en yaygın kullanım)
IOException          → Dosya/ağ hataları için özel işlem
TimeoutException     → Timeout durumunda özel retry
ArgumentException    → Geçersiz parametre
InvalidOperationException → Geçersiz durum
```

**Birden fazla Catch:** Önce spesifik tipler, en sona `Exception` (genel).

---

## Sık Yapılan Hatalar

### 1. Catch bloğunu boş bırakmak

```
❌ Catch (Exception e) → [boş]
   → Hata sessizce yutulur, robot yanlış çalışmaya devam eder

✅ Catch (Exception e)
   → Log: level=Error, message= "HATA: " + e.Message + " | " + e.StackTrace
   → UpdateWorkitem: state="failed", error=e.Message
```

### 2. Finally yerine Catch içinde temizlemek

```
❌ Catch bloğunda Excel.Quit()
   → Try başarılı olursa temizleme yapılmaz → Excel process kalır

✅ Finally bloğunda Excel.Quit() + Marshal.ReleaseComObject
   → Her durumda (başarı veya hata) çalışır
```

### 3. Rethrow yerine Exception at

```csharp
// ❌ Yeni exception — orijinal stack trace kaybolur
catch (Exception e)
{
    throw new Exception("Bir hata oluştu");
}

// ✅ Rethrow aktivitesi veya:
catch (Exception e)
{
    // Log, kayıt vb. yaptıktan sonra
    throw;  // Orijinal exception'ı tekrar fırlat
}
```

### 4. Çok geniş Try bloğu

```
❌ 50 aktivite tek Try içinde
   → Hata hangi adımda olduğunu bulmak zorlaşır

✅ Mantıksal bloklara böl:
   TryCatch (Giriş)
   TryCatch (Veri okuma)
   TryCatch (İşlem)
   TryCatch (Kayıt)
```

---

## Sigorta Sektörü Örnekleri

### Temel WorkItem İşleme

```
TryCatch
├── Try
│   ├── [Veri doğrulama]
│   ├── [Polisoft işlemleri]
│   └── UpdateWorkitem → state: "successful"
│         payload: { Sonuc: teklifNo, IslemZamani: DateTime.Now }
│
├── Catch (Exception e)
│   ├── Log → level: Error
│   │         message: "[WI:" + item.name + "] HATA: " + e.Message
│   └── UpdateWorkitem → state: "failed"
│         error: e.Message + " | " + e.StackTrace.Substring(0, 500)
│
└── Finally
    ├── [Excel varsa kapat]
    └── [COM nesneleri temizle]
```

### İç İçe TryCatch — Kısmi Hata Kurtarma

```
TryCatch (Ana)
├── Try
│   ├── InvokeWorkflow: GirisYap.xaml
│   │
│   ├── TryCatch (Web Portal — kurtarılabilir)
│   │   ├── Try: InvokeWorkflow: PortaldenTeklifAl.xaml
│   │   └── Catch (Exception e)
│   │         Log → "Portal hatası, Polisoft'tan devam"
│   │         portalHata = True
│   │
│   ├── If (portalHata == False)
│   │     InvokeWorkflow: PortalTeklifKaydet.xaml
│   └── InvokeWorkflow: PolisofeKaydet.xaml
│
└── Catch (Exception e)
    UpdateWorkitem → state: "failed"
```

### Retry ile TryCatch

```
deneme = 0
While (deneme < 3 AND basarili == False)
    TryCatch
    ├── Try
    │   ├── [işlem]
    │   └── basarili = True
    └── Catch (Exception e)
        ├── deneme = deneme + 1
        ├── Log → "Deneme " + deneme + " başarısız: " + e.Message
        └── Delay: 2000 * deneme   ← exponential backoff
```

---

## Finally — Ne Zaman Kullanılır?

| Kullanım | Örnek |
|---|---|
| COM temizleme | `xlApp.Quit()`, `Marshal.ReleaseComObject()` |
| Uygulama kapatma | Polisoft, SAP oturumu kapat |
| Geçici dosya silme | `File.Delete(tempDosya)` |
| Log: işlem bitti | `"İşlem tamamlandı (başarı veya hata)"` |

**Finally her zaman çalışır** — Try başarılı olsa da, Catch yakalasa da.

---

## Fork Notları

> Keşfedilen farklar buraya eklenir.
- TryCatch yapısı vanilla OpenRPA ile aynı görünüyor.
- Rethrow aktivitesi mevcut — doğrulandı.
