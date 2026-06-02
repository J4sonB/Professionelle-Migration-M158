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

### Tabellen-Struktur

Die `db/init.sql` erstellt beim ersten Start **drei Tabellen**:

#### Tabelle `kunden`
Speichert die eindeutigen Stammdaten der Benutzer/Kunden.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `kunde_id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `name` | `VARCHAR(100)` | Vor- und Nachname des Kunden |
| `lieferadresse` | `TEXT` | Vollständige Versandadresse (Strasse, PLZ, Ort) |

#### Tabelle `artikel`
Enthält das Produktsortiment mit den verfügbaren Varianten und Preisen.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `artikel_id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `produkt_name` | `VARCHAR(100)` | Bezeichnung des Produkts |
| `groesse` | `VARCHAR(10)` | Verfügbare Größe (z.B. S, M, L, XL) |
| `farbe` | `VARCHAR(30)` | Farbe der Produktvariante |
| `preis` | `DECIMAL(10, 2)` | Preis des Artikels |

#### Tabelle `bestellungen`
Diese Transaktionstabelle verbindet über Fremdschlüssel die Kunden mit den bestellten Artikeln und hält den genauen Bestellzeitpunkt fest.

| Spalte | Typ | Beschreibung |
|--------|-----|--------------|
| `bestell_id` | `INT AUTO_INCREMENT` | Primärschlüssel |
| `fk_kunde_id` | `INT` | Fremdschlüssel auf `kunden.kunde_id` |
| `fk_artikel_id` | `INT` | Fremdschlüssel auf `artikel.artikel_id` |
| `bestelldatum` | `TIMESTAMP` | Zeitstempel des Bestelleingangs |

---

## ⚙️ 4. Das Backend (Node.js / Express)

### Technologie-Stack

| Paket | Version | Zweck |
|-------|---------|-------|
| `express` | `^4.19.2` | Web-Framework für die REST-API |
| `mysql2` | `^3.9.7` | MySQL-Datenbanktreiber (Promise-basiert) |
| `cors` | `^2.8.5` | Cross-Origin Resource Sharing |

### API-Endpunkte im Detail

#### `GET /api/artikel` – Alle Produkte laden
Gibt alle Zeilen der `artikel`-Tabelle als JSON-Array zurück, sortiert nach `artikel_id`.

#### `POST /api/checkout` – Warenkorb-Checkout
Verarbeitet den Checkout-Prozess. Da die App kein Login-System nutzt, werden Name und Adresse direkt im Checkout übermittelt.

**Request-Body:**
```json
{
  "name": "Max Mustermann",
  "lieferadresse": "Musterweg 5, 12345 Stadt",
  "cart": [
    { "id": 1, "quantity": 2 },
    { "id": 3, "quantity": 1 }
  ]
}
```

**Ablauf (Transaktional):**
1. **Kunde prüfen:** Es wird geprüft, ob ein Kunde mit diesem Namen bereits existiert. Falls ja, wird die `lieferadresse` aktualisiert. Falls nein, wird ein neuer Kunde in `kunden` angelegt.
2. **Bestellungen eintragen:** Für jeden Artikel im Warenkorb wird entsprechend der gekauften Menge (quantity) eine neue Zeile in die Tabelle `bestellungen` geschrieben. Bei 2x Artikel ID 1 entstehen 2 Zeilen in `bestellungen`.
3. **Commit/Rollback:** Das Ganze läuft als Transaktion ab.

---

## 🌐 5. Nginx Reverse Proxy
*(Keine Änderungen zum vorherigen Aufbau - Nginx leitet /api/ an das Backend weiter).*

---

## 🎨 6. Das Frontend (Single-Page Application)

Die gesamte Anwendung besteht aus einer einzigen Datei: `frontend/index.html` (~700 Zeilen). Sie enthält HTML, CSS und JavaScript inline.

### Tab-Navigation
Die SPA besteht aus **vier Bereichen**, die per JavaScript ein-/ausgeblendet werden:

| Tab | ID | Beschreibung |
|-----|----|--------------|
| Startseite | `#home` | Hero-Bereich mit großer Überschrift |
| Produkte | `#shop` | Dynamisch geladene Artikelkarten (mit Anzeige von Größe und Farbe) |
| Warenkorb | `#cart` | Komplette Warenkorbansicht mit Artikeltabelle, Eingabe von Name/Adresse und Checkout |
| Kontakt | `#contact` | Kontaktformular (Demo) |

### 🛒 Warenkorb-System (im Detail)

Der Warenkorb wird **komplett clientseitig** im JavaScript verwaltet – ein `cart`-Array im Speicher:

#### Warenkorb-Ansicht
Wenn der User auf den „Warenkorb"-Tab klickt, sieht er die Tabelle der Artikel. 
Im unteren Footer gibt es nun **zwei Eingabefelder**:
- Dein Name
- Lieferadresse

#### Checkout-Ablauf
1. User füllt Name und Lieferadresse aus und klickt „Jetzt kaufen 🚀".
2. Button wird disabled, Text wechselt zu „Wird verarbeitet...".
3. `POST /api/checkout` wird mit den Feldern und dem Warenkorb gesendet.
4. Bei Erfolg: Toast „Einkauf erfolgreich!", Warenkorb wird geleert.

### Toast-Benachrichtigungen
Ein eigenes Benachrichtigungssystem zeigt Feedback unten rechts an (z.B. wenn Felder im Checkout fehlen).

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