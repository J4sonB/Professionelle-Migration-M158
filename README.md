# 📖 Projektdokumentation: Docker Shop

Diese Dokumentation beschreibt die Architektur und alle Schritte, die unternommen wurden, um das Projekt von einer statischen Nginx/PostgreSQL-Seite zu einer dynamischen, modernen **Full-Stack E-Commerce-Anwendung** umzubauen.

---

## 🏛️ 1. Architektur-Übersicht

Das Projekt basiert auf einer **Microservices-Architektur**, die vollständig über `docker-compose` orchestriert wird. Es besteht aus vier Containern, die über ein gemeinsames Docker-Netzwerk (`app_network`) miteinander kommunizieren:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Docker Network: app_network                                        │
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   Nginx       │───▶│   Backend    │───▶│   MySQL      │          │
│  │  (Port 80)    │    │  (Port 3000) │    │  (Port 3306) │          │
│  │  Reverse Proxy│    │  Node.js API │    │  Datenbank   │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│         ▲                                                           │
│  ┌──────────────┐                                                   │
│  │  Cloudflare   │                                                   │
│  │  Tunnel       │ ── Öffentliche URL ──▶ Internet                  │
│  └──────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Container-Übersicht

| Container | Image | Interner Port | Externer Port | Aufgabe |
|-----------|-------|---------------|---------------|---------|
| `project_nginx` | `nginx:1.27-alpine` | 80 | 80 | Webserver & Reverse Proxy |
| `project_backend` | Custom (Node.js) | 3000 | – (nur intern) | REST-API & Geschäftslogik |
| `project_db` | `mysql:8.0` | 3306 | 3307 | Persistente Datenspeicherung |
| `project_tunnel` | `cloudflare/cloudflared:latest` | – | – | Öffentlicher Internetzugang |

---

## 📂 2. Projektstruktur

```
MyProjectDockerM169/
├── docker-compose.yml          # Orchestrierung aller 4 Container
├── .env                        # Umgebungsvariablen (DB-Zugangsdaten)
├── .gitignore                  # Dateien, die Git ignoriert
├── README.md                   # Diese Dokumentation
│
├── frontend/                   # Frontend (wird von Nginx ausgeliefert)
│   ├── Dockerfile              # Basis-Image: nginx:1.27-alpine
│   └── index.html              # Komplette Single-Page Application (SPA)
│
├── backend/                    # Backend API-Server
│   ├── Dockerfile              # Basis-Image: node:20-alpine
│   ├── package.json            # Node.js-Abhängigkeiten
│   └── src/
│       └── index.js            # Express-Server mit allen API-Endpunkten
│
├── db/                         # Datenbank-Initialisierung
│   └── init.sql                # Tabellen-Erstellung & Demo-Daten
│
└── nginx/                      # Nginx-Konfiguration
    └── default.conf            # Reverse-Proxy-Regeln für /api/
```

---

## 💾 3. Die Datenbank (MySQL 8.0)

### Wechsel von PostgreSQL zu MySQL
Ursprünglich war das Projekt für PostgreSQL konfiguriert. Wir haben dies vollständig auf MySQL umstrukturiert:
- **Port-Konflikt gelöst:** Da auf dem Mac bereits ein lokaler Dienst auf Port `3306` lief, haben wir den MySQL-Port nach aussen auf `3307` verlegt (`3307:3306` in `docker-compose.yml`).
- **Initialisierung:** Die Datei `db/init.sql` wurde mit MySQL-Syntax ausgestattet (z.B. `AUTO_INCREMENT` statt `SERIAL`, `INSERT IGNORE` statt `ON CONFLICT`).

### Persistenz
Die Datenbank nutzt ein **Docker Volume** (`db_data`), das unter `/var/lib/mysql` gemountet wird. Dadurch bleiben alle Daten auch nach einem `docker-compose down` erhalten. Nur ein `docker-compose down -v` löscht das Volume und setzt die Datenbank komplett zurück.

