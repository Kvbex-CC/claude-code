# Claude Code Prompt — StockTracker Web App

Gebruik dit document als volledige instructieset voor Claude Code om de StockTracker web app te bouwen.

---

## 🎯 Projectoverzicht

Bouw een productie-klare web app waarmee gebruikers aandelen en ETF's kunnen opvolgen via watchlists, koersgrafieken kunnen bekijken, en prijsalerts kunnen instellen. Gebruikers loggen in via Google. Alle data wordt per gebruiker opgeslagen in Firestore.

---

## 🏗️ Architectuur

### Overzicht

```
┌─────────────────────────────────────────────┐
│              Browser (React SPA)             │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │Watchlist │  │ Charts   │  │  Alerts   │  │
│  │  Module  │  │  Module  │  │  Module   │  │
│  └──────────┘  └──────────┘  └───────────┘  │
│         │            │             │         │
│         └────────────┴─────────────┘         │
│                      │                       │
│              ┌───────────────┐               │
│              │  React Query  │               │
│              │  (caching)    │               │
│              └───────────────┘               │
└──────────────────────┬──────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────▼────┐   ┌─────▼────┐  ┌─────▼──────┐
   │Firebase │   │ Netlify  │  │   EODHD    │
   │  Auth + │   │Functions │  │    API     │
   │Firestore│   │(scheduled│  │  (koersen) │
   └─────────┘   │  alerts) │  └────────────┘
                 └──────────┘
```

### Waarom deze keuzes

- **React + Vite**: snelle builds, moderne tooling, ideaal voor Netlify static deploy
- **Firebase Auth**: native Google login, geen eigen auth backend nodig
- **Firestore**: NoSQL document store, real-time updates, gratis tier ruim voldoende
- **Netlify Functions**: serverless functions voor alert-checks (cron) en het proxyen van EODHD API calls zodat de API key NOOIT in de browser terechtkomt
- **React Query (TanStack Query)**: intelligente caching van koersdata, voorkomt onnodige API calls
- **Lightweight Charts (TradingView)**: professionele financiële grafieken, performant

---

## 🛠️ Technische Stack

| Onderdeel | Technologie | Versie |
|---|---|---|
| Frontend framework | React + Vite | React 18, Vite 5 |
| Styling | Tailwind CSS | v3 |
| Auth | Firebase Authentication | v10 |
| Database | Cloud Firestore | v10 |
| Grafieken | Lightweight Charts | v4 |
| Data fetching | TanStack Query | v5 |
| Routing | React Router | v6 |
| Icons | Lucide React | latest |
| Hosting | Netlify (static) | — |
| Serverless | Netlify Functions | — |
| E-mail | Resend (of SendGrid) | — |

---

## 📁 Projectstructuur

```
stocktracker/
├── public/
│   └── favicon.svg
├── src/
│   ├── components/
│   │   ├── auth/
│   │   │   └── LoginPage.jsx
│   │   ├── layout/
│   │   │   ├── AppShell.jsx
│   │   │   ├── Sidebar.jsx
│   │   │   └── TopBar.jsx
│   │   ├── watchlist/
│   │   │   ├── WatchlistPanel.jsx
│   │   │   ├── WatchlistItem.jsx
│   │   │   ├── AddSymbolModal.jsx
│   │   │   └── WatchlistManager.jsx
│   │   ├── charts/
│   │   │   ├── PriceChart.jsx
│   │   │   └── ComparisonChart.jsx
│   │   ├── alerts/
│   │   │   ├── AlertsPanel.jsx
│   │   │   ├── AlertForm.jsx
│   │   │   └── AlertItem.jsx
│   │   └── settings/
│   │       └── SettingsPanel.jsx
│   ├── hooks/
│   │   ├── useAuth.js
│   │   ├── useWatchlists.js
│   │   ├── useQuote.js
│   │   ├── useHistoricalData.js
│   │   └── useAlerts.js
│   ├── services/
│   │   ├── firebase.js          # Firebase initialisatie
│   │   ├── firestore.js         # Firestore CRUD helpers
│   │   └── eodhd.js             # API calls via Netlify proxy
│   ├── store/
│   │   └── userSettingsStore.js # Zustand store voor settings
│   ├── utils/
│   │   ├── formatters.js        # Valuta, %, datum formatting
│   │   └── constants.js
│   ├── App.jsx
│   └── main.jsx
├── netlify/
│   └── functions/
│       ├── eodhd-proxy.js       # Proxiet EODHD calls, injecteert API key
│       └── check-alerts.js      # Scheduled function: controleert alerts
├── .env.example
├── .env                         # NOOIT committen
├── netlify.toml
├── tailwind.config.js
├── vite.config.js
└── package.json
```

---

## 🔐 Beveiliging — Kritieke Vereisten

### 1. API Keys beschermen

