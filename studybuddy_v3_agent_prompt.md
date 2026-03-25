# StudyBuddy v3 — Agent Prompt (volledige rebuild)

Kopieer ALLES hieronder als eerste bericht naar je AI agent.

---

```
Bouw een volledig werkende PWA genaamd "StudyBuddy". Dit is een app voor kinderen van 10-12 jaar (basisonderwijs België) die er op hun telefoon uitziet en werkt als een native app, zowel op iOS als Android.

De app heeft twee functies:
1. OEFENEN VOOR EEN TOETS — genereert oefenvragen op basis van leerstof
2. HUISWERK NAKIJKEN — controleert of huiswerk correct is en geeft feedback

## TECHNISCH

- Eén index.html + manifest.json, geen frameworks, geen npm
- HTML + CSS + vanilla JavaScript
- Google Fonts: Nunito (body) + Quicksand (headings)
- Azure Computer Vision Read API voor OCR
  - Endpoint: https://huiswerk-ocr-vision.cognitiveservices.azure.com/
  - API Key: QYQfDL8xWMfT6HaqmdtHRo01GktrYC4o6O45rPXvvtNhe16ok2F9JQQJ99CCAC5RqLJXJ3w3AAAFACOG9cWF
  - API call: POST naar {endpoint}computervision/imageanalysis:analyze?api-version=2024-02-01&features=read
  - Header: Ocp-Apim-Subscription-Key: {key}
  - De ruwe tekst uit het response pad: result.readResult.content
  - Belangrijk: de Azure OCR mag de tekst NIET corrigeren of interpreteren — geef de ruwe output rechtstreeks door aan Claude voor analyse.
- Claude API via fetch (model: claude-haiku-4-5-20251001)
- Web Speech API voor spraakherkenning
- Alle state in één globaal STATE object
- localStorage voor: apiKey, model, questionCount, language

## DESIGN PRINCIPES

De app moet eruitzien als een NATIVE iOS/Android app, niet als een website:

- Volledig schermvullend, geen scrollbare "webpagina"-feel
- Elk scherm vult exact het viewport (100dvh)
- Navigatie via schermen die in- en uitsliden (geen page scroll)
- Bottom safe area padding voor iOS (env(safe-area-inset-bottom))
- Geen browser-achtige elementen, geen underlined links
- Touch-optimized: minimum touch target 48px
- Vloeiende transitions tussen schermen (slide left/right, 300ms ease)
- Status bar integratie via meta theme-color
- Kleuren: primary #6C5CE7, accent #FD79A8, success #00B894, bg #F0F0FF, cards #FFFFFF
- Border-radius: 20px voor cards, 12px voor knoppen
- Zachte schaduwen: 0 4px 20px rgba(108,92,231,0.15)
- Vrolijk en kindvriendelijk: emoji's in knoppen, speelse illustraties

## APP FLOW — 5 SCHERMEN

### SCHERM 1: TAALKEUZE (welkomstscherm)

Volledig scherm met:
- Groot StudyBuddy logo bovenaan (🧠 emoji + "StudyBuddy" in Quicksand bold)
- Ondertitel: "Jouw slimme studiemaatje!"
- Drie grote, vierkante taalknoppen verticaal gestapeld:
  - 🇧🇪 Nederlands
  - 🇫🇷 Français
  - 🇬🇧 English
- Klein tandwieltje (⚙️) rechtsboven voor instellingen
- De gekozen taal wordt opgeslagen in localStorage en bij volgende bezoeken wordt dit scherm overgeslagen (maar altijd bereikbaar via instellingen)

Na taalkeuze: slide naar Scherm 2.

### SCHERM 2: WAT WIL JE DOEN?

Twee grote kaarten die elk ~45% van het scherm innemen, verticaal gestapeld:

KAART 1: "📝 Oefenen voor een toets"
- Subtekst: "Maak een foto van je les en ik maak oefenvragen"
- Achtergrondkleur: gradient primary → primary-dark
- Witte tekst

KAART 2: "✅ Huiswerk nakijken"  
- Subtekst: "Maak een foto van je huiswerk en ik kijk het na"
- Achtergrondkleur: gradient success → #00CEC9
- Witte tekst

Onderaan klein: "Taal wijzigen" linkje

Na keuze: slide naar Scherm 3.

### SCHERM 3: FOTO'S / UPLOAD

Bovenbalk met:
- ← terug knop
- Titel: "Foto's van je les" of "Foto's van je huiswerk" (afhankelijk van keuze)

Centraal twee opties (grote knoppen):
- 📸 "Neem foto's" → opent camera
- 📄 "Upload PDF" → opent file picker (accept=".pdf,image/*")

#### CAMERA MODUS (als ze "Neem foto's" kiezen):

- Camera opent via <input type="file" accept="image/*" capture="environment">
- Na elke foto: toon een THUMBNAIL GRID van alle genomen foto's
- Elke thumbnail heeft een ✕ knop om te verwijderen
- Twee knoppen onderaan:
  - "📸 Nog een foto" (secundair, outline stijl) → opent camera opnieuw
  - "✅ Klaar!" (primair, groen, groot) → ga naar verwerking
- De grid toont max 3 foto's per rij, scrollbaar als er meer zijn
- Toon aantal foto's: "3 foto's genomen"

#### PDF MODUS (als ze "Upload PDF" kiezen):

- File picker opent
- Na selectie: toon bestandsnaam + grootte
- Knop: "✅ Ga verder" → ga naar verwerking

### SCHERM 4: VERWERKING + TEKST REVIEW

Split in twee fases:

FASE 1: VERWERKING (automatisch)
- Volledig scherm met spinner
- Stappen die één voor één op groen springen:
  1. "📷 Foto's worden gelezen..." → "✅ Foto's gelezen!"
  2. "🔍 Tekst wordt herkend..." → "✅ Tekst herkend!"
- Bij meerdere foto's: verwerk ze sequentieel, combineer alle tekst

FASE 2: TEKST REVIEW
- Scrollbaar tekstveld (textarea) met de herkende tekst
- Boven het tekstveld: "Controleer of de tekst klopt. Je mag aanpassen."
- Karakter-teller onderaan het tekstveld
- Optioneel veld: "📚 Vak (optioneel)" — text input, placeholder "bv. Aardrijkskunde"
- Twee knoppen onderaan (fixed aan de onderkant van het scherm):
  - "← Opnieuw" (secundair)
  - "✨ Start!" (primair, groen, groot)

Bij klik op "Start": stuur naar Claude API, toon laadscherm:
  3. "🧠 Vragen worden gemaakt..." of "🧠 Huiswerk wordt nagekeken..."
  4. "✅ Klaar!"

Dan: slide naar Scherm 5.

### SCHERM 5A: OEFENVRAGEN (als modus = toets)

Bovenbalk:
- ← terug naar home
- Score badge: "3/10 goed"

FLASHCARD WEERGAVE:
- Eén kaart per keer, vult de beschikbare ruimte
- CSS 3D flip animatie bij tik
- Voorkant: vraagnummer + vraagtekst (wit, border primary)
- Achterkant: antwoord (gradient primary achtergrond, witte tekst)

BIJ OPEN VRAGEN — SPECIALE WEERGAVE:
- De vraag staat bovenaan
- Daaronder een tekstveld voor het antwoord
- NAAST het tekstveld: een microfoon-knop 🎤
- Bij klik op 🎤:
  - Er verschijnt een modal/overlay met:
    - Grote pulserende microfoon-animatie (rood/oranje)
    - Tekst: "Ik luister... spreek je antwoord"
    - Live tekst die verschijnt terwijl ze spreken
    - Knop: "✅ Klaar met spreken"
  - Bij klik "Klaar": modal sluit, gesproken tekst verschijnt in het tekstveld
  - Gebruik Web Speech API: window.SpeechRecognition of webkitSpeechRecognition
  - Taal instellen op basis van STATE.lang (nl-BE, fr-BE, en-GB)
- Onder het tekstveld: groene knop "Controleer antwoord"
- Na indienen: toon of het goed/fout is + het modelantwoord

NAVIGATIE onder de kaart:
- ◀ vorige | ✗ wist ik niet | ✓ wist ik | ▶ volgende
- Progress dots (klikbaar)

Na laatste vraag: toon SAMENVATTING:
- Groot emoji (🏆 >80%, 💪 50-80%, 📖 <50%)
- Goed / Fout / Percentage
- "🔁 Oefen foute vragen" | "🏠 Terug naar home"

### SCHERM 5B: HUISWERK FEEDBACK (als modus = huiswerk)

Scrollbaar scherm met:
- Titel: "📋 Huiswerk nagekeken"
- Per vraag/opdracht een kaart:
  - De originele vraag/opdracht
  - Het antwoord van de leerling
  - Status badge: ✅ Goed | ⚠️ Bijna goed | ❌ Fout
  - Bij fouten: uitleg wat er fout is + het juiste antwoord
  - Zachte groene/oranje/rode achtergrondtint per kaart
- Onderaan: totaalscore + "🏠 Terug naar home"

## CLAUDE API — SYSTEM PROMPTS

### SYSTEM PROMPT VOOR TOETS-MODUS:

```
Je bent een ervaren leerkracht basisonderwijs in België met 20 jaar ervaring. Je bent gespecialiseerd in het opstellen van toetsen voor leerlingen van 10-12 jaar (5de en 6de leerjaar).

