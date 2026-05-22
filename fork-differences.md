# Fork Farkları — Epoch/X Creator Studio 2.0

Bu dosya, vanilla OpenRPA ile şirketin fork'u (**Epoch/X Creator Studio**) arasındaki farkları biriktirir.

**Güncelleme kuralı:** Yeni fark keşfedilince hemen ekle. Etkilenen skill/note'u da güncelle.

---

## Platform Kimliği

| | Vanilla | Epoch/X |
|---|---|---|
| **Araç adı** | OpenRPA | Epoch/X Creator Studio |
| **Orchestrator** | OpenFlow (MongoDB) | Epoch/X Orchestrator (MongoDB → PostgreSQL) |
| **Web otomasyon** | NativeMessaging (Chrome Extension) | Microsoft.Playwright (NuGet tabanlı) |
| **Versiyon yönetimi** | Yok | Azure DevOps Git entegrasyonu |
| **Log sistemi** | Log aktivitesi | Business Log (EpochxWriteLine + MetricLog + Timeout) |
| **Chrome Extension** | OpenRPA NM | epochxcreatorstudio (Chrome Web Store'da yayınlı) |

---

## 2026-05-21 Platform: Web Otomasyon — NM → Playwright

**Vanilla:** `NM.GetElement` aktivitesi ile Chrome/Firefox otomasyon.

**Epoch/X:** Playwright tabanlı özel aktiviteler:
- `Playwright Open URL` — tarayıcı + URL aç
- `Playwright Click` — elemente tıkla
- `Playwright Fill` — alana değer gir
- `Playwright Get` — değer oku
- `Playwright Exists` — element var mı?
- `Playwright Close Tab` — sekmeyi kapat
- `Playwright Close Browser` — tarayıcıyı kapat

**Selector farkı:**
```
// NM (eski)
selector: id="tcKimlikNo"

// Playwright (yeni)
selector: "xpath=//input[@id='tcKimlikNo']"
selector: "css=input#tcKimlikNo"
selector: "text=Giriş Yap"
```

**KRİTİK — Playwright XPath tırnak kuralı:**
```
// ❌ YANLIŞ — backslash kaçış hatası
"xpath=//input[@id=\'Username\']"

// ✅ DOĞRU — tek tırnak, kaçış yok
"xpath=//input[@id='Username']"
```

**Etki:** Tüm web portal (sigorta siteleri, bankacılık) workflow'ları.

**İlgili skill güncellendi mi:** openrpa-selector ✅, openrpa-workflow ✅

---

## 2026-05-21 Aktivite: Image Selector — Import Sonrası Kayıp

**Vanilla:** Image tanımları XAML içinde korunur.

**Epoch/X:** 2.0'a workflow import edildiğinde tüm image tanımları kayboluyor. İkinci import'ta da aynı sorun oluşuyor.

**Geçici Çözüm:**
1. Eski XAML'deki image ID'lerini al
2. Image dosyasını base64'e çevir
3. XAML'deki ID referanslarını base64 kodlu string ile değiştir

**Etki:** Image aktivitesi kullanan tüm Polisoft ve masa üstü workflow'ları.

**İlgili skill güncellendi mi:** openrpa-selector ✅

---

## 2026-05-21 Aktivite: Yeni Özel Aktiviteler (Epoch/X'e Özgü)

Vanilla OpenRPA'da bulunmayan, Epoch/X toolbox'ında yer alan aktiviteler:

| Aktivite | İşlevi |
|---|---|
| `Click Element and Verify` | Tıklama + doğrulama + retry |
| `Bulk Add Workitems` | Toplu WorkItem ekleme |
| `ClickCoordinates` | Koordinat bazlı tıklama |
| `Detector` | Görsel algılama aktivitesi |
| `Focus Element` | Elemente odaklan |
| `Foreach DataRow` | DataTable satır döngüsü |
| `For Each<>` | Generic döngü |
| `Get and Update RPA Settings` | Süreç/robot bazlı ayar okuma |
| `Create YouTrack Issue` | Hata bildirimi oluşturma |
| `Delete Workitem` | WorkItem silme |
| `Add Workitem` | Tekil WorkItem ekleme |
| `EpochxWriteLine` | Debug Play output (Console.WriteLine yerine) |
| `Timeout` (sarmalayıcı) | Toplam süre sınırı + business log |
| `MetricLog` | Performans metriği loglama |
| `Comment Out` | Aktiviteyi yorum satırına al |

**Etki:** Tüm Epoch/X workflow'ları — vanilla aktivite listesi geçersiz.

**İlgili skill güncellendi mi:** openrpa-workflow ✅

---

## 2026-05-21 Özellik: Versiyon Yönetimi (Git)

**Vanilla:** Versiyon yönetimi yok.

**Epoch/X:**
- Her Save'de otomatik snapshot alınır
- `Version` butonu ile commit mesajıyla Azure DevOps'a push yapılır
- **Son versiyon push edilmeden workflow çalıştırılamaz**
- Proje ismi zorunlu format: `[3 büyük harf][3 rakam]` (örn: `SIG101`, `OTO103`)
- Debug Play modu: versiyonlama istemeden, Business Log basmadan test çalışması

**Yaygın Hata:**
```
"Cannot run workflow: minimum required version is 1.5, but your version is 1.4.57. 
Please update EpochxCreatorStudio."
```
→ Creator Studio güncellemesi zorunlu.

**Etki:** Tüm geliştirme ve çalıştırma akışları.

---

## 2026-05-21 Aktivite: Console.WriteLine → EpochxWriteLine

**Vanilla:** `Console.WriteLine` InvokeCode içinde Output panelinde görünür.

**Epoch/X:** `Console.WriteLine` çağrıları desteklenmiyor — uyarı mesajı verir:
```
Console.WriteLine çağrıları hala mevcuttur. 
[aktivite adı] özel aktivitesine dönüştürülmemiştir.
```

**Çözüm:** `EpochxWriteLine` aktivitesi veya `Log` aktivitesi kullan.

**Etki:** InvokeCode içinde debug amacıyla Console.WriteLine kullanan tüm workflow'lar.

---

## 2026-05-21 Aktivite: Image.GetElement — Yeni Parametreler

**Vanilla:** Temel image eşleme.

**Epoch/X:** Ek parametreler:
- `retry` — başarısız olursa tekrar dene
- `retrydelaycount` — yeniden deneme aralığı
- `cache` — önceki eşleşmeyi önbellekle
- `matchmode` — `accurate` / `pyramid` / `pyramidfast`
- `tracklastmatch` — son eşleşme konumunu izle

**Etki:** Image tabanlı Polisoft ve desktop workflow'ları.

---

## 2026-05-21 Özellik: Business Log ID Tipi Değişikliği

**Vanilla / Epoch/X 1.x:** `str_MainID = "prefix_" + Guid.NewGuid()` (string)

**Epoch/X 2.0:** `guid_MainID = Guid.NewGuid()` (System.Guid)

```csharp
// ❌ Eski — 1.x tarzı, 2.0'da kırılıyor
string str_MainID = "POL" + Guid.NewGuid().ToString();

// ✅ Yeni — 2.0 standardı
System.Guid guid_MainID = System.Guid.NewGuid();
```

**Etki:** Business Log kullanan tüm workflow'ların InvokeCode bloklarını kırıyor.

---

## 2026-05-21 Aktivite: NM.GetElement — Yeni Parametreler

**Vanilla:** Temel NM element seçimi.

**Epoch/X:** `retry` ve `retrydelaycount` eklendi.

> Not: Web otomasyonun Playwright'a taşınmasıyla NM.GetElement kullanımı azalıyor.

---

## 2026-05-21 Aktivite: GetRPASettings

**Vanilla:** Yok.

**Epoch/X:** Workflow veya robot bazında ayar okuma aktivitesi.
- Önce workflow özelinde arar
- Bulamazsa robot ayarına bakar
- Bulamazsa null döner

Kullanım: mail şablonları, eşik değerleri, sistem adresleri gibi sabit değerleri workflow içine gömmeden yönet.

---

## 2026-05-21 Bağlantı: Orchestrator Bağlantı Formatı

**Vanilla OpenFlow:** `wss://app.openiap.io`

**Epoch/X Orchestrator:** `wss://orchestrator.sirket.com/` (müşteriye özel subdomain mümkün)

**Yaygın Hata:**
```
Disconnected from wss://... reason: Unable to connect to the remote server
```
→ Orchestrator URL'ini Settings'ten kontrol et. Yanlış URL → CPU %100 loop'a düşer.

---

## 2026-05-21 Özellik: NM Offline Sorunu

**Semptom:** Robot çalışırken Chrome kapatılıp açılınca NM `offline` durumuna düşüyor.
Status bar'da görünür: `NM : offline`

**Çözüm:** Robotu yeniden başlat.

**Kalıcı önlem:** Workflow başında NM durumunu kontrol eden aktivite ekle.

---

## 2026-05-21 Özellik: SAP Bridge Kapanmama (Düzeltildi)

**Vanilla:** SAP Bridge özelliği yok.

**Epoch/X 1.x:** Creator Studio kapatıldığında SAP Bridge arka planda açık kalıyordu.

**Epoch/X 2.0:** `Start Process` + SAP Bridge kapatma aktivitesi eklenerek düzeltildi.

---

## Keşfedilmesi Gereken Alanlar

```
[ ] Playwright — tüm selector sözdizimi dokümante edildi mi?
[ ] GetRPASettings — tüm parametre tipleri neler?
[ ] Timeout aktivitesi — iç içe kullanım destekleniyor mu?
[ ] Bulk Add Workitems — maksimum batch boyutu?
[ ] Debug Play modu — tüm kısıtlamalar neler?
[ ] Azure DevOps PAT — token yenileme süreci?
[ ] Mail → Microsoft Graph API geçişi tamamlandı mı?
```

---

## Güncelleme Protokolü

1. Çalışırken beklenmedik davranış görürsen:
   - Vanilla dokümanlarıyla karşılaştır
   - Farkı bu dosyaya ekle
   - Etkilenen skill/note'ta Fork Notları bölümüne not ekle

2. Claude sana vanilla OpenRPA sözdizimi söylerse ve Epoch/X'te farklı çalışıyorsa:
   - Fork farkı olarak kaydet
   - Claude'u düzelt: "Epoch/X'te bu farklı çalışıyor: ..."
 