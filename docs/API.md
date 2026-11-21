# REST-API v1

Für Skripte, Monitoring und Ticketsysteme. Basis-URL: `<BOUNZY_BASE_URL>/api/v1`, Antworten in JSON.

## Anmeldung

Jeder Benutzer erzeugt seine Tokens selbst unter **API-Tokens** (Navigation). Ein Token

- handelt mit den Rechten seines Benutzers (Rollen wie in der Oberfläche) und nur in den gewählten Bereichen,
- wird genau einmal angezeigt; gespeichert ist nur sein SHA-256-Hash,
- läuft nach 7, 30, 90 oder 365 Tagen ab (oder nie) und lässt sich jederzeit widerrufen,
- beginnt mit `bzt_` – Secret-Scanner erkennen es dadurch.

```bash
curl -H "Authorization: Bearer $BOUNZY_TOKEN" https://bounzy.example.de/api/v1/connections
```

Cookies der Oberfläche gelten für die API nicht, und die API akzeptiert keine Cookies – CSRF ist damit
ausgeschlossen. **Zugangsdaten lassen sich über die API nie anzeigen**; anlegen bzw. ersetzen ist möglich
(`credential` wie im Formular). Je Token höchstens 600 Aufrufe pro Minute (danach `429` mit `Retry-After`).
Alle Änderungen erscheinen im Audit-Log als „Benutzer (API-Token „Name“)“.

## Bereiche (Scopes)

| Bereich | Erlaubt |
|---|---|
| `connections:read` | `GET /connections`, `GET /connections/{id}`, `GET /me` |
| `connections:write` | `POST /connections`, `PUT /connections/{id}`, `DELETE /connections/{id}`, `POST /connections/delete` (Verwaltungsrecht nötig) |
| `sessions:read` | `GET /sessions` – laufende Sitzungen der verwalteten Mandanten |
| `sessions:write` | `DELETE /sessions/{id}` – Sitzung beenden |
| `audit:read` | `GET /audit` |

## Endpunkte

### `GET /me`
```json
{ "user": "ada@example.test (API-Token „Monitoring“)", "admin": true }
```

### `GET /connections`
Alle Verbindungen, die der Benutzer sieht.
```json
{ "connections": [ { "id": "cm…", "shortId": "Feop…", "name": "dc01", "protocol": "RDP", "host": "10.0.0.10",
  "port": 3389, "tenant": "Kunde A", "group": "Kunde A/Server", "tags": ["windows"], "gateway": "Lokal",
  "critical": false, "readonly": false, "reachable": true, "reachableAt": "2026-11-15T14:00:00.000Z" } ] }
```

### `GET /connections/{id}`
Eine Verbindung (ID oder Kurz-ID), Felder wie oben. Nicht sichtbar → `404`.

### `POST /connections` · `PUT /connections/{id}`
Gleiche Felder wie das Verbindungsformular, z. B.:
```json
{ "gatewayId": "cm…", "name": "web01", "protocol": "SSH", "host": "10.0.0.20", "port": 22,
  "group": "Kunde A/Linux", "tags": ["linux"],
  "credential": { "type": "inline", "kind": "PASSWORD", "username": "root", "secret": { "password": "…" } } }
```
`credential.type`: `none`, `prompt`, `inherit` (vom Ordner), `vault` (`credentialId`), `inline`.
Protokolle: `RDP`, `VNC`, `SSH`, `TELNET`, `WEB` (`params.url`). Im Mehrmandanten-Betrieb zusätzlich
`tenantId`. Antwort: `{ "id": "…", "shortId": "…" }`.

### `DELETE /connections/{id}`
Antwort `{ "ok": true }`. Sitzungsverlauf und Aufnahmen bleiben erhalten; läuft gerade eine Sitzung → `409`.

### `POST /connections/delete`
Mehrere auf einmal (höchstens 500), alles oder nichts: `{ "ids": ["cm…", "cm…"] }` → `{ "deleted": 2 }`.
Ist eine davon nicht sichtbar → `404`, läuft auf einer gerade eine Sitzung → `409`, gelöscht wird dann keine.

### `GET /sessions`
```json
{ "sessions": [ { "id": "…", "connection": "dc01", "protocol": "RDP", "user": "tech@example.test",
  "tenant": "Kunde A", "startedAt": "…", "readonly": false, "recording": true, "viewers": 0 } ] }
```

### `DELETE /sessions/{id}`
Beendet die Sitzung (wie „Beenden“ unter Live), Antwort `{ "ok": true }`.

### `GET /audit`
Neueste zuerst, 50 je Seite. Filter: `category` (`session`, `auth`, `admin`), `result`
(`success`, `failure`, `denied`), `q` (Suche), `tenantId`, `from`/`to` (`YYYY-MM-DD`), Cursor `before`.
```json
{ "events": [ { "id": "1234", "ts": "…", "actorLabel": "…", "action": "session.start",
  "actionLabel": "Sitzung gestartet", "result": "success", "tenantName": "…", "target": "dc01",
  "ip": "…", "summary": "…" } ], "nextCursor": "1184" }
```

## Fehler

| Status | Bedeutung |
|---|---|
| `401` | Token fehlt, falsch, abgelaufen oder widerrufen |
| `403` | Bereich fehlt im Token oder Benutzer ohne Recht |
| `404` | Objekt nicht vorhanden oder nicht sichtbar |
| `409` | Konflikt (z. B. Name bereits vergeben) |
| `422` | Eingaben ungültig – `{ "error": "…", "fields": { "name": "…" } }` |
| `429` | Zu viele Anfragen |

## Webhooks

Admins richten unter **Verwaltung › Webhooks** Empfänger je Mandant ein (nur `https://`). Bounzy sendet
Ereignisse aus dem Audit-Log der gewählten Bereiche (`alert`, `session`, `approval`, `connection`, `auth`, `admin`)
als `POST` in Paketen zu höchstens 50 – Ereignisse ohne Mandant (z. B. Anmeldungen) gehen an Webhooks des
Standard-Mandanten. Als zugestellt gilt eine Antwort `2xx` binnen 10 s (keine Weiterleitungen); sonst
wiederholt Bounzy mit wachsendem Abstand (30 s … 1 h), ohne Ereignisse zu verlieren.

```json
{ "webhook": "SIEM", "events": [ { "id": "1234", "ts": "2026-11-15T14:00:00.000Z", "type": "session.start",
  "category": "session", "result": "success", "actor": "tech@example.test", "tenant": "Kunde A",
  "target": { "type": "Connection", "id": "cm…" }, "ip": "203.0.113.5", "details": { "protocol": "RDP" } } ] }
```

Signatur prüfen (Geheimnis `whsec_…` wird beim Anlegen bzw. Erneuern einmal angezeigt):

```js
const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(`${timestamp}.${rawBody}`).digest('hex');
// timestamp = Header X-Bounzy-Timestamp (Unix-Sekunden, höchstens 5 min alt), Vergleich zeitkonstant
// mit X-Bounzy-Signature; „Test senden“ schickt ein Ereignis webhook.test mit "test": true
```
