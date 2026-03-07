# Kärlek & Närvaro 💛

En iPhone-anpassad webbapp för par som vill hålla relationen levande med dagliga tips och ett gemensamt quiz.

**Live:** https://carlgerhardsson.github.io/karlek-och-narvaro/

---

## Vad appen gör idag

### Skärm 1 — Dagligt tips
- Visar ett inspirerande tips per dag om hur man är en bra partner
- 22 tips roterar automatiskt baserat på datum (en ny varje dag)
- Visar dagens datum, kategori och tipsnummer
- Knapp längst ner: **♡ Dagens fråga** → leder till quizet

### Skärm 2 — Parkoppling (första gången)
- Båda partners anger ett **gemensamt parnamn** (t.ex. `CarlOchAnna`) och sitt **eget namn**
- Sparas i `localStorage` så de slipper fylla i det igen
- Det gemensamma namnet används som nyckel i Firebase för att synka svaren

### Skärm 3 — Dagligt quiz
- En ny fråga varje dag (24 frågor om kärlek, psykologi och relationer)
- **Tre svarsalternativ (A, B, C)** — ett är rätt
- Man väljer sitt svar → det låses direkt
- Firebase synkar svaren i realtid
- När **båda svarat** visas varandras svar med ✓/✗ i grönt/rött
- Det rätta svaret avslöjas längst ner

---

## Teknisk stack

| Del | Teknik |
|-----|--------|
| Hosting | GitHub Pages (`carlgerhardsson/karlek-och-narvaro`) |
| Frontend | Vanilla HTML/CSS/JS — en enda fil (`index.html`) |
| Realtidssynk | Firebase Realtime Database (Spark-plan, gratis) |
| Typsnitt | Cormorant Garamond + Josefin Sans (Google Fonts) |

### Firebase-struktur
```
pairs/
  {parnamn}/
    days/
      {YYYY-MM-DD}/
        date: "2026-03-07"
        answers/
          {personnamn}/
            name: "Carl"
            choice: 1          (index 0-2)
            answer: "Oxytocin"
            correct: true
            ts: 1741342800000
```

### Designbeslut
- Varm, organisk färgpalett: cream/warm/rose/gold
- Glassmorfism-kort med backdrop-filter
- Hjärtslag-animation på ♡-ikonen
- Animerade inladdningar med fadeIn + translateY
- iPhone-optimerad: `viewport-fit=cover`, `apple-mobile-web-app-capable`

---

## Nästa steg — Google Photos-integration

### Idé
Ersätt de generiska quizfrågorna med bilder från ett privat Google Photos-album (t.ex. "Kärlek & Närvaro"). Frågan blir: **"Var togs den här bilden?"**

### Flöde
1. Användaren loggar in med Google (OAuth 2.0)
2. Appen hämtar en bild per dag från albumet via **Google Photos API**
3. **Om bilden har GPS-metadata:** generera tre ortnamn automatiskt (rätt + två nära/liknande platser)
4. **Om bilden saknar GPS-metadata:** fritext-svar där båda gissar platsen

### OAuth-setup som krävs
- Google Cloud Console → nytt projekt → aktivera **Google Photos Library API**
- Skapa OAuth 2.0-klienter (Web application)
- Lägg till `https://carlgerhardsson.github.io` som auktoriserad JavaScript-ursprung
- Scope: `https://www.googleapis.com/auth/photoslibrary.readonly`

### Utmaningar att lösa
- Google Photos API returnerar inte GPS direkt — koordinater finns i EXIF men måste extraheras via **Nominatim (OpenStreetMap)** för reverse geocoding
- OAuth-token måste hanteras i sessionStorage (inte localStorage) av säkerhetsskäl
- Bilderna serveras via Googles CDN med tidsbegränsade URL:er — behöver hanteras vid Firebase-synk
- Alternativgenerering: om GPS finns → hitta 2 platser inom ~50-200 km som distraktorer

### Filstruktur framöver (förslag)
```
index.html          — huvudfil (hem + setup + quiz)
photos.js           — Google Photos API + OAuth-logik
geocode.js          — reverse geocoding + alternativgenerering
firebase.js         — Firebase-konfiguration och helpers
README.md           — denna fil
```

---

## Hur man uppdaterar appen

1. Gör ändringar i `index.html` (via Claude eller manuellt)
2. Commita till `main`-branchen på GitHub
3. GitHub Pages uppdateras automatiskt inom ~1 minut
4. Appen uppdateras nästa gång användaren öppnar den i Safari

## Hur man lägger till hemskärmsikon (iPhone)
1. Öppna länken i **Safari**
2. Tryck på dela-ikonen → **Lägg till på hemskärmen**
3. Appen beter sig som en native app (utan webbläsarens krom)
