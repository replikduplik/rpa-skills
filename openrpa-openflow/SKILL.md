---
name: openrpa-openflow
description: >
  Use when managing multiple OpenRPA robots, distributing work via queues, scheduling
  robots, monitoring execution, or designing insurance automation orchestration with
  WorkItems (Polisoft batch, web portal scraping, policy renewal, claim processing).
  Triggers on: "workitem", "kuyruk", "queue", "birden fazla robot", "orkestrasyon",
  "zamanlama", "OpenFlow", "batch", "poliçe yenileme toplu", "hasar kuyruğu",
  "robot izleme", "offline mod", "iş dağıtımı", "paralel robot".
---

# OpenRPA + OpenFlow Entegrasyon Rehberi

## Mimari

```
┌─────────────────────────────────────────────────┐
│                   OpenFlow                      │
│      Node.js + MongoDB + RabbitMQ               │
│  ┌────────────┐  ┌──────────┐  ┌─────────────┐ │
│  │ WorkItem   │  │ Schedule │  │ Credentials │ │
│  │ Queue      │  │ (Cron)   │  │ (Güvenli)   │ │
│  └────────────┘  └──────────┘  └─────────────┘ │
└──────────────────┬──────────────────────────────┘
                   │ WebSocket (wss://)
        ┌──────────┴──────────┐
   ┌────▼────┐          ┌─────▼────┐
   │ Robot 1  │          │ Robot 2  │
   │(OpenRPA) │          │(OpenRPA) │
   └──────────┘          └──────────┘
```

**Bağlantı:** Robot başlangıçta WebFlow'a bağlanır. Kısa ağ kesintileri tolere edilir.

---

## WorkItem (İş Öğesi) Kuyrukları

**Her işlenecek kayıt = bir WorkItem.** Poliçe, hasar, teklif, müşteri → hepsi WorkItem.

### WorkItem Yaşam Döngüsü

```
new → processing → successful
              ↘ failed → (retry) → new
```

### WorkItem Veri Yapısı — Sigorta

```json
{
  "_type":  "workitem",
  "wiq":    "policeye-yenileme-kuyruğu",
  "name":   "Poliçe PN-2024-98765 Yenileme",
  "state":  "new",
  "priority": 1,
  "payload": {
    "PoliceNo":    "PN-2024-98765",
    "Plaka":       "34ABC123",
    "TCKimlik":    "12345678901",
    "Brans":       "Kasko",
    "BitisTarihi": "31.01.2025",
    "Kanal":       "Polisoft"
  }
}
```

### OpenRPA'da WorkItem İşleme — Temel Şablon

```
Sequence
├── GetWorkItemQueue
│     WorkItemQueueName: "policeye-yenileme-kuyruğu"
│     Çıktı: wiq
│
└── While (True)
    ├── PopWorkitem
    │     wiq: wiq
    │     Çıktı: item
    │     [item null ise Break]
    │
    ├── Log → "İşleniyor: " + item.name
    │
    └── TryCatch
        ├── Try
        │   ├── [VeriDogrula — item.payload kontrol et]
        │   ├── [İş mantığı — Polisoft/web/SAP]
        │   └── UpdateWorkitem
        │         item: item, state: "successful"
        │         payload: { "Sonuc": teklifNo, "IslemZamani": DateTime.Now }
        │
        └── Catch (Exception e)
            ├── Log → level: Error, message: e.Message
            └── UpdateWorkitem
                  item: item
                  state: "failed"
                  error: e.Message + " | " + e.StackTrace.Substring(0,500)
```

### Payload'a Erişim (InvokeCode)

```csharp
// Güvenli erişim — null kontrolü dahil
string policeNo  = item.payload["PoliceNo"]?.ToString() ?? "";
string plaka     = item.payload["Plaka"]?.ToString() ?? "";
string brans     = item.payload["Brans"]?.ToString() ?? "Trafik";
DateTime bitis   = DateTime.Parse(item.payload["BitisTarihi"].ToString());

if (string.IsNullOrEmpty(policeNo))
    throw new Exception("PoliceNo boş — WorkItem geçersiz");
```

---

## Kuyruğa WorkItem Ekleme

### InvokeCode ile Toplu Ekleme (C#)

