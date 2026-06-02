# 📖 Projektdokumentation: Docker Shop

Diese Dokumentation beschreibt die Architektur und alle Schritte, die unternommen wurden, um das Projekt von einer statischen Nginx/PostgreSQL-Seite zu einer dynamischen, modernen **Full-Stack E-Commerce-Anwendung** umzubauen.

---

## 🏛️ 1. Architektur-Übersicht

Das Projekt basiert auf einer **Microservices-Architektur**, die vollständig über `docker-compose` orchestriert wird. Es besteht aus vier Hauptkomponenten:

1. **Frontend (Nginx)**: Liefert die Webseite aus und dient als "Reverse Proxy".
2. **Backend (Node.js)**: Verarbeitet die Geschäftslogik (APIs) und spricht mit der Datenbank.
3. **Datenbank (MySQL)**: Speichert Artikel, Benutzer und Login-Protokolle persistent ab.
4. **Cloudflare Tunnel**: Stellt das lokale Setup weltweit über eine sichere, dynamische URL zur Verfügung.

---

## 💾 2. Die Datenbank (MySQL)

### Wechsel von PostgreSQL zu MySQL
Ursprünglich war das Projekt für PostgreSQL konfiguriert. Wir haben dies vollständig auf MySQL umstrukturiert:
- **Port-Konflikt gelöst:** Da auf deinem Mac bereits ein lokaler Dienst auf Port `3306` lief, haben wir den MySQL-Port nach außen auf `3307` verlegt (`3307:3306`).
- **Initialisierung:** Die Datei `db/init.sql` wurde mit MySQL-Syntax ausgestattet (z. B. `AUTO_INCREMENT` statt `SERIAL`, `INSERT IGNORE` statt `ON CONFLICT`).

### Tabellen-Struktur
In der `init.sql` werden beim ersten Start drei Tabellen erstellt:
- **`app_users`**: Speichert registrierte Benutzer mit gehashtem Passwort (`id`, `username`, `password_hash`, `created_at`).
- **`login_logs`**: Protokolliert alle Login-Versuche mit Erfolgs-/Fehlerstatus, IP-Adresse und Zeitstempel.
- **`products`**: Speichert die Shop-Artikel (`id`, `name`, `price`, `stock`). Hier haben wir zum Start 4 Demo-Artikel mit Lagerbeständen eingefügt.

---

## ⚙️ 3. Das Backend (Node.js / Express)

Um sicher mit der Datenbank zu kommunizieren, wurde ein eigener Container im Ordner `backend/` geschaffen.

- **Stack:** Node.js, Express.js (Web-Framework), `mysql2` (Datenbanktreiber), `bcryptjs` (Passwort-Hashing), `jsonwebtoken` (JWT-Authentifizierung).

### API-Endpunkte

| Methode | Endpunkt | Beschreibung |
|---------|----------|--------------|
| `POST` | `/api/auth/login` | **Auto-Register & Login** – Existiert der Benutzer nicht, wird automatisch ein Konto erstellt. Gibt ein JWT-Token zurück. |
| `POST` | `/api/auth/register` | Klassische Registrierung (separater Endpunkt). |
| `GET` | `/api/auth/me` | Gibt die Benutzerdaten des eingeloggten Users zurück (geschützt per JWT). |
| `GET` | `/api/auth/logs` | Lädt die letzten 50 Login-Versuche aller Benutzer (geschützt per JWT). |
| `GET` | `/api/products` | Holt alle Artikel aus der MySQL-Tabelle und gibt sie als JSON ans Frontend. |
| `POST` | `/api/buy/:id` | Kauft ein einzelnes Produkt – reduziert den Bestand um 1 (mit Transaktion). |
| `POST` | `/api/checkout` | **Warenkorb-Checkout** – Verarbeitet einen kompletten Warenkorb mit mehreren Artikeln und Mengen in einer einzigen Transaktion. Prüft den Bestand für jedes Produkt vorab. |

- **Dockerisierung:** Ein eigenes `Dockerfile` installiert die Abhängigkeiten (`package.json`) und startet den Node-Server auf Port `3000`.

---

## 🌐 4. Nginx Reverse Proxy

Nginx ist das Tor zum System. In der `nginx/default.conf` haben wir Nginx zwei Aufgaben gegeben:
1. **Statisches Hosting:** Alles, was normal aufgerufen wird (z. B. `/`), liefert Nginx aus dem `/frontend`-Ordner (HTML/CSS) aus.
2. **Reverse Proxy:** Alles, was an `/api/...` gesendet wird, leitet Nginx unsichtbar an den internen `backend`-Container weiter (`proxy_pass http://backend:3000/api/;`). So vermeiden wir komplexe CORS-Fehler im Browser.

---

## 🎨 5. Das Frontend (Single-Page Application)

Die `frontend/index.html` wurde zu einer interaktiven SPA (Single-Page Application) im modernen Design umgeschrieben.

