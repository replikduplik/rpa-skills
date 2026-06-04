# Regex Aktiviteleri — Match, Matches, Replace

## Platform Aktiviteleri Varken InvokeCode Kullanma [KZ-09]

`System.Text.RegularExpressions` ile işlem yapmak için `InvokeCode` kullanmak
**[KZ-09] ihlalidir**. Epoch/X Creator Studio'da doğrudan `Match`, `Matches` ve `Replace`
aktiviteleri mevcuttur.

---

## Match — Tek Eşleşme Kontrolü

**Ne Yapar:** Bir metinde regex deseni var mı yok mu söyler.

| Özellik | Tür | Açıklama |
|---|---|---|
| `Input` | `InArgument<string>` | Aranacak metin |
| `Pattern` | `InArgument<string>` | Regex deseni |
| `Result` | `OutArgument<bool>` | Eşleşme var mı |

```
Match
  Input: str_PdfMetni
  Pattern: "(?i)TCKN[\s:]*([0-9]{11})"
  → Result → bool_TcknBulundu

If [bool_TcknBulundu]
  [TCKN işlemleri]
```

---

## Matches — Tüm Eşleşmeleri Döngüyle İşle

**Ne Yapar:** Metindeki tüm eşleşmeler üzerinde Break/Continue destekli döngü açar.

| Özellik | Tür | Açıklama |
|---|---|---|
| `Input` | `InArgument<string>` | Aranacak metin |
| `Pattern` | `InArgument<string>` | Regex deseni |
| `Results` | `OutArgument<Match[]>` | Tüm eşleşmeler |

```
Matches
  Input: str_PdfMetni
  Pattern: "\d{11}"     ← 11 haneli sayılar (TCKN/VKN)
  → her eşleşme için döngü

  Gövde:
    Assign: str_Deger = item.Value      ← eşleşen metin
    Assign: str_Grup  = item.Groups(1).Value  ← grup 1
    [işlemler]
```

---

## Replace — Regex ile Metin Değiştirme

**Ne Yapar:** Desenle eşleşen tüm kısımları değiştirir.

| Özellik | Tür | Açıklama |
|---|---|---|
| `Input` | `InArgument<string>` | Kaynak metin |
| `Pattern` | `InArgument<string>` | Regex deseni |
| `Replacement` | `InArgument<string>` | Değiştirme metni |
| `Result` | `OutArgument<string>` | Sonuç |

```
Replace
  Input: str_HamMetin
  Pattern: "[\r\n\t]+"       ← boşluk/satır sonu temizle
  Replacement: " "
  → Result → str_TemizMetin

Replace
  Input: str_Tckn
  Pattern: "(?<=.{5}).(?=.{1})"   ← ortasını maskele
  Replacement: "*"
  → Result → str_TcknMasked
```

---

## InvokeCode Yerine Platform Aktivitesi — Karşılaştırma

```
❌ YANLIŞ [KZ-09] — InvokeCode ile regex
InvokeCode (C#)
  var match = Regex.Match(str_PdfMetni, @"TCKN[\s:]*([0-9]{11})");
  if (match.Success)
      str_Tckn = match.Groups[1].Value;

✅ DOĞRU — Match aktivitesi
Match
  Input: str_PdfMetni
  Pattern: "TCKN[\s:]*([0-9]{11})"
  → Result → bool_EslesmeBulundu

If [bool_EslesmeBulundu]
  // Groups erişimi için Matches kullan:
  Matches
    Input: str_PdfMetni
    Pattern: "TCKN[\s:]*([0-9]{11})"
    Gövde:
      Assign: str_Tckn = item.Groups(1).Value
      Break   ← ilk eşleşme yeterli
```

---

## Yaygın Regex Desenleri — Sigorta Sektörü

| Amaç | Pattern | Notlar |
|---|---|---|
| TCKN (11 hane) | `(?<!\d)\d{11}(?!\d)` | Kenar boundary ile |
| VKN (10 hane) | `(?<!\d)\d{10}(?!\d)` | |
| Tarih (dd.MM.yyyy) | `\d{2}\.\d{2}\.\d{4}` | |
| Para tutarı | `[\d.,]+\s*(TL\|₺\|TRY)` | |
| Plaka | `[0-9]{2}\s*[A-Z]{1,3}\s*[0-9]{2,4}` | |
| E-posta | `[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}` | |

---

## Fork Notları

- Namespace: `EpochxCreatorStudio.Utilities`
- `Matches` aktivitesi `BreakableLoop` tabanlı — döngü içinde `Break` kullanılabilir
- `RegexOptions` bayrakları (`IgnoreCase`, `Compiled` vb.) aktivite parametreleri olarak mevcut
