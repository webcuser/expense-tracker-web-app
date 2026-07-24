# expense-tracker-web-app

App web semplice per tracciare entrate/uscite e vedere statistiche mensili

## Overview

# PRD — expense-tracker-web-app (v1.0)

Data: 24/07/2026  
Versione: 1.0

## 1. Project Overview

Expense Tracker è una web application per la gestione semplice e veloce delle finanze personali (entrate/uscite). Mira a sostituire fogli Excel o app troppo complesse con un’esperienza moderna, accessibile da qualsiasi dispositivo, che consenta di:
- registrare movimenti di denaro (entrate/uscite),
- monitorare saldo e trend mensili,
- filtrare e ricercare movimenti,
- visualizzare un riepilogo in dashboard.

MVP focalizzato su: autenticazione, CRUD movimenti, dashboard con totali mensili e ultimi movimenti, categorie predefinite e personalizzate, filtri e ricerca.

## 2. Goals & Success Metrics

Obiettivi principali:
- Ridurre l’attrito nella registrazione delle spese.
- Fornire insight chiari e immediati sullo stato finanziario mensile.
- Garantire sicurezza e affidabilità con performance elevate.

KPI (target v1):
- Tempo medio per registrare una spesa < 20 secondi.
- Tempo di caricamento dashboard < 1 secondo (p50), < 2 secondi (p95).
- Tempo di risposta API < 300 ms (p95) per endpoint principali (GET/POST /api/transactions, GET dashboard).
- Disponibilità del servizio > 99% su base mensile.
- Tasso di successo login > 98% (escluse credenziali errate).
- Error rate API (5xx) < 0,1%.

Criteri di accettazione (MVP):
- L’utente può registrarsi, verificare la mail, autenticarsi, fare logout, recuperare la password.
- L’utente può creare, modificare, eliminare movimenti.
- Il saldo si aggiorna automaticamente su dashboard.
- Filtri per categoria, tipo e periodo funzionano correttamente.
- La dashboard mostra dati coerenti con i movimenti registrati e con il fuso orario utente.
- Le API restituiscono codici HTTP e messaggi di errore adeguati su input non valido.

## 3. Target Users

- Privati: tracciare spese quotidiane e entrate.
- Studenti: gestire budget limitati, spese ricorrenti.
- Freelance: separare movimenti personali e professionali leggeri.
- Famiglie: visione condivisa e semplice delle uscite principali.

Esigenze:
- Inserimento rapido ovunque (mobile/desktop).
- Semplicità d’uso senza curva di apprendimento.
- Statistiche mensili chiare (totali, saldo).
- Filtri per categoria e periodo, ricerca per descrizione.

## 4. Core Features

### 4.1 Autenticazione e Account

- Registrazione con nome, email, password.
- Verifica email obbligatoria prima dell’accesso a feature protette.
- Login, logout (cookie-based Sanctum).
- Recupero password via email con token monouso hashato e scadenza.
- Endpoint:
  - POST /api/register
  - POST /api/login
  - POST /api/logout
  - POST /api/forgot-password
  - POST /api/reset-password
  - GET /api/email/verify/{id}/{hash}
  - POST /api/email/verification-notification
  - GET /api/user
- Provider email:
  - Dev: Mailpit
  - Prod: Amazon SES (alternabili Mailgun/Postmark)
- Sicurezza password: Argon2id; policy minime:
  - Lunghezza minima 8; raccomandato 12+
  - Combinazione caratteri raccomandata (ma non bloccante per MVP)

Flussi principali:
- Registrazione:
  1) Utente invia nome/email/password.
  2) Sistema crea account, invia email verifica.
  3) Utente clicca link verifica.
  4) Accesso abilitato alle aree protette.
- Recupero Password:
  1) Utente invia email.
  2) Sistema genera token monouso hashato, invia link.
  3) Utente imposta nuova password, token invalidato.

Rate limiting:
- Login: 5/min per IP.
- Registrazione: 3/15 min per IP.
- Recupero password: 3/15 min per email/IP.
- API autenticate: 60/min per utente.

### 4.2 Gestione Movimenti (CRUD)

Campi movimento:
- amount decimal(10,2) — importo positivo, formattazione UI: 1.250,50 €
- type enum[income, expense]
- category_id bigint — deve essere coerente con type
- description text — opzionale, max 1.000 caratteri
- transaction_date datetime — salvato in UTC
- currency string(3) — default “EUR” (per estensioni future)
- created_at timestamp (server)

Operazioni:
- Aggiungi movimento (POST /api/transactions)
- Modifica movimento (PUT /api/transactions/{id})
- Elimina movimento (DELETE /api/transactions/{id}) — hard delete per MVP
- Elenco movimenti (GET /api/transactions) con filtri/ricerca/ordinamento/paginazione.

Validazioni di business:
- amount > 0
- type obbligatorio
- category_id deve esistere e corrispondere al type della categoria
- transaction_date obbligatoria e valida (ISO 8601); salvata in UTC
- description max 1.000 char
- currency = “EUR” per MVP

