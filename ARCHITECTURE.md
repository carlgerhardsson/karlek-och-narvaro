# Systemdesign & Arkitektur

En beskrivning av hur appen, Firebase och GitHub hänger ihop — vad som körs var, hur data flödar och varför varje del finns.

---

## Var körs vad?

Det finns ingen server i den här appen. All logik körs antingen i **mobilens webbläsare** eller är lagrad statiskt i **molntjänster**. GitHub och Firebase är bara lagring — de kör ingen kod.

```
┌──────────────────────────────────────────────────────────────────┐
│                    MOBILENS WEBBLÄSARE                           │
│         (Safari på iPhone, Chrome på Samsung)                    │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  index.html  —  laddas ned en gång, körs lokalt            │  │
│  │                                                            │  │
│  │  • Visar UI (tips, quiz, resultat)                         │  │
│  │  • Väljer dagens bild (getDayOfYear % antal bilder)        │  │
│  │  • Hämtar bildfil som rådata (Blob)                        │  │
│  │  • Läser GPS ur bilden med exifr.js          ← lokalt      │  │
│  │  • Konverterar HEIC → JPEG med heic2any      ← lokalt      │  │
│  │  • Frågar Nominatim om platsnamn             → internet    │  │
│  │  • Sparar svar till Firebase                 → internet    │  │
│  │  • Lyssnar på partners svar i realtid        ← internet    │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
          │ laddar ned index.html        │ hämtar bildfil
          ▼                              ▼
┌──────────────────┐          ┌────────────────────────┐
│  GitHub Pages    │          │  Firebase Storage       │
│  (statisk fil)   │          │  photos/bild.heic       │
└──────────────────┘          └────────────────────────┘
          │ sparar/läser svar            │ platsnamn från GPS
          ▼                              ▼
┌──────────────────┐          ┌────────────────────────┐
│  Firebase        │          │  Nominatim             │
│  Realtime DB     │          │  (OpenStreetMap API)   │
└──────────────────┘          └────────────────────────┘
```

---

## Steg-för-steg: Vad händer när du öppnar appen?

### 1. Webbläsaren laddar appen
Telefonen hämtar `index.html` från GitHub Pages. Det är en enda fil (~37 KB) som innehåller all HTML, CSS och JavaScript. Inget annat behöver laddas ned för att appen ska fungera grundläggande — utom externa bibliotek (Firebase SDK, exifr, heic2any) som hämtas från CDN.

### 2. Startsidan visas direkt
Dagens tips väljs ur en hårdkodad lista med 22 tips, baserat på vilken dag på året det är (`getDayOfYear() % 22`). Inget nätverksanrop krävs — tipset finns redan i filen.

### 3. Du trycker på "♡ Dagens fråga"
Appen kollar om parnamn och personnamn finns sparade i telefonens `localStorage`. Om inte visas inställningsskärmen där du fyller i dem. De sparas lokalt och behöver aldrig fyllas i igen.

### 4. Appen hämtar bildlistan från Firebase Storage
```
Mobilen → Firebase Storage: "Lista alla filer i photos/"
Firebase → Mobilen: [bild1.heic, bild2.heic, bild3.jpg, ...]
```
Appen räknar ut index: `getDayOfYear() % antal bilder`. Samma beräkning görs på båda telefonerna — de väljer därmed alltid samma bild samma dag utan att kommunicera med varandra.

### 5. Bildfilen hämtas som rådata
```
Mobilen → Firebase Storage: "Ge mig download-URL för bild3.heic"
Firebase → Mobilen: "https://firebasestorage.googleapis.com/..."
Mobilen → Firebase CDN: "Hämta filen"
Firebase CDN → Mobilen: [rådata, ~3-8 MB]
```
Filen sparas i minnet som en `Blob` — ett binärt dataobjekt.

### 6. GPS läses ur bilden — helt lokalt, inget nätverksanrop
Biblioteket `exifr.js` tolkar EXIF-metadata direkt ur Blob:en i webbläsarens minne:
```
exifr.gps(blob) → { latitude: 57.7089, longitude: 11.9746 }
```
Inga koordinater skickas till någon server i detta steg.

### 7. HEIC konverteras till JPEG om det behövs — helt lokalt
Om filen är `.heic` och webbläsaren är Chrome/Android (som inte stödjer HEIC):
```
heic2any({ blob, toType: 'image/jpeg' }) → jpegBlob
URL.createObjectURL(jpegBlob) → "blob:https://..."
<img src="blob:https://...">
```
Konverteringen sker i webbläsarens minne. Ingen data lämnar telefonen i detta steg.

