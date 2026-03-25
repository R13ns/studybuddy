# StudyBuddy v3 — Follow-up prompt voor je agent

Plak dit als vervolg-bericht in dezelfde chat met je agent.

---

```
Goed plan. Voer het uit met deze aanvullingen en correcties:

## 1. KRITIEKE FIXES

### Model ID
Gebruik OVERAL: claude-haiku-4-5-20251001 als default model.
NIET claude-3-5-haiku-20241022 (die is verlopen).
Sonnet optie: claude-sonnet-4-20250514

### API Headers
Elke fetch naar api.anthropic.com MOET deze headers hebben:
```javascript
headers: {
  'Content-Type': 'application/json',
  'x-api-key': STATE.apiKey,
  'anthropic-version': '2023-06-01',
  'anthropic-dangerous-direct-browser-access': 'true'
}
```
De header 'anthropic-dangerous-direct-browser-access' is VERPLICHT voor browser-side API calls. Zonder dit krijg je een CORS error.

## 2. MULTI-FOTO FLOW

De camera input moet zo werken:
- Eerste klik: open <input type="file" accept="image/*" capture="environment">
- Na foto: toon thumbnail in een grid (3 per rij)
- Knop "📸 Nog een foto" → opent dezelfde input opnieuw
- Knop "✅ Klaar met foto's" → ga naar verwerking
- Elke thumbnail heeft een ✕ verwijder-knop
- Toon teller: "3 foto's genomen"

Bij verwerking van meerdere foto's:
- Loop sequentieel door elke foto
- OCR elke foto apart met Azure Computer Vision Read API
- Combineer alle tekst met een dubbele newline ertussen
- Toon voortgang: "Foto 1/3 wordt gelezen..."

## 3. SCREEN TRANSITIONS

Gebruik deze CSS voor native-achtige slide transitions:

```css
.screen {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  transform: translateX(100%);
  transition: transform 300ms ease;
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}
.screen.active {
  transform: translateX(0);
}
.screen.exit-left {
  transform: translateX(-100%);
}
```

Navigatie functie:
```javascript
function navigateTo(screenId, direction = 'forward') {
  const current = document.querySelector('.screen.active');
  const next = document.getElementById(screenId);
  
  if (direction === 'forward') {
    next.style.transform = 'translateX(100%)';
    requestAnimationFrame(() => {
      current.classList.add('exit-left');
      current.classList.remove('active');
      next.classList.add('active');
      next.style.transform = '';
    });
  } else {
    next.style.transform = 'translateX(-100%)';
    requestAnimationFrame(() => {
      current.style.transform = 'translateX(100%)';
      current.classList.remove('active');
      next.classList.add('active');
      next.style.transform = '';
    });
  }
  
  // Cleanup na transition
  setTimeout(() => {
    current.classList.remove('exit-left');
  }, 350);
}
```

## 4. WEB SPEECH API — COMPLETE IMPLEMENTATIE

```javascript
function openSpeechModal() {
  const modal = document.getElementById('speechModal');
  modal.classList.add('active');
  
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    document.getElementById('speechLiveText').textContent = 
      'Spraakherkenning niet beschikbaar in deze browser. Gebruik Chrome.';
    return;
  }
  
  const recognition = new SpeechRecognition();
  const langMap = { nl: 'nl-BE', fr: 'fr-BE', en: 'en-GB' };
  recognition.lang = langMap[STATE.lang] || 'nl-BE';
  recognition.continuous = true;
  recognition.interimResults = true;
  
  let finalTranscript = '';
  
  recognition.onresult = (event) => {
    let interim = '';
    for (let i = event.resultIndex; i < event.results.length; i++) {
      if (event.results[i].isFinal) {
        finalTranscript += event.results[i][0].transcript + ' ';
      } else {
        interim += event.results[i][0].transcript;
      }
    }
    document.getElementById('speechLiveText').textContent = finalTranscript + interim;
  };
  
  recognition.onerror = (event) => {
    if (event.error === 'not-allowed') {
      document.getElementById('speechLiveText').textContent = 
        'Geef toestemming voor je microfoon in je browser-instellingen.';
    }
  };
  
  recognition.start();
  STATE.recognition = recognition;
  STATE.finalTranscript = '';
}

function closeSpeechModal() {
  if (STATE.recognition) {
    STATE.recognition.stop();
  }
  const transcript = document.getElementById('speechLiveText').textContent.trim();
  document.getElementById('openAnswerInput').value = transcript;
  document.getElementById('speechModal').classList.remove('active');
}
```

## 5. SPEECH MODAL HTML + CSS

```html
<div id="speechModal" class="speech-overlay">
  <div class="speech-card">
    <div class="mic-pulse">🎤</div>
    <p class="speech-status">Ik luister<span class="dots">...</span></p>
    <div class="speech-transcript" id="speechLiveText"></div>
    <button class="speech-done-btn" onclick="closeSpeechModal()">
      ✅ Klaar met spreken
    </button>
  </div>
</div>
```

```css
.speech-overlay {
  display: none;
  position: fixed; inset: 0;
  background: rgba(0,0,0,0.7);
  backdrop-filter: blur(10px);
  z-index: 300;
  align-items: center; justify-content: center;
}
.speech-overlay.active { display: flex; }