### Umgebungsvariablen (`.env`)
Die Zugangsdaten werden in der `.env`-Datei gespeichert und per `${VARIABLE}` in die `docker-compose.yml` injiziert:

| Variable | Wert | Verwendung |
|----------|------|------------|
| `MYSQL_DATABASE` | `appdb` | Name der Datenbank |
| `MYSQL_USER` | `appuser` | Benutzername für die App |
| `MYSQL_PASSWORD` | `change_me_please` | Passwort für `appuser` |
| `MYSQL_ROOT_PASSWORD` | `root_password_please_change` | Root-Passwort (nur für Admin) |

### Tabellen-Struktur

Die `db/init.sql` erstellt beim ersten Start **drei Tabellen**:

#### Tabelle `app_users`
Speichert registrierte Benutzer mit sicher gehashtem Passwort.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `username` | `VARCHAR(80) UNIQUE` | Eindeutiger Benutzername |
| `password_hash` | `VARCHAR(255)` | bcrypt-gehashtes Passwort (12 Salt-Rounds) |
| `created_at` | `TIMESTAMP` | Zeitstempel der Erstellung |

#### Tabelle `login_logs`
Protokolliert **jeden Login-Versuch** – egal ob erfolgreich oder fehlgeschlagen.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `user_id` | `INT` | Fremdschlüssel auf `app_users.id` (`ON DELETE SET NULL`) |
| `username` | `VARCHAR(80)` | Benutzername (auch wenn User gelöscht wird) |
| `success` | `BOOLEAN` | `true` = Login erfolgreich, `false` = fehlgeschlagen |
| `ip_address` | `VARCHAR(45)` | IP-Adresse des Clients (IPv4/IPv6-kompatibel) |
| `logged_at` | `TIMESTAMP` | Zeitstempel des Versuchs |

#### Tabelle `products`
Speichert die Shop-Artikel mit Preis und aktuellem Lagerbestand.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `name` | `VARCHAR(255)` | Produktname |
| `price` | `DECIMAL(10, 2)` | Preis in Euro (2 Dezimalstellen) |
| `stock` | `INT DEFAULT 0` | Aktueller Lagerbestand |
| `created_at` | `TIMESTAMP` | Zeitstempel der Erstellung |

#### Demo-Daten
Beim ersten Start werden automatisch 4 Demo-Artikel eingefügt (`INSERT IGNORE` verhindert Duplikate bei Neustarts):

| Produkt | Preis | Bestand |
|---------|-------|---------|
| Premium T-Shirt | 29,99 € | 15 |
| Kaffeebecher | 12,50 € | 42 |
| Laptop-Sticker (Pack) | 5,00 € | 100 |
| Wireless Maus | 49,99 € | 8 |

---

## ⚙️ 4. Das Backend (Node.js / Express)

### Technologie-Stack

| Paket | Version | Zweck |
|-------|---------|-------|
| `express` | `^4.19.2` | Web-Framework für die REST-API |
| `mysql2` | `^3.9.7` | MySQL-Datenbanktreiber (Promise-basiert) |
| `bcryptjs` | `^2.4.3` | Sicheres Passwort-Hashing (12 Salt-Rounds) |
| `jsonwebtoken` | `^9.0.2` | JWT-Token-Erzeugung & -Verifizierung |
| `cors` | `^2.8.5` | Cross-Origin Resource Sharing |

### Datenbankverbindung
Der Backend-Server nutzt einen **Connection Pool** (`mysql2/promise`) mit maximal 10 gleichzeitigen Verbindungen. Die Zugangsdaten werden über Umgebungsvariablen aus der `docker-compose.yml` injiziert:

