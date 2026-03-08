# Systemdesign & Arkitektur

En beskrivning av hur appen, Firebase och GitHub hänger ihop — hur data flödar, varför varje del finns och vilka beslut som tagits längs vägen.

---

## Översikt

```
┌─────────────────────────────────────────────────────────────┐
│                        ANVÄNDARE                            │
│                                                             │
│   📱 Carl (iPhone/Safari)    📱 Annika (Samsung/Chrome)     │
└────────────┬────────────────────────────┬───────────────────┘
             │                            │
             ▼                            ▼
┌─────────────────────────────────────────────────────────────┐
│              GitHub Pages (hosting)                         │
│         carlgerhardsson.github.io/karlek-och-narvaro        │
│                                                             │
│   index.html  ─────  en enda fil, all logik inbäddad        │
└────────────┬────────────────────────────────────────────────┘
             │  läser bilder + sparar/synkar svar
             ▼
┌─────────────────────────────────────────────────────────────┐
│                      Firebase (Google)                      │
│                   Projekt: karlek-narvaro                   │
│                                                             │
│   ┌─────────────────┐      ┌──────────────────────────┐     │
│   │ Realtime Database│      │       Storage            │     │
│   │  (svar & sync)  │      │   (foton i photos/)      │     │
│   └─────────────────┘      └──────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
             │  reverse geocoding (gratis, ingen nyckel)
             ▼
┌─────────────────────────────────────────────────────────────┐
│              Nominatim / OpenStreetMap                      │
│   Omvandlar GPS-koordinater → stadsnamn på svenska          │
└─────────────────────────────────────────────────────────────┘
```

---

## GitHub Pages — Hosting

**Vad:** Statisk webbhosting direkt från `main`-branchen i repot.

**Varför:** Gratis, automatisk deploy vid varje commit, ingen byggprocess. Hela appen är en enda HTML-fil utan beroenden som behöver kompileras.

**Flöde:**
1. Kod pushas till `main` på GitHub
2. GitHub Pages bygger om automatiskt (~1 minut)
3. Ny version är live på `carlgerhardsson.github.io/karlek-och-narvaro`

**Begränsning:** Statisk hosting — ingen server-side logik. All dynamik sköts via Firebase och JavaScript i webbläsaren.

---

## Firebase Realtime Database — Realtidssynk av svar

**Vad:** En JSON-databas i realtid. Alla ändringar propageras direkt till alla anslutna klienter.

**Varför:** Gör att Annika kan se när Carl svarat (och vice versa) utan att ladda om sidan. Ingen polling, inga websockets att hantera manuellt — Firebase sköter det.

**Datastruktur:**

```
pairs/
  {parnamn}/                        ← t.ex. "calleoannika"
    days/
      {YYYY-MM-DD}/                 ← t.ex. "2026-03-08"
        photo/
          date: "2026-03-08"
          filename: "goteborg.heic"
          answers/
            {personnamn}/           ← t.ex. "calle"
              name: "Calle"
              answer: "Göteborg"
              correct: true
              type: "choice"        ← eller "freetext"
              ts: 1741430400000
            {personnamn}/           ← t.ex. "annika"
              name: "Annika"
              answer: "Malmö"
              correct: false
              type: "choice"
              ts: 1741430520000
```

**Viktiga designbeslut:**

- **`/photo/answers/`** — Foto-svaren sparas i en separat nod för att inte krocka med gamla svar från den tidigare kunskapsquizen (som låg direkt under `/answers/`). Detta är en läxa från ett tidigt produktionsfel.
- **Parnamn som nyckel** — Båda partners måste ange exakt samma parnamn. Det används som nyckel i databasen. Inget login krävs.
- **`safeKey()`** — Parnamn och personnamn normaliseras (lowercase, specialtecken bort) innan de används som Firebase-nycklar, eftersom Firebase inte tillåter `/`, `.`, `#`, `$` m.fl. i nycklar.
- **`ts` (timestamp)** — Varje svar får en Unix-timestamp för att sortera vems svar som visas först (den som svarade senast visas sist).

---

## Firebase Storage — Bildlagring

**Vad:** Objektlagring för binärfiler (bilder).

**Varför:** Google Photos API slutade tillåta läsning av befintliga album i april 2025. Firebase Storage på Blaze-planen ger full kontroll — bilderna laddas upp manuellt via Firebase Console och serveras via CDN.

**Mapp:** Alla bilder ska ligga i `photos/` — appen listar hela mappen och väljer bild via `getDayOfYear() % totalImages`.