```csharp
// Excel'den okunan poliçe listesini kuyruğa ekle
var client = global::OpenRPA.Interfaces.global.webSocketClient;
var items  = new System.Collections.Generic.List<Newtonsoft.Json.Linq.JObject>();

foreach (System.Data.DataRow row in policeTablosu.Rows)
{
    // Idempotency: aynı poliçe zaten kuyruktaysa ekleme
    // (OpenFlow'da unique field ile kontrol edilebilir)
    var wi = new Newtonsoft.Json.Linq.JObject
    {
        ["_type"]    = "workitem",
        ["wiq"]      = "policeye-yenileme-kuyruğu",
        ["name"]     = $"Poliçe {row["PoliceNo"]} Yenileme",
        ["state"]    = "new",
        ["priority"] = 1,
        ["payload"]  = Newtonsoft.Json.Linq.JObject.FromObject(new {
            PoliceNo    = row["PoliceNo"].ToString(),
            Plaka       = row["Plaka"].ToString(),
            TCKimlik    = row["TCKimlik"].ToString(),
            Brans       = row["Brans"].ToString(),
            BitisTarihi = row["BitisTarihi"].ToString()
        })
    };
    items.Add(wi);
}

await client.InsertMany("openrpa", items.ToArray(), false, false, false);
eklemeSayisi = items.Count;
```

### PowerShell ile Ekleme

```powershell
$items = Import-Csv "C:\Data\yenileme_listesi.csv" | ForEach-Object {
    @{
        _type   = "workitem"
        wiq     = "policeye-yenileme-kuyruğu"
        name    = "Poliçe $($_.PoliceNo)"
        state   = "new"
        payload = @{
            PoliceNo    = $_.PoliceNo
            Plaka       = $_.Plaka
            BitisTarihi = $_.BitisTarihi
        }
    }
}

# OpenFlow REST API ile gönder
$body = $items | ConvertTo-Json -Depth 5
Invoke-RestMethod `
    -Uri         "$env:OPENFLOW_URL/api/v2/workitems/bulk" `
    -Method      POST `
    -Body        $body `
    -Headers     @{ Authorization = "Bearer $env:OPENRPA_JWT" } `
    -ContentType "application/json"
```

---

## Çok Robotlu Orkestrasyon

### Rekabetçi Kuyruk (Competitive Queue)

```
Kuyruk: "teklif-kuyruğu" (500 kayıt)
              ↓
    ┌─────────┴──────────┐
  Robot 1             Robot 2
  PopWorkitem          PopWorkitem
  (çakışma olmadan OpenFlow dağıtır)
```

Her robot aynı kuyruğu yoklar — yük dengeleme otomatik.

### Sigorta Pipeline — Adım Adım Kuyruk

```
Adım 1 → Robot 1: Excel → müşteri verisi al → "dogrulama-kuyruğu"
Adım 2 → Robot 2: "dogrulama-kuyruğu" → TC/VKN kontrol → "teklif-kuyruğu"
Adım 3 → Robot 3: "teklif-kuyruğu"    → web portaldan teklif al → "kayit-kuyruğu"
Adım 4 → Robot 4: "kayit-kuyruğu"     → Polisoft'a kayıt → "bildirim-kuyruğu"
Adım 5 → Robot 5: "bildirim-kuyruğu"  → mail/SMS gönder → done
```

Her adım bağımsız ve yeniden başlatılabilir.

### Öncelik (Priority) ile Acil İşler

```json
{ "priority": 0, "name": "Normal teklif" }
{ "priority": 1, "name": "ACİL — VIP müşteri poliçesi" }
```

Yüksek priority WorkItem önce işlenir.

---

## Robot Zamanlama

### OpenFlow UI'dan (Kodsuz)

1. OpenFlow → Workflows → robota tıkla → "Schedule" sekmesi
2. Cron ifadesi gir + timezone

```
0 2 * * 1-5        → Haftaiçi her gece 02:00 (gece batch)
0 8 * * *          → Her gün sabah 08:00 (günlük rapor)
*/30 9-17 * * 1-5  → Mesai saatlerinde her 30 dakika
0 9,12,15 * * 1-5  → 09:00, 12:00, 15:00
0 1 L * *          → Her ayın son günü 01:00 (aylık rapor)
```

Test: [crontab.guru](https://crontab.guru) | Timezone: `Europe/Istanbul`

### Programatik Zamanlama (Node.js)

```javascript
await cli.InsertOne({
    collectionname: "openrpa",
    item: {
        _type:      "schedule",
        workflowid: wfId,
        name:       "Gece Poliçe Yenileme",
        cron:       "0 2 * * 1-5",
        timezone:   "Europe/Istanbul",
        enabled:    true
    }
});
```

---

## Credential Yönetimi (Güvenlik — Kritik)

OpenFlow'da güvenli credential deposu kullan — workflow'da parola hardcode etme.

**OpenFlow UI:** Settings → Credentials → "Yeni Ekle"

```csharp
// InvokeCode — credential al
var cred = await global::OpenRPA.Interfaces.global.webSocketClient
    .GetCredential("polisoft-prod");     // OpenFlow'daki isim
