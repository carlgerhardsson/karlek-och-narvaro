# Kärlek & Närvaro 💛

En iPhone-anpassad webbapp för par som vill hålla relationen levande med dagliga tips och ett gemensamt foto-quiz.

**Live:** https://carlgerhardsson.github.io/karlek-och-narvaro/

---

## Vad appen gör idag

### Skärm 1 — Dagligt tips
- Visar ett inspirerande tips per dag om hur man är en bra partner
- 22 tips roterar automatiskt baserat på datum (en ny varje dag)
- Visar dagens datum, kategori och tipsnummer
- Knapp längst ner: **♡ Dagens fråga** → leder till foto-quizet

### Skärm 2 — Parkoppling (första gången)
- Båda partners anger ett **gemensamt parnamn** (t.ex. `CarlOchAnna`) och sitt **eget namn**
- Sparas i `localStorage` så de slipper fylla i det igen
- Det gemensamma namnet används som nyckel i Firebase för att synka svaren

### Skärm 3 — Dagligt foto-quiz
- Hämtar en bild per dag från `photos/`-mappen i Firebase Storage
- Bilden väljs baserat på dag på året (roterar automatiskt)
- Läser EXIF GPS-koordinater ur bildfilen via **exifr.js**
- **Om GPS finns:** reverse geocoding via Nominatim (OpenStreetMap) → tre svarsalternativ (rätt stad + två automatiskt genererade distraktorer inom ~150–350 km)
- **Om GPS saknas:** fritextruta där båda gissar platsen
- Firebase synkar svaren i realtid
- När **båda svarat** visas varandras svar och rätt plats avslöjas

---

## Teknisk stack

| Del | Teknik |
|-----|--------|
| Hosting | GitHub Pages (`carlgerhardsson/karlek-och-narvaro`) |
| Frontend | Vanilla HTML/CSS/JS — en enda fil (`index.html`) |
| Bildlagring | Firebase Storage (Blaze-plan) — mappen `photos/` |
| Realtidssynk | Firebase Realtime Database |
| EXIF-läsning | exifr.js v7 (CDN) |
| Geocoding | Nominatim / OpenStreetMap (gratis, ingen API-nyckel) |
| Typsnitt | Cormorant Garamond + Josefin Sans (Google Fonts) |

---

## Firebase-setup

### Storage-regler
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /photos/{fileName} {
      allow read: if true;
    }
  }
}
```

### CORS (körs en gång via Google Cloud Shell)
```bash
echo '[{"origin":["https://carlgerhardsson.github.io"],"method":["GET"],"maxAgeSeconds":3600}]' > cors.json
gsutil cors set cors.json gs://karlek-narvaro.firebasestorage.app
```

### Firebase Realtime Database-struktur
```
pairs/
  {parnamn}/
    days/
      {YYYY-MM-DD}/
        date: "2026-03-07"
        type: "photo"
        filename: "roma-2023.jpg"
        answers/
          {personnamn}/
            name: "Carl"
            answer: "Rom"         (valt alternativ eller fritext)
            correct: true         (null om fritext)
            type: "choice"        (eller "freetext")
            ts: 1741342800000
```

---

## Hur man lägger till fler bilder

1. Gå till [Firebase Console](https://console.firebase.google.com) → **Storage**
2. Öppna mappen `photos/`
3. Ladda upp nya bilder (JPG/HEIC med GPS-data ger bäst upplevelse)
4. Klart — appen väljer automatiskt en ny bild per dag baserat på antalet bilder

---

## Designbeslut
- Varm, organisk färgpalett: cream/warm/rose/gold
- Glassmorfism-kort med backdrop-filter
- Hjärtslag-animation på ♡-ikonen
- Animerade inladdningar med fadeIn + translateY
- iPhone-optimerad: `viewport-fit=cover`, `apple-mobile-web-app-capable`
- Laddningsindikator med spinner och statustext medan GPS och geocoding körs

---

## Hur man uppdaterar appen

1. Gör ändringar i `index.html` (via Claude eller manuellt)
2. Commita till `main`-branchen på GitHub
3. GitHub Pages uppdateras automatiskt inom ~1 minut

## Hur man lägger till hemskärmsikon (iPhone)
1. Öppna länken i **Safari**
2. Tryck på dela-ikonen → **Lägg till på hemskärmen**
3. Appen beter sig som en native app (utan webbläsarens krom)