.speech-card {
  background: white;
  border-radius: 24px;
  padding: 40px 28px;
  text-align: center;
  width: 90%; max-width: 360px;
  animation: slideUp 0.3s ease;
}

.mic-pulse {
  width: 80px; height: 80px;
  border-radius: 50%;
  background: #FF6B6B;
  display: flex; align-items: center; justify-content: center;
  font-size: 36px;
  margin: 0 auto 20px;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { box-shadow: 0 0 0 0 rgba(255,107,107,0.5); }
  70% { box-shadow: 0 0 0 25px rgba(255,107,107,0); }
  100% { box-shadow: 0 0 0 0 rgba(255,107,107,0); }
}

.speech-status {
  font-size: 18px; font-weight: 700;
  color: #636E72; margin-bottom: 16px;
}

.dots { animation: blink 1.5s infinite; }
@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

.speech-transcript {
  min-height: 60px;
  padding: 16px;
  background: #F0F0FF;
  border-radius: 12px;
  font-size: 15px;
  line-height: 1.5;
  color: #2D3436;
  margin-bottom: 20px;
  text-align: left;
  max-height: 150px;
  overflow-y: auto;
}

.speech-done-btn {
  width: 100%; padding: 16px;
  background: linear-gradient(135deg, #00B894, #00CEC9);
  color: white; border: none;
  border-radius: 12px;
  font-family: inherit;
  font-size: 16px; font-weight: 800;
  cursor: pointer;
}
```

## 6. OPEN VRAAG WEERGAVE IN FLASHCARD

Als een vraag type "open" is, toon dan NIET de standaard flashcard maar:

```html
<div class="open-question-card">
  <div class="oq-label">Open vraag</div>
  <p class="oq-question">{vraag}</p>
  <div class="oq-answer-area">
    <textarea id="openAnswerInput" placeholder="Typ je antwoord hier..."></textarea>
    <button class="mic-btn" onclick="openSpeechModal()">🎤</button>
  </div>
  <button class="check-answer-btn" onclick="checkOpenAnswer()">
    ✅ Controleer antwoord
  </button>
</div>
```

Na indienen: toon het modelantwoord + een beoordeling (goed/gedeeltelijk/fout).
Voor de beoordeling: vergelijk gewoon tekstueel, geen API call nodig. 
Zoek of de kernwoorden uit het modelantwoord in het gegeven antwoord voorkomen.

## 7. HUISWERK FEEDBACK KAARTEN

Elke kaart moet er zo uitzien:

```html
<div class="hw-card hw-{status}">
  <div class="hw-badge">{✅|⚠️|❌} {Goed|Bijna goed|Fout}</div>
  <div class="hw-opdracht">
    <strong>Opdracht:</strong> {opdracht}
  </div>
  <div class="hw-antwoord">
    <strong>Jouw antwoord:</strong> {antwoord_leerling}
  </div>
  <div class="hw-feedback">{feedback}</div>
  {als fout/bijna: <div class="hw-correct">💡 Juiste antwoord: {correct_antwoord}</div>}
</div>
```

CSS per status:
- correct: achtergrond #F0FFF4, border-left 4px solid #00B894
- bijna_correct: achtergrond #FFFBF0, border-left 4px solid #FDCB6E
- fout: achtergrond #FFF0F0, border-left 4px solid #FF6B6B

## 8. BEWAAR ALLE BESTAANDE LOGICA

BELANGRIJK: Verwijder NIETS van de bestaande werkende code:
- Claude API calls (quiz + homework)
- Azure Computer Vision OCR
- Consensus comparison (Needleman-Wunsch)
- PDF.js integratie
- Word-level diff builder
- Demo mode
- Error handling

Wrap de bestaande logica in de nieuwe scherm-flow. Refactor de UI, niet de business logic.

## 9. iOS SAFE AREAS

Voeg toe aan de CSS:
```css
body {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
}
```

En op fixed bottom bars:
```css
.bottom-bar {
  padding-bottom: calc(16px + env(safe-area-inset-bottom));
}
```

## 10. TEST CHECKLIST

Na de build, verifieer:
- [ ] Taalkeuze verschijnt bij eerste bezoek
- [ ] Taalkeuze wordt overgeslagen na eerste keer (localStorage)
- [ ] Slide transitions werken vloeiend
- [ ] Camera opent en meerdere foto's kunnen worden genomen
- [ ] Thumbnails tonen met verwijder-knop
- [ ] OCR werkt op foto's (Azure Computer Vision)
- [ ] Tekst review scherm toont herkende tekst
- [ ] API call gaat uit met correcte headers en model
- [ ] Flashcards flippen met 3D animatie
- [ ] Open vragen tonen tekstveld + microfoon knop
- [ ] Spraakherkenning werkt (alleen Chrome)
- [ ] Huiswerk feedback toont gekleurde kaarten
- [ ] Settings modal opent en slaat API key op
- [ ] Demo modus werkt zonder API key
- [ ] Terug-navigatie werkt op elk scherm
- [ ] Geen console errors

Bouw het nu. Lever index.html + manifest.json op.
```
