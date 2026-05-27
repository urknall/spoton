# Phase 2: Auth + API Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-27
**Phase:** 02-auth-api-foundation
**Areas discussed:** Credential-Eingabe & Speicherung, Multi-Account & Account-Wechsel, Rate-Limiting-Strategie, Batch-API-Entfall & API-Architektur

---

## Credential-Eingabe & Speicherung

### Frage 1: Wie sollen Nutzer ihre Spotify-Zugangsdaten eingeben?

| Option | Description | Selected |
|--------|-------------|----------|
| Username/Passwort im Settings-Formular | Nutzer gibt Spotify-Mail und Passwort in der LMS-Settings-Seite ein. Plugin übergibt sie an librespot für login5-Authentifizierung. Einfachster Weg, Spotty-Muster. | |
| Zeroconf-Discovery über Spotify-App | Nutzer verbindet sich per Spotify-App mit dem librespot-Daemon. Credentials werden automatisch gespeichert. Kein Passwort im LMS-Formular nötig, aber setzt laufenden Daemon voraus. | |
| Claude entscheidet | Research-Phase evaluiert beide Ansätze und wählt basierend auf Machbarkeit und UX. | ✓ |

**User's choice:** Claude entscheidet
**Notes:** Research-Phase soll beide Ansätze evaluieren.

### Frage 2: Wo sollen die Spotify-Credentials dauerhaft gespeichert werden?

| Option | Description | Selected |
|--------|-------------|----------|
| librespot-Cache-Verzeichnis | Credentials als auth-blob im Cache-Dir unter prefs/spoton/. Entspricht dem librespot-nativen Format. chmod 600 direkt durchsetzbar. | ✓ |
| LMS Prefs (verschlüsselt) | Credentials direkt in den LMS-Preferences gespeichert. Nachteil: LMS Prefs sind Klartext-YAML, müssten verschlüsselt werden. | |
| Claude entscheidet | Research-Phase evaluiert Speicheroptionen. | |

**User's choice:** librespot-Cache-Verzeichnis
**Notes:** Natürliches Format für Keymaster/login5, chmod 600/700 direkt durchsetzbar.

### Frage 3: Wie soll die Token-Beschaffung via librespot funktionieren?

| Option | Description | Selected |
|--------|-------------|----------|
| Kurzlebiger Prozess pro Token | Plugin spawnt librespot mit --get-token, erhält Token über stdout, Prozess beendet sich. Erneuter Spawn bei Refresh. | ✓ |
| Persistenter Auth-Daemon | Ein langlebiger librespot-Prozess läuft permanent und liefert Tokens auf Anfrage. | |
| Claude entscheidet | Research-Phase klärt, welcher Ansatz technisch optimal ist. | |

**User's choice:** Kurzlebiger Prozess pro Token
**Notes:** Kein unnötiger Daemon-Overhead. Connect-Daemons kommen erst in Phase 5.

### Frage 4: Wie soll die automatische Token-Erneuerung getriggert werden?

| Option | Description | Selected |
|--------|-------------|----------|
| Proaktiver Timer | Plugin startet einen LMS-Timer der z.B. bei 45 von 60 Minuten den Token erneuert — bevor er abläuft. | ✓ |
| Lazy bei Bedarf | Token wird erst erneuert wenn ein API-Call einen 401 erhält. | |
| Hybrid: proaktiv + Fallback | Proaktiver Timer als Normalfall, plus 401-Handler als Sicherheitsnetz. | |

**User's choice:** Proaktiver Timer
**Notes:** Kein API-Call soll je wegen abgelaufenem Token fehlschlagen.

---

## Multi-Account & Account-Wechsel

### Frage 1: Wie sollen mehrere Spotify-Konten organisiert werden?

| Option | Description | Selected |
|--------|-------------|----------|
| Globale Kontoliste + Menü-Switcher | Konten werden in den Plugin-Settings konfiguriert. Im OPML-Menü erscheint ein Switcher als erste Zeile. | ✓ |
| Einzelkonto jetzt, Multi später | Phase 2 unterstützt nur ein Konto. Multi-Account wird verschoben. | |
| Claude entscheidet | Research-Phase evaluiert den besten Zeitpunkt. | |

**User's choice:** Globale Kontoliste + Menü-Switcher
**Notes:** User fragte zunächst nach Account als Top-Level-Menü-Ebene (Drill-Down pro Account). Gemeinsam evaluiert: Account-Switcher als erste Zeile im Menü ist pragmatischer — vermeidet eine Extra-Ebene im OPML für den Ein-Account-Normalfall. User bestätigte den Switcher-Ansatz. Technisch machbar via `nextWindow => 'refreshOrigin'`.

### Frage 2: Connect ohne konfiguriertes Konto?

**User's question:** Funktioniert Connect auch wenn das Spotify-Konto nicht in LMS eingerichtet ist? Z.B. Frau connectet mit ihrem Spotify, nur das eigene Konto ist konfiguriert.
**Answer:** Ja. Connect (Phase 5) funktioniert unabhängig — Spotify-App authentifiziert direkt am librespot-Daemon via Zeroconf. Nur Browse/Search/Library braucht konfiguriertes Konto.

### Frage 3: Soll Multi-Account bereits in Phase 2?

| Option | Description | Selected |
|--------|-------------|----------|
| Multi-Account in Phase 2 | Settings-Seite, API/Client und Token-Management arbeiten von Anfang an mit Account-IDs. | ✓ |
| Ein Konto jetzt, Multi in Phase 3 | Phase 2 nur ein Konto, Multi-Account-UI kommt mit dem Menü in Phase 3. | |
| Claude entscheidet | Research-Phase bestimmt den optimalen Zeitpunkt. | |

**User's choice:** Multi-Account in Phase 2
**Notes:** Vermeidet späteres Refactoring.