**Åtkomstkontroll:**
```
match /photos/{fileName} {
  allow read: if true;   ← publik läsning
}
```
Skrivning kräver inloggning (görs via Console, aldrig från appen).

**CORS:** Eftersom appen körs på `github.io` och bilderna hämtas från `firebasestorage.app` behövs CORS-konfiguration. Körs en gång via Google Cloud Shell:
```bash
echo '[{"origin":["https://carlgerhardsson.github.io"],"method":["GET"],"maxAgeSeconds":3600}]' > cors.json
gsutil cors set cors.json gs://karlek-narvaro.firebasestorage.app
```

---

## EXIF & GPS-läsning — exifr.js

**Vad:** JavaScript-bibliotek som läser metadata (EXIF) direkt ur bildfilen i webbläsaren.

**Varför:** GPS-koordinater är inbäddade i bildfilen av kameran/telefonen. Appen behöver aldrig skicka dessa till en server — allt sker lokalt.

**Flöde:**
1. Bild hämtas som `Blob` (rådata) från Firebase Storage
2. `exifr.gps(blob)` returnerar `{ latitude, longitude }` om koordinater finns
3. Koordinaterna skickas till Nominatim för reverse geocoding

**Om GPS saknas:** Appen faller tillbaka till fritextsvar — båda gissar och jämför efteråt.

---

## Nominatim / OpenStreetMap — Reverse Geocoding

**Vad:** Gratis API som omvandlar koordinater till platsnamn.

**Varför:** Inget API-nyckel krävs. Täcker hela världen. Returnerar svenska namn med `Accept-Language: sv`.

**Flöde för att bygga svarsalternativ:**
1. Rätt plats: `reverseGeocode(lat, lng)` → t.ex. "Göteborg"
2. Distraktorer: Samma anrop med koordinat-offset (~200–400 km bort) → t.ex. "Oslo", "Malmö"
3. Tre alternativ blandas slumpmässigt och presenteras som A/B/C

**Fallback:** Om Nominatim returnerar samma stad för flera offset-koordinater kompletteras med hårdkodade städer (Stockholm, Berlin, Paris etc.).

---

## HEIC-hantering — heic2any

**Vad:** JavaScript-bibliotek som konverterar HEIC/HEIF-bilder till JPEG direkt i webbläsaren.

**Varför:** iPhone sparar foton i HEIC-format. Safari stödjer detta nativt, men Android/Chrome gör det inte. Utan konvertering visas en bruten bild för Annika.

**Flöde:**
1. Kolla om filnamnet slutar på `.heic` eller `.heif`
2. Testa om webbläsaren kan visa HEIC nativt (liten test-bild)
3. Om inte → konvertera `Blob` till JPEG med `heic2any()`
4. Skapa en lokal URL med `URL.createObjectURL()` och sätt som `<img src>`

EXIF-läsningen sker alltid på **originalblobben** (innan konvertering) eftersom metadata kan gå förlorad vid konvertering.

---

## Daglig bildrotation

Appen väljer dagens bild deterministiskt — samma bild visas för båda partners oavsett när på dagen de öppnar appen:

```javascript
const idx = getDayOfYear() % items.length;
```

`getDayOfYear()` returnerar dag 1–365. Med t.ex. 50 bilder roterar appen 50 dagar i taget. Bilder sorteras i den ordning Firebase Storage returnerar dem (alfabetiskt på filnamn) — namnge filer för att styra ordningen om så önskas.

---

## Lokal state & localStorage

Inget login-system. Identitet hanteras med `localStorage`:

| Nyckel | Värde | Syfte |
|--------|-------|-------|
| `pairCode` | `"CarlOchAnnika"` | Gemensam nyckel i Firebase |
| `myName` | `"Calle"` | Visningsnamn och Firebase-undernyckel |

Rensas med "Byt parkod / namn"-länken i appen.

---

## Sammanfattning: Varför denna stack?

| Krav | Lösning | Alternativ som övervägdes |
|------|---------|--------------------------|
| Gratis hosting | GitHub Pages | Vercel, Netlify |
| Realtidssynk utan server | Firebase Realtime Database | Supabase, polling |
| Bildlagring med full kontroll | Firebase Storage | Google Photos API (slutade fungera april 2025) |
| GPS ur bilder | exifr.js (browser) | Server-side EXIF-parsing |
| Platsnamn från koordinater | Nominatim (gratis) | Google Maps API (kräver betalkort) |
| HEIC på Android | heic2any (browser) | Konvertera vid uppladdning |
| Enkel deploy | Single HTML-fil | React, Vue, etc. |