```javascript
const pool = mysql.createPool({
  host: process.env.DB_HOST || 'db',      // Docker-Service-Name
  user: process.env.DB_USER || 'appuser',
  password: process.env.DB_PASSWORD || 'change_me_please',
  database: process.env.DB_NAME || 'appdb',
  connectionLimit: 10,
});
```

### Auth-Middleware
Geschützte Endpunkte verwenden eine `authMiddleware`-Funktion, die den `Authorization: Bearer <token>`-Header prüft. Das JWT wird mit `jwt.verify()` validiert. Bei Erfolg wird `req.user` mit `{ id, username }` befüllt – bei Fehler kommt `401 Unauthorized`.

### API-Endpunkte im Detail

#### `POST /api/auth/login` – Auto-Register & Login
Das Herzstück der Authentifizierung. Arbeitet in zwei Modi:
1. **Benutzer existiert:** Passwort wird gegen den gespeicherten `bcrypt`-Hash geprüft.
2. **Benutzer existiert nicht:** Es wird automatisch ein neues Konto erstellt (Passwort muss mind. 6 Zeichen haben), danach direkt eingeloggt.

In **beiden Fällen** wird der Login-Versuch (erfolgreich oder nicht) in der `login_logs`-Tabelle protokolliert mit der IP-Adresse des Clients (`X-Forwarded-For`-Header von Nginx).

Bei Erfolg wird ein **JWT-Token** zurückgegeben, der 8 Stunden gültig ist:
```json
{
  "message": "Willkommen zurück, jason!",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "username": "jason"
}
```

#### `POST /api/auth/register` – Klassische Registrierung
Separater Endpunkt für explizite Registrierung. Prüft:
- Ob Benutzername und Passwort angegeben sind
- Ob Passwort mindestens 6 Zeichen hat
- Ob Benutzername bereits vergeben ist (`ER_DUP_ENTRY`)

#### `GET /api/auth/me` – Benutzerinfo (geschützt)
Gibt `{ username, id }` des eingeloggten Users zurück. Erfordert gültigen JWT-Token im Header.

#### `GET /api/auth/logs` – Login-Protokoll (geschützt)
Liefert die **letzten 50 Login-Versuche** aller Benutzer, sortiert nach Zeit (neueste zuerst). Jeder Eintrag enthält: `username`, `success` (Boolean), `ip_address`, `logged_at`.

#### `GET /api/products` – Alle Produkte laden
Gibt alle Zeilen der `products`-Tabelle als JSON-Array zurück, sortiert nach `id`. Kein Authentifizierung nötig.

#### `POST /api/buy/:id` – Einzelkauf (Legacy)
Kauft **ein einziges Stück** eines Produkts. Nutzt eine **MySQL-Transaktion** mit `SELECT ... FOR UPDATE` (pessimistic locking), um Race Conditions bei gleichzeitigen Käufen zu vermeiden:
1. Zeile sperren (`FOR UPDATE`)
2. Bestand prüfen (`stock > 0`)
3. Bestand reduzieren (`stock - 1`)
4. Transaktion committen oder rollbacken

#### `POST /api/checkout` – Warenkorb-Checkout
Verarbeitet einen **kompletten Warenkorb** mit mehreren Artikeln und beliebigen Mengen in einer einzigen Transaktion:

**Request-Body:**
```json
{
  "cart": [
    { "id": 1, "quantity": 2 },
    { "id": 3, "quantity": 5 }
  ]
}
```

**Ablauf:**
1. **Validierungsphase:** Für jedes Produkt im Warenkorb wird geprüft, ob es existiert und ob genügend Bestand vorhanden ist. Bei Fehler wird die gesamte Transaktion abgebrochen.
2. **Ausführungsphase:** Alle Bestände werden um die jeweilige Menge reduziert.
3. **Commit oder Rollback:** Nur wenn alles erfolgreich ist, wird committed. Ansonsten wird alles zurückgerollt – es werden keine Teilkäufe ausgeführt.