**De EODHD API key mag NOOIT in de browser-bundle terechtkomen.**

Implementeer een Netlify Function als proxy:

```javascript
// netlify/functions/eodhd-proxy.js
import { getAuth } from 'firebase-admin/auth';

export async function handler(event) {
  // 1. Controleer of de gebruiker ingelogd is via Firebase ID token
  const authHeader = event.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return { statusCode: 401, body: 'Unauthorized' };
  }

  try {
    const idToken = authHeader.split('Bearer ')[1];
    await getAuth().verifyIdToken(idToken); // Gooit error als ongeldig
  } catch {
    return { statusCode: 401, body: 'Invalid token' };
  }

  // 2. Voer de EODHD call uit met de server-side API key
  const { endpoint, params } = JSON.parse(event.body);
  const url = `https://eodhd.com/api/${endpoint}?api_token=${process.env.EODHD_API_KEY}&${params}&fmt=json`;

  const response = await fetch(url);
  const data = await response.json();

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  };
}
```

### 2. Firestore Security Rules

Stel deze rules in via de Firebase Console. Elke gebruiker mag ENKEL zijn eigen data lezen/schrijven:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Gebruikersinstellingen
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Watchlists
    match /users/{userId}/watchlists/{watchlistId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Symbolen binnen watchlists
    match /users/{userId}/watchlists/{watchlistId}/symbols/{symbolId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Alerts
    match /users/{userId}/alerts/{alertId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Alles wat niet expliciet is toegestaan: verboden
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

### 3. Environment Variables

Maak een `.env.example` aan als template (commit dit WEL):

```bash
# Firebase (public — worden gebundeld in de frontend)
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=

# Server-side only (NOOIT VITE_ prefix — komen niet in de browser)
EODHD_API_KEY=
FIREBASE_SERVICE_ACCOUNT_JSON=
RESEND_API_KEY=
```

Voeg `.env` toe aan `.gitignore`. Stel de server-side variabelen in via het Netlify dashboard onder Site Settings → Environment Variables.

### 4. Rate Limiting op de Proxy

Voeg rate limiting toe aan de `eodhd-proxy` function om misbruik te voorkomen:

```javascript
// Simpele in-memory rate limiter per gebruiker (per function instance)
const rateLimitMap = new Map();
const LIMIT = 30; // max requests
const WINDOW_MS = 60_000; // per minuut

function isRateLimited(uid) {
  const now = Date.now();
  const entry = rateLimitMap.get(uid) || { count: 0, start: now };
  if (now - entry.start > WINDOW_MS) {
    rateLimitMap.set(uid, { count: 1, start: now });
    return false;
  }
  if (entry.count >= LIMIT) return true;
  entry.count++;
  return false;
}
```

### 5. Content Security Policy

Voeg een CSP header toe via `netlify.toml`:

```toml
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
    Content-Security-Policy = """
      default-src 'self';
      connect-src 'self' https://*.firebaseio.com https://*.googleapis.com;
      script-src 'self' 'unsafe-inline';
      style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
      font-src 'self' https://fonts.gstatic.com;
    """
```

---

## 🗄️ Firestore Datamodel

```
users/
  {uid}/
    settings:
      currency: "EUR"
      timezone: "Europe/Brussels"
      defaultChartRange: "1M"
      theme: "dark"
      dashboardLayout: "default"

    watchlists/
      {watchlistId}/
        name: "Tech aandelen"
        createdAt: timestamp
        order: 0

        symbols/
          {symbolId}/
            ticker: "AAPL"
            exchange: "US"
            name: "Apple Inc."
            addedAt: timestamp

    alerts/
      {alertId}/
        ticker: "AAPL"
        type: "price" | "percent_change"
        # Voor price alerts:
        condition: "above" | "below"
        targetPrice: 170.00
        # Voor % change alerts:
        direction: "up" | "down"
        percentThreshold: 5.0
        periodDays: 10        # 1, 3, 5, of 10 dagen
        # Gemeenschappelijk:
        active: true
        triggered: false
        lastCheckedAt: timestamp
        createdAt: timestamp
