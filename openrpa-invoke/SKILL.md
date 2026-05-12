---
name: openrpa-invoke
description: >
  Use when calling OpenRPA workflows from outside (PowerShell, API) or from within
  another workflow, or when passing parameters between workflows. Covers PowerShell
  Invoke-OpenRPA, InvokeWorkflow activity, OpenFlow API, sync/async patterns,
  and insurance automation orchestration (Polisoft, web portals, multi-step flows).
  Triggers on: "invoke", "dışarıdan çalıştır", "PowerShell robot", "workflow çağır",
  "parametre geç", "InvokeWorkflow", "Invoke-OpenRPA", "tetikle", "API ile robot",
  "sigorta robotu başlat", "zamanlama", "cross-workflow".
---

# OpenRPA — Invoke (Çağırma) Rehberi

## Yöntem Seçim Tablosu

| Senaryo | Yöntem |
|---|---|
| Windows/PowerShell → robot | `Invoke-OpenRPA` (PowerShell) |
| OpenRPA workflow → başka workflow | `InvokeWorkflow` aktivitesi |
| Web servis / Python / Node.js → robot | OpenFlow SDK |
| Robot → OpenFlow Node-RED flow | `InvokeOpenFlow` aktivitesi |

---

## 1. PowerShell ile Invoke-OpenRPA

### Kurulum

```powershell
Import-Module "C:\Program Files\OpenRPA\Modules\OpenRPA.PowerShell.psd1"
```

### Temel Kullanım

```powershell
# Credential'ı ortam değişkeninden al — asla hardcode etme
$url = $env:OPENRPA_URL   # "wss://app.openiap.io" veya kendi sunucun
$jwt = $env:OPENRPA_JWT

# Basit çalıştırma
$result = Invoke-OpenRPA `
    -WorkflowName "PoliçeYenile" `
    -url          $url `
    -jwt          $jwt

# Parametre geçirerek — sigorta örneği
$result = Invoke-OpenRPA `
    -WorkflowName "TeklifAl" `
    -Parameters   @{
        TCKimlik = "12345678901"
        Plaka    = "34ABC123"
        Brans    = "Kasko"
        Baslangic = "01.01.2025"
    } `
    -Timeout           120 `
    -WaitForCompletion $true `
    -url $url -jwt $jwt

# Sonuçları kullan
if ($result.Basarili -eq $true) {
    Write-Host "Teklif No: $($result.TeklifNo) | Prim: $($result.PrimTutari)"
} else {
    Write-Error "Hata: $($result.HataMesaji)"
}
```

### Tüm Parametreler

| Parametre | Açıklama | Zorunlu |
|---|---|---|
| `-WorkflowName` | Workflow'un tam adı | Evet |
| `-url` | OpenFlow WebSocket URL | Evet |
| `-jwt` | Auth token | Evet |
| `-Parameters` | Hashtable — workflow değişkenleri | Hayır |
| `-RobotId` | Belirli robot üzerinde çalıştır | Hayır |
| `-Timeout` | Saniye (varsayılan 60) | Hayır |
| `-WaitForCompletion` | `$true` = senkron | Hayır |

### JWT Token Alma

```powershell
$token = Get-OpenRPAToken `
    -url      $env:OPENRPA_URL `
    -username $env:OPENRPA_USER `
    -password $env:OPENRPA_PASS

$env:OPENRPA_JWT = $token
```

### Dizi Parametresi (v1.4.57.13+ fix)

```powershell
# Array desteği — eski sürümlerde bozuktu, v1.4.57.13 ile düzeltildi
$result = Invoke-OpenRPA `
    -WorkflowName "TopluTeklifAl" `
    -Parameters   @{
        PlakaListesi = @("34ABC123", "06DEF456", "35GHI789")
        Brans        = "Trafik"
    } `
    -url $url -jwt $jwt
```

### Hata Yönetimi + Retry