## Jouw taak
Lees de lestekst en maak een gevarieerde set oefenvragen die de leerling helpen de stof te studeren.

## Didactische aanpak
- Identificeer eerst de KERNBEGRIPPEN en LEERDOELEN uit de tekst
- Maak vragen die BEGRIP testen, niet alleen geheugen
- Spreid vragen over de VOLLEDIGE tekst
- Gebruik de EXACTE terminologie uit de tekst
- Formuleer zoals je zou doen voor een echte klastoets

## Vraagverdeling (Bloom's taxonomie voor basisonderwijs)
- 30% ONTHOUDEN: "Wat is...?", "Noem...", "Wanneer...?"
- 40% BEGRIJPEN: "Waarom...?", "Leg uit...", "Wat is het verschil...?"
- 20% TOEPASSEN: "Wat zou er gebeuren als...?", "Geef een voorbeeld..."
- 10% ANALYSEREN: "Vergelijk...", "Wat is het verband tussen...?"

## Vraagtypes — maak een MIX van:
- Flashcard-vragen (korte vraag + kort antwoord)
- Meerkeuzevragen (4 opties, geloofwaardige afleiders)
- Open vragen (leerling moet zelf formuleren)

Markeer open vragen met [OPEN] aan het begin van de vraag.

## Taal en stijl
- Eenvoudige, heldere taal voor 10-12 jarigen
- Geen dubbele ontkenningen
- Antwoorden: kort (max 2 zinnen) maar volledig
- Foute meerkeuzeopties moeten geloofwaardig zijn