### Dockerfile (Backend)
```dockerfile
FROM node:20-alpine       # Schlankes Alpine-Linux-Image mit Node.js 20
WORKDIR /app
COPY package*.json ./     # Nur Package-Files zuerst (Docker-Cache-Optimierung)
RUN npm install           # Abhängigkeiten installieren
COPY src/ ./src/          # Quellcode kopieren
EXPOSE 3000               # Port dokumentieren
CMD ["npm", "start"]      # Server starten: node src/index.js
```

---

## 🌐 5. Nginx Reverse Proxy

Nginx (`nginx:1.27-alpine`) übernimmt zwei zentrale Aufgaben. Die Konfiguration liegt in `nginx/default.conf`:

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # 1. Statisches Hosting: Frontend ausliefern
    location / {
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy: API-Aufrufe an Backend weiterleiten
    location /api/ {
        proxy_pass http://backend:3000/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Warum ein Reverse Proxy?
- **Keine CORS-Probleme:** Da Frontend und API unter derselben Domain laufen (`:80`), gibt es keine Cross-Origin-Konflikte.
- **Sicherheit:** Der Backend-Container ist nur intern erreichbar (kein Port nach aussen gemappt). Alle Anfragen laufen über Nginx.
- **Transparenz:** Der Browser merkt nicht, dass `/api/products` eigentlich an einen anderen Container geht.

### Volume-Mounts
Das Frontend wird per **Volume** in den Container gemountet (`./frontend:/usr/share/nginx/html:ro`). Das `:ro` steht für read-only. Dadurch werden Änderungen an der `index.html` sofort live übernommen – **ohne Docker-Neustart**.

---

## 🎨 6. Das Frontend (Single-Page Application)

Die gesamte Anwendung besteht aus einer einzigen Datei: `frontend/index.html` (~700 Zeilen). Sie enthält HTML, CSS und JavaScript inline – keine externen Frameworks, kein Build-Prozess.

### Tab-Navigation
Die SPA besteht aus **fünf Bereichen**, die per JavaScript ein-/ausgeblendet werden:

| Tab | ID | Beschreibung |
|-----|----|--------------|
| Startseite | `#home` | Hero-Bereich mit grosser Überschrift, Beschreibungstext und CTA-Button „Jetzt einkaufen" |
| Produkte | `#shop` | Dynamisch geladene Produktkarten im CSS-Grid-Layout |
| Warenkorb | `#cart` | Komplette Warenkorbansicht mit Artikeltabelle, Mengensteuerung und Checkout |
| Login | `#login` | Auto-Register-Login-Formular / nach Login: Benutzer-Panel mit Login-Protokoll |
| Kontakt | `#contact` | Kontaktformular (Demo, sendet keine echte Mail) |

Die `switchTab(tabId, btn)`-Funktion blendet per CSS-Klasse (`.active`) den aktiven Bereich ein und alle anderen aus. Beim Wechsel auf „Produkte" werden automatisch die aktuellen Daten vom Backend geladen (`fetchProducts()`).

### Design-System

#### CSS-Variablen (`:root`)
| Variable | Wert | Verwendung |
|----------|------|------------|
| `--primary` | `#9333EA` | Hauptfarbe (Violett) für Buttons, aktive Tabs |
| `--primary-hover` | `#7E22CE` | Dunkleres Violett für Hover-Zustände |
| `--bg-color` | `#000000` | Hintergrundfarbe |
| `--text-main` | `#F8FAFC` | Haupttextfarbe (fast weiss) |
| `--text-muted` | `#CBD5E1` | Sekundärer Text (grau) |
| `--card-bg` | `rgba(30, 41, 59, 0.7)` | Halbtransparenter Kartenhintergrund |
| `--card-border` | `rgba(255, 255, 255, 0.1)` | Dezente helle Kartenränder |
| `--success` | `#10B981` | Grün (Lager verfügbar) |
| `--danger` | `#EF4444` | Rot (Fehler, Ausverkauft) |

#### Design-Techniken
- **Dark Theme & Glassmorphism:** Karten haben einen halbtransparenten Hintergrund (`rgba`) mit `backdrop-filter: blur(12px)` für den Milchglas-Effekt.
- **Gradient-Hintergrund:** Der Body nutzt zwei `radial-gradient`-Layer in Violett, die in den oberen Ecken leuchten.
- **Gradient-Text:** Überschriften nutzen `linear-gradient` mit `-webkit-background-clip: text`, sodass der Text selbst einen Farbverlauf hat.
- **Schrift:** Google Font „Inter" in den Gewichten 400 (normal), 600 (semi-bold), 800 (extra-bold).
- **Schneeflocken:** Ein `setInterval(createSnowflake, 150)` erzeugt alle 150ms ein neues `<div>` mit einem ❄-Emoji, das per `@keyframes fall` von oben nach unten animiert und danach entfernt wird.

### 🛒 Warenkorb-System (im Detail)

Der Warenkorb wird **komplett clientseitig** im JavaScript verwaltet – ein `cart`-Array im Speicher:

```javascript
let cart = [];
// Jedes Element: { id, name, price, quantity, maxStock }
```

#### Ablauf beim Hinzufügen
1. User klickt „In den Warenkorb" auf einer Produktkarte
2. `addToCart(id, name, price, maxStock)` wird aufgerufen
3. Falls Produkt schon im Warenkorb → `quantity++` (bis `maxStock`)
4. Falls neu → neues Objekt ins `cart`-Array pushen
5. `updateCartBadge()` → Zahl im Navigations-Badge aktualisieren
6. `renderCart()` → Tabelle im Warenkorb-Tab neu rendern
7. Toast-Benachrichtigung: „Premium T-Shirt im Warenkorb"

#### Warenkorb-Ansicht
Wenn der User auf den „Warenkorb"-Tab klickt, sieht er:

- **Leerer Zustand:** Ein 🛍️-Icon mit Text „Dein Warenkorb ist noch leer" und Button zurück zum Shop.
- **Gefüllter Zustand:** Eine Tabelle mit:

| Spalte | Inhalt |
|--------|--------|
| Produkt | Produktname (fett) |
| Menge | `−` / Zahl / `+` Buttons zum Ändern der Menge |
| Einzelpreis | z.B. „29,99 €" |
| Gesamt | Einzelpreis × Menge (violett hervorgehoben) |
| Aktion | „✕ Entfernen"-Button (rot) |

- **Footer:** Gesamtbetrag (grosse violette Zahl) + „Jetzt kaufen 🚀"-Button

#### Bestandslogik
- Der **Lagerbestand wird nicht sofort reduziert**, wenn ein Produkt in den Warenkorb gelegt wird. Das ist Standard-E-Commerce-Verhalten.
- Erst beim **Checkout** (`POST /api/checkout`) wird der Bestand in der Datenbank transaktional aktualisiert.
- Die `maxStock`-Eigenschaft im Cart verhindert, dass der User mehr Artikel hinzufügt als auf Lager sind (clientseitige Validierung). Das Backend prüft dies zusätzlich serverseitig.

#### Checkout-Ablauf
1. User klickt „Jetzt kaufen 🚀"
2. Button wird disabled, Text wechselt zu „Wird verarbeitet..."
3. `POST /api/checkout` wird mit dem Warenkorb als JSON gesendet
4. Bei Erfolg: Toast „Einkauf erfolgreich!", Warenkorb wird geleert, Produktliste wird neu geladen (aktualisierte Bestände)
5. Bei Fehler: Toast mit Fehlermeldung (z.B. „Nicht genügend Bestand für Wireless Maus")

### 🔐 Authentifizierungs-System

#### Login-Flow
1. User gibt Benutzername + Passwort ein und klickt „Einloggen" (oder Enter)
2. `POST /api/auth/login` wird gesendet
3. **Neuer User:** Konto wird automatisch erstellt → Toast: „Konto erstellt! Willkommen, jason!"
4. **Bestehender User:** Passwort wird geprüft → Toast: „Willkommen zurück, jason!"
5. JWT-Token und Username werden in `localStorage` gespeichert
6. Die Navigation zeigt nun „👤 jason" + „Ausloggen"-Button an

#### Nach dem Login
- Das Login-Formular wird durch ein **Benutzer-Panel** ersetzt:
  - Grosser runder Avatar mit dem ersten Buchstaben des Usernamens
  - Begrüssung: „Hallo, jason! 👋"
  - Button „🔍 Login-Protokoll laden" → zeigt eine Tabelle mit allen Login-Versuchen (Benutzer, Status ✓/✗, IP-Adresse, Zeitstempel)

#### Logout
- `localStorage` wird geleert, UI wird zurückgesetzt, User wird auf die Startseite weitergeleitet.

### Toast-Benachrichtigungen
Ein eigenes Benachrichtigungssystem zeigt Feedback unten rechts an:
- **Grüner Toast:** Erfolg (z.B. Kauf, Login)
- **Roter Toast:** Fehler (z.B. falsches Passwort, Bestand erschöpft)
- Wird nach 3 Sekunden automatisch ausgeblendet (CSS-Transition)

---

## 🚀 7. Cloudflare Tunnel (Internet-Freigabe)

Der Container `project_tunnel` nutzt das offizielle `cloudflare/cloudflared`-Image und erstellt einen **temporären, öffentlich zugänglichen Tunnel** ohne Cloudflare-Account:

```yaml
tunnel:
    image: cloudflare/cloudflared:latest
    command: tunnel --url http://nginx:80    # Verbindet sich intern mit Nginx
    depends_on:
      - nginx
```

- Er baut eine verschlüsselte Verbindung von dem lokalen Docker-Netzwerk zu den Cloudflare-Servern auf.
- In den Logs des Containers wird ein **eindeutiger Link** generiert (z.B. `https://name-wort.trycloudflare.com`).
- Wer auf diesen Link klickt, wird automatisch sicher auf den lokalen Nginx-Server geleitet.
- Der Link ändert sich bei jedem Neustart des Containers.

---

## 🔧 8. Wichtige Docker-Befehle

| Befehl | Beschreibung |
|--------|--------------|
| `docker-compose up -d` | Alle Container im Hintergrund starten |
| `docker-compose down` | Alle Container stoppen und entfernen |
| `docker-compose down -v` | Container stoppen + **Datenbank-Volume löschen** (Reset) |
| `docker-compose up -d --build` | Backend neu bauen (nach Code-Änderungen in `backend/`) |
| `docker-compose logs tunnel` | Öffentlichen Cloudflare-Link anzeigen |
| `docker-compose logs backend` | Backend-Logs anzeigen (Fehler debuggen) |
| `docker-compose ps` | Status aller Container anzeigen |
| `docker-compose restart nginx` | Nur Nginx neustarten |

### Wann muss was neu gestartet werden?

| Änderung an... | Aktion nötig? |
|----------------|---------------|
| `frontend/index.html` | ❌ Nein – wird per Volume live eingebunden. Browser-Reload reicht. |
| `backend/src/index.js` | ✅ Ja – `docker-compose up -d --build` (neues Docker-Image bauen) |
| `db/init.sql` | ✅ Ja – `docker-compose down -v && docker-compose up -d` (Volume löschen, damit `init.sql` erneut ausgeführt wird) |
| `docker-compose.yml` | ✅ Ja – `docker-compose up -d` (Container werden automatisch neu erstellt) |
| `nginx/default.conf` | ✅ Ja – `docker-compose restart nginx` |
| `.env` | ✅ Ja – `docker-compose down && docker-compose up -d` |