### Frage 4: Wie sollen Konten in den Settings verwaltet werden?

| Option | Description | Selected |
|--------|-------------|----------|
| Dynamische Liste mit Hinzufügen/Entfernen | Settings-Seite zeigt eine Liste konfigurierter Konten mit "+ Konto hinzufügen" Button. | ✓ |
| Feste Slots (max. 4 Konten) | Settings-Seite hat 4 feste Konto-Felder. Einfacher aber unelegant. | |
| Claude entscheidet | Research-Phase evaluiert was im LMS-Settings-HTML-Template am besten machbar ist. | |

**User's choice:** Dynamische Liste mit Hinzufügen/Entfernen

---

## Rate-Limiting-Strategie

### Frage 1: Welcher Rate-Limiting-Mechanismus?

| Option | Description | Selected |
|--------|-------------|----------|
| Token Bucket | Klassischer Ansatz: N Tokens pro Zeitfenster. Erlaubt kleine Bursts. | |
| Sliding Window | Zählt Requests in einem gleitenden Zeitfenster. Präziser. | |
| Adaptive (dynamisch) | Startet hoch, reduziert nach 429, erhöht bei Erfolg. Selbstjustierend. | |
| Claude entscheidet | Research-Phase evaluiert den besten Mechanismus. | ✓ |

**User's choice:** Claude entscheidet

### Frage 2: Max gleichzeitige API-Requests?

| Option | Description | Selected |
|--------|-------------|----------|
| Max 3 gleichzeitig | Konservativ. Spotty-NG hatte Probleme ab 5+. | |
| Max 1 (strikt seriell) | Einfachste Implementierung. Null 429-Risiko. | |
| Claude entscheidet | Research-Phase bestimmt optimales Limit. | ✓ |

**User's choice:** Claude entscheidet

### Frage 3: Request-Queue mit Priorisierung?

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, mit Prioritäten | Token-Refresh und Playback haben Vorrang vor Browse/Search. | |
| FIFO ohne Prioritäten | Alle Requests gleich behandelt. Einfacher. | |
| Claude entscheidet | Research-Phase evaluiert Notwendigkeit. | ✓ |

**User's choice:** Claude entscheidet

### Frage 4: Nutzer-Feedback bei Drosselung?

| Option | Description | Selected |
|--------|-------------|----------|
| Nur im Log | 429-Responses und Throttling nur im Debug-Log. | |
| Menü-Hinweis bei Drosselung | Bei aktiver Drosselung Hinweis im OPML-Menü. Mehr Transparenz. | ✓ |
| Claude entscheidet | Research-Phase bestimmt das beste Feedback-Muster. | |

**User's choice:** Menü-Hinweis bei Drosselung
**Notes:** Transparenz für den Nutzer wichtiger als Einfachheit.

---

## Batch-API-Entfall & API-Architektur

### Frage 1: Batch-Fähigkeit vorbereiten?

| Option | Description | Selected |
|--------|-------------|----------|
| Rein Einzel-Requests | Kein Batch-Abstraktionslayer. Extended Quota unrealistisch für Open-Source. YAGNI. | ✓ |
| Batch-Abstraktion vorbereiten | fetchMultiple() intern Einzel-Requests, per Config auf echte Batch umschaltbar. | |
| Claude entscheidet | Research-Phase evaluiert. | |

**User's choice:** Rein Einzel-Requests
**Notes:** Extended Quota erfordert 250k MAU + kommerzielle Org — unrealistisch für Open-Source-Plugin.

### Frage 2: API-Client-Struktur?

| Option | Description | Selected |
|--------|-------------|----------|
| Monolithisch mit Methoden pro Endpunkt-Gruppe | Ein Client.pm mit getTrack(), search(), getMe(). Intern gemeinsamer _request(). | |
| Geschichtete Module (Client + Endpoints) | Client.pm nur HTTP/Auth/Throttle. Separate Module für Endpoint-Gruppen. | |
| Claude entscheidet | Research-Phase evaluiert optimale Modul-Struktur für LMS-Plugins. | ✓ |

**User's choice:** Claude entscheidet

### Frage 3: Endpoint-Scope in Phase 2?

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Auth-relevante Endpoints | Token-Management, GET /me, Error-Handling, Rate-Limiting-Infrastruktur. | ✓ |
| Alle Endpoints vorab | Alle Spotify-Endpoints sofort implementieren. | |
| Claude entscheidet | Research-Phase bestimmt den Scope. | |

**User's choice:** Nur Auth-relevante Endpoints
**Notes:** Kein toter Code. Browse/Search/Library-Endpoints kommen in Phase 3.

### Frage 4: Response-Cache?

| Option | Description | Selected |
|--------|-------------|----------|
| LMS-Cache (Slim::Utils::Cache) | Namespace 'spoton'. Persistiert über Neustarts, TTL-Support eingebaut. | ✓ |
| Eigener In-Memory-Hash | Perl-Hash mit eigenem TTL. Schneller, geht bei Neustart verloren. | |
| Claude entscheidet | Research-Phase evaluiert. | |

**User's choice:** LMS-Cache
**Notes:** Bewährtes Pattern, Spotty und Qobuz nutzen denselben Ansatz.

---

## Claude's Discretion

- Credential-Eingabemethode: Username/Passwort vs. Zeroconf — Research evaluiert
- Rate-Limiting-Mechanismus: Token Bucket vs. Sliding Window vs. Adaptive
- Concurrency-Limit: Optimaler Wert basierend auf Spotify-Rate-Limits
- Queue-Priorisierung: Ob High/Normal-Trennung nötig ist
- API-Client-Modulstruktur: Monolithisch vs. geschichtet

## Deferred Ideas

None — discussion stayed within phase scope