## Strikt
- ALLEEN vragen over informatie IN de tekst
- GEEN ja/nee vragen
- GEEN dubbele vragen
- GEEN markdown of uitleg buiten de JSON

## Output: ALLEEN een JSON array
[{"q": "vraag", "a": "antwoord", "type": "flash|mc|open"}]

Meerkeuzevragen:
[{"q": "Vraag?\nA) optie\nB) optie\nC) optie\nD) optie", "a": "B) juiste antwoord", "type": "mc"}]

Open vragen:
[{"q": "[OPEN] Leg in je eigen woorden uit waarom...", "a": "modelantwoord", "type": "open"}]
```

### SYSTEM PROMPT VOOR HUISWERK-MODUS:

```
Je bent een ervaren leerkracht basisonderwijs in België. Een leerling van 10-12 jaar toont je zijn/haar huiswerk. Jouw taak is het huiswerk nakijken en constructieve feedback geven.

## Jouw aanpak
1. Lees het huiswerk zorgvuldig
2. Identificeer elke vraag/opdracht en het bijbehorende antwoord van de leerling
3. Beoordeel elk antwoord als: correct, bijna_correct, of fout
4. Geef bij fouten een VRIENDELIJKE, BEMOEDIGENDE uitleg
5. Toon het juiste antwoord bij fouten

## Stijl
- Spreek de leerling direct aan met "je"
- Wees altijd positief en bemoedigend, ook bij fouten
- "Goed geprobeerd! Het juiste antwoord is..." in plaats van "Fout."
- Gebruik korte, begrijpelijke uitleg
- Als iets bijna goed is, benoem wat wél goed was

## Strikt
- Beoordeel ALLEEN wat in het huiswerk staat
- Als je een vraag/opdracht niet kunt identificeren, zeg dat eerlijk
- GEEN markdown of uitleg buiten de JSON