### 8. Platsnamnet slås upp via Nominatim
Med GPS-koordinaterna från steg 6 görs tre–fem anrop till OpenStreetMap:
```
Mobilen → Nominatim: "Vad heter platsen vid 57.7089, 11.9746?"
Nominatim → Mobilen: "Göteborg"

Mobilen → Nominatim: "Vad heter platsen vid 60.5, 13.5?" (offset för distraktor)
Nominatim → Mobilen: "Falun"

Mobilen → Nominatim: "Vad heter platsen vid 55.6, 13.0?" (offset för distraktor)
Nominatim → Mobilen: "Malmö"
```
Tre alternativ blandas slumpmässigt och visas som A/B/C. Rätt svar är det första.

Om bilden saknar GPS hoppas steg 6–8 över och ett fritextsvar visas istället.

### 9. Du väljer ett svar
Svaret sparas till Firebase Realtime Database:
```
Mobilen → Firebase DB:
  pairs/calleoannika/days/2026-03-08/photo/answers/calle = {
    name: "Calle",
    answer: "Göteborg",
    correct: true,
    type: "choice",
    ts: 1741430400000
  }
```

### 10. Realtidslyssnaren triggas på partnerns telefon
Firebase skickar automatiskt ut en notis till alla som lyssnar på samma sökväg. Annikas telefon tar emot Calles svar direkt utan att hon behöver göra något:
```
Firebase DB → Annikas telefon: "Calle har svarat"
Annikas telefon: uppdaterar UI med Calles svar
```
När Annika också svarat visas båda svaren och rätt plats avslöjas.

---

## Vad händer vid deploy?

```
Calle/Claude skriver kod
        ↓
Pushas till main på GitHub
        ↓
GitHub Pages bygger om automatiskt (~1 minut)
        ↓
Nästa gång telefonen laddar appen hämtas den nya index.html
```

Inga byggsteg, ingen CI/CD-pipeline, ingen server att starta om. GitHub Pages serverar filen direkt.

---

## Firebase-datastruktur

```
pairs/
  {parnamn}/                          ← t.ex. "calleoannika"
    days/
      {YYYY-MM-DD}/                   ← t.ex. "2026-03-08"
        photo/
          date: "2026-03-08"
          filename: "goteborg.heic"
          answers/
            calle/
              name: "Calle"
              answer: "Göteborg"
              correct: true
              type: "choice"
              ts: 1741430400000
            annika/
              name: "Annika"
              answer: "Malmö"
              correct: false
              type: "choice"
              ts: 1741430520000
```

**Viktiga designbeslut:**
- **`/photo/answers/`** — Separerat från `/answers/` för att inte krocka med gamla svar från den tidigare kunskapsquizen. Läxa från ett tidigt produktionsfel.
- **Parnamn som nyckel** — Inget login. Båda anger samma parnamn, det används som nyckel i databasen.
- **`safeKey()`** — Parnamn normaliseras (lowercase, specialtecken → `_`) eftersom Firebase inte tillåter `.`, `/`, `#`, `$` i nycklar.
- **`!data.exists()` i DB-regler** — Ett svar kan bara skrivas en gång per person och dag. Ingen kan skriva över sin partners svar.

---

## Lokal state i telefonen

Inget login-system. Identitet lagras i telefonens `localStorage` (persistent, men bara på den enheten):

| Nyckel | Exempel | Syfte |
|--------|---------|-------|
| `pairCode` | `"CarlOchAnnika"` | Gemensam nyckel i Firebase |
| `myName` | `"Calle"` | Visningsnamn och Firebase-undernyckel |

Rensas via "Byt parkod / namn" i appen.

---

## Säkerhetsgränser

| Vad | Skydd |
|-----|-------|
| Firebase API-nyckel i källkod | Nyckeln är begränsad till `carlgerhardsson.github.io` i Google Cloud |
| Vem kan läsa bilder | Publik läsning tillåten (avsiktligt) |
| Vem kan ladda upp bilder | Endast via Firebase Console (write: false i Storage-regler) |
| Vem kan skriva svar | Vem som helst med rätt parnamn — men bara en gång per dag per person |
| Överskrivning av svar | Blockerat med `!data.exists()` i Database-regler |

---

## Sammanfattning: Varför denna stack?

| Krav | Lösning | Alternativ som övervägdes |
|------|---------|-----------------------------|
| Gratis hosting | GitHub Pages | Vercel, Netlify |
| Ingen server att drifta | Allt i webbläsaren | Node.js backend |
| Realtidssynk | Firebase Realtime Database | Supabase, polling |
| Bildlagring | Firebase Storage | Google Photos API (stängde april 2025) |
| GPS ur bilder | exifr.js i webbläsaren | Server-side EXIF-parsing |
| Platsnamn från koordinater | Nominatim (gratis) | Google Maps API (kräver betalkort) |
| HEIC på Android | heic2any i webbläsaren | Konvertera vid uppladdning |
| Enkel deploy | En enda HTML-fil | React, Vue, byggprocess |