### Tab-Navigation
Die Anwendung besteht aus fünf Bereichen, zwischen denen per Tab-Navigation gewechselt wird:

| Tab | Beschreibung |
|-----|--------------|
| **Startseite** | Hero-Bereich mit Willkommensnachricht und CTA-Button zum Shop. |
| **Produkte** | Dynamisch geladene Produktkarten mit Preis, Lagerbestand und „In den Warenkorb"-Button. |
| **Warenkorb** | Auflistung aller Artikel im Warenkorb mit Mengensteuerung (+/−), Einzelpreis, Gesamtpreis pro Artikel, Gesamtbetrag und Checkout-Button. |
| **Login** | Auto-Register-Formular – existiert der Benutzer nicht, wird automatisch ein Konto erstellt. Nach dem Login: Benutzer-Panel mit Login-Protokoll. |
| **Kontakt** | Kontaktformular (Demo). |

### 🛒 Warenkorb-System
- Artikel werden über „In den Warenkorb" hinzugefügt.
- Der **Warenkorb-Badge** in der Navigation zeigt die aktuelle Anzahl der Artikel an.
- Im Warenkorb-Tab werden alle Artikel in einer **Tabelle** aufgelistet:
  - Produktname, Menge (mit +/− Buttons), Einzelpreis, Gesamt pro Zeile.
  - Artikel können einzeln entfernt werden.
  - Der **Gesamtbetrag** wird automatisch berechnet.
- Der Bestand wird **erst beim Checkout** in der Datenbank reduziert (nicht beim Hinzufügen zum Warenkorb). Das ist das Standard-E-Commerce-Verhalten.
- Nach erfolgreichem Checkout wird der Warenkorb geleert und die Produktanzeige aktualisiert.

### Design Features (CSS)
- **Dark-Theme & Glassmorphism:** Milchglas-Effekte (`backdrop-filter: blur()`), dunkle Hintergrundtöne und elegante violette Farbverläufe (`linear-gradient`).
- **Schneeflocken:** Ein JavaScript-Skript erzeugt dynamisch kleine `<div>`-Elemente, die als fallende Schneeflocken animiert werden.
- **Benachrichtigungen:** Ein eigens programmiertes "Toast"-Benachrichtigungssystem ploppt unten rechts auf, um Bestätigungen (z.B. "Premium T-Shirt im Warenkorb") anzuzeigen.
- **Responsive Design:** Die Produktkarten passen sich per CSS Grid automatisch an die Bildschirmbreite an.

### JavaScript Logik
- **Tab-Navigation (`switchTab`)**: Blendet per CSS-Klasse (`display: block` vs `display: none`) zwischen den fünf Bereichen hin und her.
- **Dynamisches Rendering**: Ruft `fetch('/api/products')` auf und generiert das HTML für die Artikelkarten live im Browser.
- **Warenkorb-Management**: Lokales `cart`-Array speichert alle Artikel mit Menge. Die Funktionen `addToCart()`, `removeFromCart()`, `changeQuantity()` und `renderCart()` steuern die gesamte Warenkorb-Logik clientseitig.
- **Checkout-Logik**: Sendet den gesamten Warenkorb als `POST /api/checkout` an das Backend, das alle Bestände transaktional aktualisiert.
- **Auth-System**: Login/Register mit JWT-Token, der im `localStorage` gespeichert wird. Eingeloggte Benutzer sehen ihren Namen in der Navigation und können das Login-Protokoll einsehen.

---

## 🚀 6. Cloudflare Tunnel (Internet-Freigabe)

Um die Seite mit dem Internet zu teilen, haben wir den Dienst `cloudflared` in die `docker-compose.yml` integriert.
- Er baut eine verschlüsselte Verbindung von deinem lokalen Docker-Netzwerk zu den Cloudflare-Servern auf.
- In den Logs des Containers (`docker-compose logs tunnel`) wird ein eindeutiger Link generiert (z. B. `https://name-wort.trycloudflare.com`).
- Wer auf diesen Link klickt, wird automatisch sicher auf deinen lokalen Nginx-Server geleitet.

---

## 🔧 7. Wichtige Docker-Befehle

Hier ist dein Cheatsheet für den täglichen Umgang mit deinem Projekt:

- **Projekt starten** (im Hintergrund): `docker-compose up -d`
- **Projekt stoppen**: `docker-compose down`
- **Datenbank komplett zurücksetzen**: `docker-compose down -v` (löscht das Datenbank-Volume)
- **Öffentlichen Cloudflare-Link anzeigen**: `docker-compose logs tunnel`
- **Backend neu bauen** (nach Code-Änderungen im Backend): `docker-compose up -d --build`

*Hinweis: Änderungen im Frontend (z.B. `index.html`) werden sofort live im Browser sichtbar, sobald du speicherst und neu lädst. Es ist kein Docker-Neustart nötig.*