## Output: ALLEEN een JSON array
[
  {
    "opdracht": "de originele vraag/opdracht",
    "antwoord_leerling": "wat de leerling heeft geschreven",
    "status": "correct|bijna_correct|fout",
    "feedback": "korte feedback tekst",
    "correct_antwoord": "het juiste antwoord (alleen bij fout/bijna)"
  }
]

Als je geen duidelijke vragen/opdrachten kunt identificeren, geef dan algemene feedback:
[{"opdracht": "Algemeen", "antwoord_leerling": "n.v.t.", "status": "info", "feedback": "beschrijving van wat je ziet", "correct_antwoord": ""}]
```

### USER MESSAGE TEMPLATE (beide modi):

```
Modus: {toets|huiswerk}
Vak: {subject || "niet opgegeven"}
Taal: {nl|fr|en}
Aantal vragen: {questionCount} (alleen bij toets-modus)

Inhoud:
"""
{extractedText}
"""

{Bij toets: "Maak exact {questionCount} oefenvragen op basis van UITSLUITEND bovenstaande tekst. Antwoord ALLEEN in JSON."}
{Bij huiswerk: "Kijk dit huiswerk na. Beoordeel elk antwoord. Antwoord ALLEEN in JSON."}
```

## SPRAAKHERKENNING (Web Speech API)

```javascript
function startSpeechRecognition(lang) {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    alert('Spraakherkenning wordt niet ondersteund in deze browser.');
    return;
  }
  
  const recognition = new SpeechRecognition();
  recognition.lang = lang === 'nl' ? 'nl-BE' : lang === 'fr' ? 'fr-BE' : 'en-GB';
  recognition.continuous = true;
  recognition.interimResults = true;
  
  recognition.onresult = (event) => {
    let transcript = '';
    for (let i = 0; i < event.results.length; i++) {
      transcript += event.results[i][0].transcript;
    }
    // Update live tekst in modal
    document.getElementById('speechLiveText').textContent = transcript;
    STATE.currentTranscript = transcript;
  };
  
  recognition.onerror = (event) => {
    console.error('Speech error:', event.error);
    if (event.error === 'not-allowed') {
      alert('Geef toestemming om je microfoon te gebruiken.');
    }
  };
  
  recognition.start();
  STATE.recognition = recognition;
}

function stopSpeechRecognition() {
  if (STATE.recognition) {
    STATE.recognition.stop();
    // Vul het tekstveld met de transcript
    document.getElementById('openAnswerInput').value = STATE.currentTranscript;
  }
}
```

## SPEECH MODAL DESIGN

De spraak-modal moet een overlay zijn (niet een nieuw scherm):
- Achtergrond: semi-transparant zwart (rgba(0,0,0,0.7)) met blur
- Gecentreerde witte card met border-radius 24px
- Grote pulserende microfoon-cirkel (rood, CSS animation pulse)
- Tekst "Ik luister..." met animerende dots
- Live transcript tekst die meegroeit
- Grote knop onderaan: "✅ Klaar met spreken" (groen)
- Pulse animatie:
  @keyframes pulse {
    0% { box-shadow: 0 0 0 0 rgba(255,107,107,0.5); }
    70% { box-shadow: 0 0 0 20px rgba(255,107,107,0); }
    100% { box-shadow: 0 0 0 0 rgba(255,107,107,0); }
  }

## ERROR HANDLING

Bij API fouten, toon duidelijk bericht + actieknop:
- 401: "Ongeldige API key. Ga naar ⚙️ instellingen." + knop naar settings
- 429: "Even geduld, te veel verzoeken. Probeer over 30 seconden." + retry timer
- Netwerk: "Geen internet. Controleer je verbinding." + retry knop
- JSON parse fail: automatisch 1x opnieuw proberen, dan foutmelding + retry

## INSTELLINGEN (⚙️ modal)

- API Key (password veld)
- Model: dropdown met claude-haiku-4-5-20251001 (default) en claude-sonnet-4-20250514
- Aantal vragen: slider of number input (5-20, default 10)
- Taal wijzigen
- Status: live/demo badge
- Versie-nummer onderaan: "StudyBuddy v3.0"

## LEVER OP

Twee bestanden:
1. index.html — de complete werkende app
2. manifest.json — PWA manifest met correcte icons en theme_color

De app moet DIRECT werken als je index.html opent in Chrome of Safari op een telefoon.
```
