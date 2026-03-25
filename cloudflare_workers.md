# Cloudflare Workers: StudyBuddy

Hier is de code voor beide Cloudflare Workers. Ze vangen je CORS-headers af voor `https://r13ns.github.io` en verbergen je API keys veilig op de server.

> [!IMPORTANT]
> Let op: Je moet de variabelen `AZURE_KEY` en `ANTHROPIC_KEY` toevoegen bij de **Settings > Variables & Secrets** van je Workers in het Cloudflare dashboard.

---

## Worker 1: `studybuddy-ocr`

Maak een nieuwe Worker aan, noem hem `studybuddy-ocr`, plak de onderstaande code in de editor en voeg de `AZURE_KEY` secret toe voordat je op Deploy klikt.

```javascript
/**
 * studybuddy-ocr Worker
 * Env bindings required:
 * AZURE_KEY (string)
 */

const corsHeaders = {
  "Access-Control-Allow-Origin": "https://r13ns.github.io",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Max-Age": "86400",
};

export default {
  async fetch(request, env) {
    // 1. CORS Preflight (OPTIONS request) afhandelen
    if (request.method === "OPTIONS") {
      let headers = new Headers(corsHeaders);
      if (request.headers.has("Access-Control-Request-Headers")) {
        headers.set("Access-Control-Allow-Headers", request.headers.get("Access-Control-Request-Headers"));
      }
      return new Response(null, { headers });
    }

    if (request.method !== "POST") {
      return new Response("Method not allowed", { status: 405, headers: corsHeaders });
    }

    try {
      const endpoint = 'https://huiswerk-ocr-vision.cognitiveservices.azure.com/computervision/imageanalysis:analyze?api-version=2024-02-01&features=read';
      
      // We nemen de binaire data (base64 of blob) rechtstreeks over van de frontend
      const fileBlob = await request.blob();
      
      // Request doorsturen naar Azure
      const azureResponse = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Ocp-Apim-Subscription-Key': env.AZURE_KEY,
          'Content-Type': 'application/octet-stream'
        },
        body: fileBlob
      });

      if (!azureResponse.ok) {
        throw new Error('Azure API Error: ' + azureResponse.statusText);
      }

      const data = await azureResponse.json();
      
      // Tekst extractie (blocks en lines concatenen)
      let extractedText = '';
      if (data.readResult) {
        if (data.readResult.content) {
          extractedText = data.readResult.content;
        } else if (data.readResult.blocks) {
          data.readResult.blocks.forEach(block => {
            block.lines.forEach(line => {
              extractedText += line.text + '\\n';
            });
            extractedText += '\\n';
          });
        }
      }

      // Geef de schone tekst direct terug aan je app
      return new Response(extractedText.trim(), {
        headers: {
          ...corsHeaders,
          "Content-Type": "text/plain;charset=UTF-8"
        }
      });
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), { status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" } });
    }
  }
};
```

---

## Worker 2: `studybuddy-claude`

Maak de tweede Worker aan onder de naam `studybuddy-claude`, plak onderstaande code, en voeg hier `ANTHROPIC_KEY` toe als encrypted secret.

```javascript
/**
 * studybuddy-claude Worker
 * Env bindings required:
 * ANTHROPIC_KEY (string)
 */

const corsHeaders = {
  "Access-Control-Allow-Origin": "https://r13ns.github.io",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Max-Age": "86400",
};

export default {
  async fetch(request, env) {
    // 1. CORS Preflight
    if (request.method === "OPTIONS") {
      let headers = new Headers(corsHeaders);
      if (request.headers.has("Access-Control-Request-Headers")) {
        headers.set("Access-Control-Allow-Headers", request.headers.get("Access-Control-Request-Headers"));
      }
      return new Response(null, { headers });
    }

    if (request.method !== "POST") {
      return new Response("Method not allowed", { status: 405, headers: corsHeaders });
    }

    try {
      // Data die je verstuurt vanuit index.html (systeem prompt + json formaat)
      const reqBody = await request.json();
      
      const anthropicResponse = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': env.ANTHROPIC_KEY,
          'anthropic-version': '2023-06-01'
        },
        body: JSON.stringify(reqBody)
      });

      const data = await anthropicResponse.json();

      return new Response(JSON.stringify(data), {
        status: anthropicResponse.status,
        headers: {
          ...corsHeaders,
          "Content-Type": "application/json"
        }
      });
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), { 
        status: 500, 
        headers: { ...corsHeaders, "Content-Type": "application/json" } 
      });
    }
  }
};
```