```powershell
$maxDeneme = 3
$deneme    = 0

do {
    $deneme++
    try {
        $result = Invoke-OpenRPA `
            -WorkflowName "TeklifAl" `
            -Parameters @{ Plaka = $plaka } `
            -Timeout 90 -WaitForCompletion $true `
            -url $url -jwt $jwt

        if ($result.Basarili) { break }
        Write-Warning "Deneme $deneme başarısız: $($result.HataMesaji)"
    }
    catch {
        Write-Warning "Deneme $deneme exception: $_"
    }
    Start-Sleep -Seconds (5 * $deneme)  # exponential backoff
} while ($deneme -lt $maxDeneme)

if ($deneme -eq $maxDeneme -and -not $result.Basarili) {
    throw "Robot $maxDeneme denemede de başarısız oldu."
}
```

### Asenkron — Fire and Forget

```powershell
# Uzun süren robot — cevabı beklemeden devam et
$r = Invoke-OpenRPA `
    -WorkflowName      "GeceMigirateBatch" `
    -WaitForCompletion $false `
    -url $url -jwt $jwt

Write-Host "Robot başlatıldı. InstanceId: $($r.InstanceId)"
# Durumu sonra kontrol et veya WorkItem üzerinden izle
```

---

## 2. InvokeWorkflow Aktivitesi (OpenRPA İçinden)

### Yapılandırma

1. Toolbox → `InvokeWorkflow` sürükle
2. `WorkflowFileName` → Browse → `.xaml` seç
3. `Arguments` → `In / Out / InOut` değişkenleri eşleştir

### Parametre Yönleri

| Yön | Açıklama |
|---|---|
| `In` | Ana → alt workflow'a veri gönder |
| `Out` | Alt → ana workflow'a sonuç döndür |
| `InOut` | Her iki yönde |

### Sigorta Örneği — Modüler Teklif Akışı

**Alt workflow: `PoliçeSorgula.xaml`**
```
Giriş (In):  plaka (String), brans (String)
Çıkış (Out): mevcutPolice (String), bitisTarihi (DateTime), yenilemeGerekli (Boolean)
```

**Ana workflow'da:**
```
InvokeWorkflow: PoliçeSorgula.xaml
Arguments:
  plaka           [In]  ← plakaDegiskeni
  brans           [In]  ← bransDegiskeni
  mevcutPolice    [Out] → mevcutPoliceDegiskeni
  bitisTarihi     [Out] → bitisTarihiDegiskeni
  yenilemeGerekli [Out] → yenilemeGerekliDegiskeni

If (yenilemeGerekli = True)
  InvokeWorkflow: PoliçeYenile.xaml [In: mevcutPolice, bitisTarihi]
```

### Sigorta İçin Önerilen Modüler Yapı

```
Ana: SigortaOtomasyonu.xaml
├── InvokeWorkflow: GirisYap.xaml
│     [In: sistem (Polisoft/Web/SAP) | Out: oturum (Session)]
├── InvokeWorkflow: VeriAl.xaml
│     [In: workItemPayload | Out: musteriData]
├── InvokeWorkflow: VeriDogrula.xaml
│     [InOut: musteriData | Out: hatalar]
├── If (hatalar boş değil) → InvokeWorkflow: HataRaporla.xaml
├── InvokeWorkflow: Polisofte Gir.xaml
│     [In: musteriData, oturum | Out: teklifNo]
└── InvokeWorkflow: Bildir.xaml
      [In: teklifNo, musteriData]
```

**Her modülün kuralları:**
- Bağımsız çalışabilir (kendi TryCatch'i var)
- 200 satırı geçmeyen, tek sorumluluğu olan workflow
- Büyük DataTable geçirme — WorkItem payload kullan

---

## 3. OpenFlow API ile Robot Tetikleme

### Node.js (Web uygulaması → robot)

```javascript
const { client } = require('@openiap/nodeapi');

async function teklifRobotuBaslat(tcKimlik, plaka, brans) {
    const cli = new client();
    await cli.connect(process.env.OPENRPA_URL);
    await cli.signin({ username: process.env.OPENRPA_USER,
                       password: process.env.OPENRPA_PASS });

    // Workflow ID'yi önbelleğe al (her çağrıda sorgulamak yerine)
    const wf = await cli.Query({
        collectionname: "openrpa",
        query: { _type: "workflow", name: "TeklifAl" }
    });

    const result = await cli.Invoke({
        workflowid: wf[0]._id,
        payload: JSON.stringify({ TCKimlik: tcKimlik, Plaka: plaka, Brans: brans })
    });

    cli.close();
    return JSON.parse(result.payload);
}
```

### Python (Batch / ETL → robot)

```python
import asyncio, openiap, os