```

---

## 📋 Feature Specificaties

### Fase 1 — MVP

#### Google Login
- Login pagina met "Inloggen met Google" knop
- Na login doorsturen naar dashboard
- Uitloggen knop in de navigatie
- Auth state bewaken via `onAuthStateChanged`; niet-ingelogde gebruikers redirecten naar `/login`

#### Watchlist Beheer
- Standaard watchlist aanmaken bij eerste login ("Mijn watchlist")
- Meerdere watchlists aanmaken, hernoemen en verwijderen
- Symbolen zoeken via EODHD search endpoint
- Symbool toevoegen aan een watchlist
- Symbool verwijderen uit een watchlist
- Watchlist items tonen met: ticker, bedrijfsnaam, huidige koers, dagwijziging (absoluut + %)
- Sorteren op naam, koers, dagwijziging

#### Koersgrafiek
- Klik op een symbool → grafiek openen
- Tijdshorizon selecteren: 1W, 1M, 3M, 6M, 12M
- Lightweight Charts candlestick of lijndiagram
- Grafiek toont OHLCV data van EODHD

#### Basisinstellingen
- Valuta (EUR, USD, GBP)
- Tijdzone
- Standaard tijdshorizon voor grafieken
- Thema (licht/donker)

### Fase 2 — Alerts

#### Prijsalerts
- Alert aanmaken voor een ticker
- Instellen: boven of onder een doelprijs
- Alert activeren/deactiveren
- Alert verwijderen

#### % Wijziging Alerts
- Alert aanmaken voor een ticker
- Instellen: richting (stijging/daling), drempelwaarde (%), tijdsvenster (1, 3, 5, 10 dagen)
- Alert activeren/deactiveren
- Alert verwijderen

#### Alert Checking (Netlify Scheduled Function)
```javascript
// netlify/functions/check-alerts.js
// Wordt elke 15 minuten uitgevoerd via netlify.toml schedule

export async function handler() {
  // 1. Haal alle actieve alerts op uit Firestore (via Firebase Admin SDK)
  // 2. Haal huidige koersen op via EODHD
  // 3. Controleer elke alert tegen de koers
  // 4. Bij trigger: stuur e-mail via Resend, markeer alert als triggered
}
```

In `netlify.toml`:
```toml
[functions."check-alerts"]
  schedule = "*/15 * * * *"
```

#### E-mail Notificatie
- Gebruik Resend (simpele API, gratis tier 3000 mails/maand)
- E-mail bevat: ticker, alert type, huidige koers, ingestelde drempel

### Fase 3 — Aandelenvergelijking

#### Vergelijkingsgrafiek
- Selecteer 2 tot 5 symbolen
- Genormaliseerde lijndiagram (startpunt = 100%)
- Zelfde tijdshorizon selectie als enkelvoudige grafiek

---

## 🚀 Opzet & Deployment

### Stap 1: Firebase Project aanmaken
1. Ga naar [console.firebase.google.com](https://console.firebase.google.com)
2. Nieuw project aanmaken
3. Authentication → inschakelen → Google provider toevoegen
4. Firestore → aanmaken in productie-modus
5. Firestore Security Rules instellen (zie boven)
6. Project settings → jouw web app config kopiëren naar `.env`

### Stap 2: Project initialiseren
```bash
npm create vite@latest stocktracker -- --template react
cd stocktracker
npm install firebase @tanstack/react-query react-router-dom \
  lightweight-charts lucide-react zustand tailwindcss \
  autoprefixer postcss
npx tailwindcss init -p
```

### Stap 3: Netlify koppelen
```bash
npm install -g netlify-cli
netlify login
netlify init
```

### Stap 4: Environment variables instellen
- Stel alle server-side variabelen in via Netlify Dashboard → Site Settings → Environment Variables
- Stel Firebase variabelen ook in (nodig voor build)

### Stap 5: Deployen
```bash
netlify deploy --prod
```

---

## ⚙️ netlify.toml (volledig)

```toml
[build]
  command = "npm run build"
  publish = "dist"
  functions = "netlify/functions"

[dev]
  command = "npm run dev"
  port = 5173

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[functions."check-alerts"]
  schedule = "*/15 * * * *"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

---

## 🎨 Design Richtlijnen

- **Thema**: professioneel financieel dashboard, donker thema als standaard
- **Kleurenpalet**: donkergrijs achtergrond (#0f1117), subtiele groene accenten voor winst, rood voor verlies
- **Typografie**: gebruik `DM Mono` voor koersen en getallen, `Sora` voor UI tekst
- **Grafieken**: donkere achtergrond, groene/rode kaarsen, minimale gridlines
- **Componenten**: strakke cards met subtiele borders, geen zware shadows
- **Responsiveness**: werkt op desktop en tablet, mobile is secundair

---

## 📌 Aandachtspunten voor Claude Code

1. **Begin met Fase 1** en commit elke feature apart
2. **Test Firestore rules** voor je verder gaat met authenticatie-gevoelige features
3. **Proxy function eerst bouwen** voor je EODHD calls doet — nooit rechtstreeks vanuit de browser
4. **React Query gebruiken** voor alle API calls met `staleTime: 60_000` (koersen zijn 1 min geldig)
5. **Error boundaries** toevoegen rond de grafiek componenten
6. **Loading states** altijd implementeren — EODHD kan traag zijn
7. **Firestore listeners** (`onSnapshot`) gebruiken voor watchlist updates in real-time
8. **Firebase Admin SDK** enkel in Netlify Functions gebruiken, nooit in de frontend