Filtri e query (GET /api/transactions):
- page (default 1), limit (min 10, default 20, max 100)
- category_id
- type in [income, expense]
- date_from, date_to (inclusivi; interpretati nel timezone utente, convertiti a UTC per query)
- search (full-text semplice su description; ILIKE %term% su PostgreSQL)
- sort in [transaction_date, amount, created_at, category]; direction [asc, desc] — default sort=transaction_date, direction=desc
- Risposta con paginator Laravel (data, meta, links)

Esempio body POST:
{
  "amount": 45.90,
  "type": "expense",
  "category_id": 3,
  "description": "Spesa supermercato",
  "transaction_date": "2026-07-24T18:30:00Z"
}

Codici di risposta:
- 200/201 OK/Created
- 400/422 Validation error
- 401 Unauthorized (non autenticato o email non verificata)
- 403 Forbidden (risorsa non appartenente all’utente)
- 404 Not found
- 429 Too Many Requests
- 500 Server error

Formato errori suggerito:
{
  "message": "Validation failed",
  "errors": { "field": ["error detail"] }
}

### 4.3 Dashboard

Contenuti:
- Saldo attuale (somma entrate - uscite su tutti i movimenti dell’utente).
- Totale entrate del mese (confine mese nel timezone utente).
- Totale uscite del mese (stesso criterio).
- Ultimi 10 movimenti (ordinati per transaction_date desc).
- Valuta: EUR, importi formattati lato frontend.

Note su fuso orario:
- Salvataggio date in UTC.
- Calcolo aggregazioni mensili nel timezone dell’utente (default Europe/Rome).

Performance:
- Query ottimizzate con indici su (user_id, transaction_date), (user_id, type), (user_id, category_id).
- Possibile caching di aggregazioni mensili a breve TTL (es. 30-60s) se necessario.

### 4.4 Categorie (Globali e Personalizzate)

Struttura:
- Categorie globali: user_id NULL, seed iniziale (IT), non modificabili/eliminabili dagli utenti.
- Categorie personalizzate: user_id = id utente; l’utente può creare, rinominare ed eliminare.

Regole:
- type enum[income, expense] vincola l’uso sui movimenti omogenei.
- Eliminazione categorie personalizzate: soft delete (flag deleted_at).
  - Le categorie associate a movimenti non vengono rimosse fisicamente.
  - I movimenti esistenti mantengono il riferimento; la categoria non appare più nelle liste selezionabili se soft-deleted.
- MVP seed in italiano (esempi): Entrate: Stipendio, Regalo, Rimborso, Altro. Uscite: Alimentari, Trasporti, Affitto, Bollette, Tempo libero, Altro.

Endpoint categorie (MVP minimo consigliato):
- GET /api/categories — lista globali + personali attive
- POST /api/categories — crea categoria personale
- PUT /api/categories/{id} — aggiorna nome (solo personali)
- DELETE /api/categories/{id} — soft delete (solo personali)

Validazioni:
- name obbligatorio, univocità per utente+type (case-insensitive).
- type obbligatorio.

### 4.5 Ricerca e Filtri

- Ricerca testuale su description.
- Filtri per categoria, tipo, e periodo (date_from/date_to).
- Ordinamento per data transazione, importo, data creazione, categoria.
- Paginazione conforme ai limiti.

### 4.6 Localizzazione e Formattazione

- Tutte le API restituiscono valori non formattati (es. amount numerico).
- UI formatta:
  - Importi: “1.250,50 €”
  - Date: “24/07/2026”
  - Date/ora: “24/07/2026 18:30”
- Fuso orario per utente configurabile (default Europe/Rome); persistito a livello profilo utente.

## 5. Technical Architecture

### 5.1 Stack Tecnologico

- Backend: Laravel 12 (PHP 8.4), Laravel Sanctum, PostgreSQL
- Frontend: Next.js + React, Tailwind CSS
- Container: Docker (servizi: app backend, db, mailpit in dev, nginx opzionale)
- Autenticazione: Sanctum cookie-based per SPA (CSRF)
- Email: Mailpit (dev), Amazon SES (prod)
- Deploy: container-based; reverse proxy Nginx; HTTPS obbligatorio in prod

### 5.2 Componenti Chiave

- API Laravel (REST) con middleware auth:sanctum, verifiche email, rate limiting.
- Moduli:
  - Auth & Security (registrazione, login, password reset, email verification, CSRF, CORS)
  - Transactions (CRUD, query builder con filtri e sorting, policy di autorizzazione per user_id)
  - Categories (globali seed, personali CRUD con soft delete)
  - Dashboard (aggregazioni ottimizzate per mese corrente nel timezone utente)
- Frontend SPA:
  - Pagine/route: /login, /register, /forgot-password, /reset-password, /verify-email, /dashboard, /transactions, /categories
  - Stato: gestione sessione con cookie HttpOnly; fetch con credenziali e CSRF token
  - UI: form validazione lato client, liste con infinite scroll/paginazione, componenti tables/filters

### 5.3 Modello Dati (PostgreSQL)

