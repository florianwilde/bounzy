# Anmeldung per SSO (OIDC) – Beispiel authentik

Bounzy meldet Benutzer per OpenID Connect an (Authorization Code mit PKCE). Rechte kommen aus den
**Gruppen** des Benutzers: Bounzy liest sie aus einem Claim (Standard `groups`) und gibt nur Benutzern
Zugang, deren Gruppe in Bounzy eine Rolle hat. Die Anleitung nutzt authentik; andere Anbieter (Keycloak,
Entra ID, Zitadel …) gehen genauso – entscheidend sind Redirect-URI und Gruppen-Claim.

Vorher: Bounzy ist unter seiner endgültigen Adresse erreichbar (System › Adresse & Domains, z. B.
`https://bounzy.example.de`), und Sie können sich mit dem lokalen Admin-Konto anmelden. Das bleibt der
Notfall-Zugang, falls SSO einmal nicht geht.

## 1. authentik: Provider und Anwendung

1. *Admin-Oberfläche › Anwendungen › Anwendungen › Mit Provider erstellen* (bzw. erst *Provider › Erstellen*).
2. **Anwendung:** Name `Bounzy`, Slug `bounzy`.
3. **Provider-Typ:** *OAuth2/OpenID Provider*.
   - **Autorisierungs-Flow:** `default-provider-authorization-implicit-consent` (ohne Zustimmungsseite)
     oder `…-explicit-consent`.
   - **Client-Typ:** *Confidential*.
   - **Redirect-URIs:** `https://bounzy.example.de/auth/callback` – Modus *strict*. Für weitere
     Adressen (Mandanten-Domains, interne Adresse) je eine Zeile ergänzen; Bounzy zeigt die nötigen URIs
     unter *Zugang › Anmeldung* und *System › Adresse & Domains*.
   - **Signaturschlüssel:** das authentik-Standardzertifikat (RS256).
   - **Scopes** (erweiterte Einstellungen): `openid`, `email`, `profile` – das Standard-Mapping
     `profile` enthält in authentik bereits den Claim `groups`.
   - **Subjekt-Modus:** *Basierend auf der gehashten Benutzer-ID* (Standard) ist richtig – Bounzy
     identifiziert Benutzer über Issuer + `sub`, nicht über die E-Mail.
4. Nach dem Speichern auf der Provider-Seite notieren:
   - **Client-ID** und **Client-Secret**
   - **OpenID-Konfigurations-Issuer**, z. B. `https://auth.example.de/application/o/bounzy/`
5. **Zugriff begrenzen (empfohlen):** in der Anwendung unter *Richtlinien/Gruppen/Benutzer-Bindungen*
   nur die Gruppen binden, die Bounzy nutzen sollen.

## 2. authentik: Gruppen anlegen

*Verzeichnis › Gruppen* – zum Beispiel:

| Gruppe | Zweck in Bounzy |
| --- | --- |
| `bounzy-admins` | Admin der ganzen Instanz |
| `bounzy-technik` | Techniker (verbinden, keine Verwaltung) |
| `kunde-a-technik` | Techniker nur für Mandant „Kunde A“ |

Benutzer den Gruppen zuordnen. Gruppenänderungen wirken beim nächsten Login in Bounzy.

## 3. Bounzy: SSO einrichten

1. Als Admin anmelden, **Zugang › Anmeldung** öffnen.
2. Ausfüllen:
   - **Issuer-URL:** der Issuer aus Schritt 1.4 (mit `/` am Ende, genau wie authentik ihn anzeigt)
   - **Client-ID** und **Client-Secret** (das Secret liegt danach verschlüsselt im Tresor)
   - **Gruppen-Claim:** `groups`
   - **Scopes:** `openid profile email groups` (Standard; `groups` ignoriert authentik, schadet nicht)
   - **Beschriftung des Knopfs:** z. B. `Mit authentik anmelden`
3. **Verbindung testen** – Bounzy lädt die OpenID-Konfiguration des Issuers. Schlägt das fehl: Issuer-URL
   prüfen und ob der Bounzy-Server `auth.example.de` per HTTPS erreicht (DNS, Firewall, Zertifikat).
4. **Anmeldung per SSO** einschalten und speichern.

## 4. Bounzy: Gruppen berechtigen

**Zugang › Benutzer & Rechte › Gruppe berechtigen**:

- `bounzy-admins` → Rolle *Admin*, Ziel *ganze Instanz*
- `bounzy-technik` → Rolle *Techniker*, Ziel *ganze Instanz* oder ein Ordner
- `kunde-a-technik` → Rolle *Techniker*, Ziel *Mandant Kunde A*

Erst wenn eine Admin-Gruppe berechtigt ist, sollten Sie den Notfall-Zugang abschalten.

## 5. Testen

1. In einem privaten Fenster `https://bounzy.example.de` öffnen › *Mit authentik anmelden*.
2. Nach der Anmeldung in authentik landen Sie in Bounzy. Unter **Benutzer & Rechte** erscheint das Konto
   mit seinen Gruppen.
3. Abgewiesen mit „keine Berechtigung“? Dann hat keine der Gruppen eine Rolle – im Audit-Log steht
   `auth.login` / `denied` / `no_role` mit den empfangenen Gruppen.

## Häufige Fehler

| Meldung / Verhalten | Ursache und Abhilfe |
| --- | --- |
| authentik: „Redirect URI Error“ | Die Redirect-URI in authentik passt nicht genau zur Bounzy-Adresse (Schema, Host, `/auth/callback`). Bounzy zeigt die erwartete URI unter *Zugang › Anmeldung*. |
| Redirect-URI zeigt eine interne IP | Die App-URL von Bounzy ist noch die interne Adresse – unter *System › Adresse & Domains* die öffentliche eintragen. |
| „Verbindung testen“ schlägt fehl | Issuer-URL ohne bzw. mit falschem Pfad, oder der Bounzy-Server erreicht authentik nicht (Proxy, DNS, selbstsigniertes Zertifikat). |
| Anmeldung klappt, aber „keine Berechtigung“ | Gruppe nicht in Bounzy berechtigt, oder der Claim heißt anders (Gruppen-Claim prüfen). |
| Abmeldung beendet authentik-Sitzung nicht | Nur wenn authentik einen End-Session-Endpunkt meldet; bei authentik ist das Standard. |
| „Zu viele Anmeldeversuche“ | Viele Benutzer hinter einer NAT-IP: `BOUNZY_OIDC_LOGIN_LIMIT` erhöhen (siehe BETRIEB.md). |

## Andere Anbieter

- **Keycloak:** Client *OpenID Connect*, *Client authentication* an, Redirect-URI wie oben. Gruppen per
  Mapper *Group Membership* als Claim `groups` (ohne „Full group path“) in ID- und Access-Token.
- **Microsoft Entra ID:** App-Registrierung, Redirect-URI (Web) wie oben, Issuer
  `https://login.microsoftonline.com/<Tenant-ID>/v2.0`. *Token-Konfiguration › Gruppenanspruch
  hinzufügen* (Sicherheitsgruppen); Entra liefert Gruppen-**IDs** – diese IDs in Bounzy berechtigen.