async def toplu_teklif_al(kayit_listesi):
    c = openiap.Client()
    await c.connect(os.environ["OPENRPA_URL"])
    await c.signin(username=os.environ["OPENRPA_USER"],
                   password=os.environ["OPENRPA_PASS"])

    for kayit in kayit_listesi:
        result = await c.invoke(
            workflowid="workflow-id",
            payload=kayit
        )
        print(f"Teklif No: {result.get('TeklifNo')}")

    c.close()
```

### InvokeOpenFlow Aktivitesi (Robot → Node-RED)

OpenFlow'daki Node-RED workflow'u doğrudan çağır:

```
InvokeOpenFlow
  WorkflowName: "mail-gonder"    ← Node-RED'de deploy edilmiş flow
  Payload:
    to:      "mudur@sirket.com"
    subject: "Teklif Raporu"
    body:    raporMetni
```

Node-RED içinde 2000+ sistem (e-posta, Slack, veritabanı, ERP API) bağlanabilir.

---

## Yöntem Seçim Rehberi

```
Nereden tetikliyorum?
├── Görev zamanlayıcı / script     → PowerShell Invoke-OpenRPA
├── Web uygulaması / API           → OpenFlow SDK (Node.js/Python)
├── OpenRPA workflow içinden       → InvokeWorkflow aktivitesi
└── Robot → dış sistem (mail vb.) → InvokeOpenFlow aktivitesi

Senkron mu gerekiyor?
├── Sonucu beklemem lazım          → WaitForCompletion=$true / await
├── Fire-and-forget                → WaitForCompletion=$false
└── Çok kayıt → paralel dağıtım   → WorkItem Queue (openrpa-openflow skill'e bak)
```

---

## Sigorta Otomasyon Örnekleri

### Gece Batch — Tüm Poliçeleri Yenile

```powershell
# Görev zamanlayıcıdan çalışır — her gece 02:00
$plakaListesi = Import-Csv "C:\Data\yenileme_listesi.csv"

foreach ($kayit in $plakaListesi) {
    try {
        $r = Invoke-OpenRPA `
            -WorkflowName "PoliçeYenile" `
            -Parameters @{ Plaka = $kayit.Plaka; PoliceNo = $kayit.PoliceNo } `
            -Timeout 180 -WaitForCompletion $true `
            -url $url -jwt $jwt

        "$($kayit.Plaka) | $($r.SonucMesaji)" | Add-Content "C:\Log\yenileme.log"
    }
    catch {
        "HATA | $($kayit.Plaka) | $_" | Add-Content "C:\Log\yenileme_hata.log"
    }
}
```

### Web Portaldan Teklif → Polisoft'a Kayıt

```
Ana Workflow
├── InvokeWorkflow: WebPortalGiris.xaml
│     [In: portal="allianz" | Out: session]
├── InvokeWorkflow: WebdenTeklifAl.xaml
│     [In: aracBilgi, session | Out: teklifData]
├── InvokeWorkflow: PoliceSoftAc.xaml
│     [Out: psSession]
└── InvokeWorkflow: PolisofeKaydet.xaml
      [In: teklifData, psSession | Out: psKayitNo]
```

---

## Yaygın Hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `Timeout expired` | Robot yavaş veya kilitlendi | Timeout artır, robot durumunu kontrol et |
| `Workflow not found` | İsim yanlış | OpenFlow'dan exact adı kontrol et |
| `Unauthorized` | JWT süresi doldu | Token yenile |
| `Array parameter error` | Eski sürüm | v1.4.57.13+ kullan |
| Parametre null geliyor | In/Out yön yanlış | Arguments'ta yönü kontrol et |
| Alt workflow değişikliği ana workflow'u etkiliyor | Tight coupling | Out parametreleri net tanımla, InOut yerine In+Out kullan |