users
- id bigint PK
- name varchar(255) NOT NULL
- email varchar(255) UNIQUE NOT NULL
- email_verified_at timestamp nullable
- password varchar(255) NOT NULL
- timezone varchar(64) DEFAULT ‘Europe/Rome’
- created_at, updated_at timestamps

categories
- id bigint PK
- user_id bigint nullable FK -> users(id) ON DELETE CASCADE (per personali)
- name varchar(255) NOT NULL
- type varchar(16) CHECK (type IN ('income','expense')) NOT NULL
- deleted_at timestamp nullable (soft delete per personali)
- created_at, updated_at timestamps
- Indici: (user_id, type, lower(name)) unique parziale per user_id non null; (type, lower(name)) unique per globali

transactions
- id bigint PK
- user_id bigint NOT NULL FK -> users(id) ON DELETE CASCADE
- category_id bigint NOT NULL FK -> categories(id)
- amount decimal(10,2) NOT NULL CHECK (amount > 0)
- type varchar(16) CHECK (type IN ('income','expense')) NOT NULL
- currency char(3) DEFAULT 'EUR' NOT NULL
- description text nullable
- transaction_date timestamp NOT NULL (UTC)
- created_at, updated_at timestamps
- Indici: (user_id, transaction_date desc), (user_id, type), (user_id, category_id), (user_id, created_at desc)
- Vincolo applicativo: type deve essere coerente con categories.type

Note:
- Salvataggio date in UTC; conversione lato applicazione per query e presentazione.
- Preparazione per estensione multi-valuta tramite campo currency.

### 5.4 API Design

Autenticazione (cookie-based):
- Protezione CSRF: chiamare /sanctum/csrf-cookie prima delle POST mutate.
- Tutte le rotte autenticate protette da auth:sanctum e verifica email.

Headers:
- Content-Type: application/json
- X-CSRF-TOKEN per richieste mutate (da cookie XSRF-TOKEN, gestito da frontend)

CORS:
- Allowed origins: https://app.expensetracker.com, https://www.expensetracker.com, http://localhost:3000 (sviluppo)
- Credentials: true
- Methods: GET, POST, PUT, PATCH, DELETE
- Headers: Content-Type, X-CSRF-TOKEN, Accept, Authorization

Logging & Error Handling:
- Formato errori JSON standardizzato
- Logging applicativo (Laravel) con livelli info/warning/error
- Tracciamento request-id per diagnosi (middleware)

### 5.5 Sicurezza

- Password hashate con Argon2id
- HTTPS obbligatorio in prod; cookie Secure e HttpOnly
- CSRF protection attiva per SPA
- Input validation server-side per tutti gli endpoint
- Rate limiting come definito
- Protezione da Enumeration:
  - Messaggi generici su login/reset (“Se l’email esiste, riceverai un link”)
- Content Security Policy consigliata (blocco inline script; hash/nonce)
- Backup giornalieri del database con cifratura at-rest
- Gestione segreti tramite variabili d’ambiente (no hardcoded)

### 5.6 Performance e Scalabilità

- Query indicizzate per transazioni e dashboard
- Paginazione server-side
- Caching leggero per aggregazioni mensili (opzionale)
- Orchestrazione Docker per scaling orizzontale del backend stateless
- Connection pooling DB (PgBouncer consigliato in prod)
- CDN per asset statici frontend (Next.js build ottimizzata)

## 6. Non-Functional Requirements

- Responsività UI (mobile-first con Tailwind)
- Compatibilità browser moderni (ultime 2 versioni Chrome, Firefox, Safari, Edge)
- Tempo di risposta API < 300 ms (p95) per endpoint principali
- Dashboard load < 1 s (p50)
- Uptime > 99%
- Sicurezza:
  - Argon2id, CSRF, HTTPS, policy CORS restrittiva
  - Validazione input, sanitizzazione output
- Privacy e Protezione dati:
  - Minimizzazione dati personali
  - Log senza dati sensibili
- Osservabilità:
  - Healthcheck endpoint
  - Metriche base (request rate, latency, error rate)
- Backup:
  - Daily full backup DB; retention 7-30 giorni (definizione in ops)
  - Test periodici di restore
- Accessibilità:
  - Contrasti e semantics ARIA base per elementi di form e tabelle
- Internazionalizzazione:
  - MVP in italiano; architettura predisposta a i18n (file di traduzione)

## 7. Out of Scope (v1)

- App mobile nativa (Android/iOS)
- Integrazioni di pagamento o connessioni bancarie (PSD2/Open Banking)
- Multi-valuta operativa (solo EUR per MVP)
- Token-based API per client esterni (Sanctum PAT) — valutazione futura
- Condivisione account/famiglia multi-utente con ruoli
- Grafici avanzati, budget, esportazione CSV, obiettivi di risparmio, notifiche — previsti in roadmap fasi successive
- SSO (Google/Apple)
- Eliminazione soft dei movimenti (v1 usa hard delete)
- Localizzazione multilingua dei seed categorie (MVP solo IT)

## 8. Open Questions

- Soft delete per movimenti: mantenere storico e possibilità di ripristino? Imp