string user = cred.username;
string pass = cred.password;

// Şimdi bu bilgileri TypeText'e ver
// Asla Log aktivitesine pass yazma!
```

**Credential isimlendirme standardı:**
```
polisoft-prod          ← Polisoft production
polisoft-test          ← Polisoft test ortamı
allianz-portal         ← Allianz web portalı
axa-portal             ← Axa web portalı
sap-prod               ← SAP üretim
```

---

## İzleme ve Hata Yönetimi

### Robot Durumu

OpenFlow UI → Robots:
- 🟢 Online — bağlı, hazır
- 🟡 Busy — çalışıyor
- 🔴 Offline — bağlantı yok

### WorkItem İzleme

OpenFlow → WorkItems → filtrele:
- Kuyruk, durum, tarih bazlı filtre
- `state=failed` → hata mesajını oku → hatayı düzelt → "Retry" ile tekrar kuyruğa al
- Toplu retry: birden fazla seç → "Retry All"

### Workflow Geçmişi

OpenFlow → Workflows → "Instances":
- Başlangıç/bitiş zamanı, durum, hata detayı
- Uzun süren robotları buradan tespit et

### Sigorta İçin Önerilen Log Formatı

```
[2025-01-15 09:23:11] [WI:wi_abc123] [BAŞLADI] PoliceNo: PN-98765, Plaka: 34ABC123
[2025-01-15 09:23:45] [WI:wi_abc123] [TEKLIF] Teklif No: T-2025-001, Prim: 4.250,00 TL
[2025-01-15 09:23:52] [WI:wi_abc123] [TAMAM] Polisoft kayıt: PS-12345
[2025-01-15 09:23:53] [WI:wi_abc123] [BITTI] Toplam süre: 42 sn
```

---

## Offline Mod

OpenFlow bağlantısı olmadan çalıştırmak için:

**`OpenRPA.exe.config`:**
```xml
<appSettings>
  <add key="wsurl"   value="" />
  <add key="offline" value="true" />
</appSettings>
```

**Offline'da ne değişir:**

| Özellik | Online | Offline |
|---|---|---|
| WorkItem Queue | ✅ | ❌ |
| Çok robot | ✅ | ❌ |
| Zamanlama | OpenFlow Cron | Windows Görev Zamanlayıcı |
| Credential | OpenFlow | app.config |
| İzleme | OpenFlow UI | Yerel log |

**Offline zamanlama:**
```powershell
$action  = New-ScheduledTaskAction `
    -Execute  "C:\Program Files\OpenRPA\OpenRPA.exe" `
    -Argument "--run GeceBatch --offline"
$trigger = New-ScheduledTaskTrigger -Daily -At "02:00"
Register-ScheduledTask -TaskName "GecePoliceyeYenileme" `
    -Action $action -Trigger $trigger -RunLevel Highest -Force
```

---

## Sigorta Sektörü Örnek Kuyruk Tasarımı

```
policeye-yenileme-kuyruğu   → Vadesi yaklaşan poliçeler (30 gün öncesi)
yeni-teklif-kuyruğu         → Gelen teklif talepleri
hasar-bildirim-kuyruğu      → Yeni hasar bildirimleri
zeyilname-kuyruğu           → Değişiklik talepleri
iptal-kuyruğu               → İptal işlemleri
rapor-kuyruğu               → Günlük/aylık raporlar
bildirim-kuyruğu            → Mail/SMS bildirimleri (son adım)
```

Her branş (Trafik, Kasko, Konut, DASK, Sağlık) ayrı kuyrukla da yönetilebilir.

---

## Yaygın Sorunlar

| Sorun | Sebep | Çözüm |
|---|---|---|
| Robot offline görünüyor | Firewall / proxy | 80/443 aç, WebSocket desteğini doğrula |
| WorkItem alınamıyor | Rol/izin eksik | OpenFlow → Security → Roles kontrol |
| Zamanlama çalışmıyor | Cron yanlış / timezone | crontab.guru test, Europe/Istanbul |
| Aynı item iki kez işlendi | Race condition | PopWorkitem atomik — OpenFlow halleder, ancak idempotency ekle |
| Credential bulunamıyor | İsim yanlış | OpenFlow → Credentials → exact ismi doğrula |
| Failed item artıyor | Sistematik hata | Retry öncesi hatayı düzelt, root cause bul |
