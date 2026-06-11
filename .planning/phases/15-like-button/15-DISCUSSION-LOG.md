# Phase 15: Like Button - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-11
**Phase:** 15-like-button
**Areas discussed:** Kontextmenü-Integration, Liked-State-Check-Timing, Scope-Upgrade UX, Feedback nach Aktion

---

## Kontextmenü-Integration

**Vorrecherche:** User bat um Recherche wie Qobuz und TIDAL Plugins Like/Favorites implementieren. Ergebnis: Qobuz nutzt trackInfoMenu()-Hook mit dynamischem Label, on-demand State-Check, showBriefly-Feedback. Tracks haben KEIN favorites_url oder inline-Aktionen.

| Option | Description | Selected |
|--------|-------------|----------|
| Qobuz-Pattern folgen | trackInfoMenu()-Hook, dynamisches Label, on-demand State-Check | ✓ |
| items-Array in _trackItem | Like/Unlike neben Artist View / Album View im Browse-Kontext | |
| Beides | trackInfoMenu UND items-Array | |

**User's choice:** Qobuz-Pattern folgen
**Notes:** Etabliertes LMS-Pattern, Konsistenz mit dem Ökosystem war ausschlaggebend

| Option | Description | Selected |
|--------|-------------|----------|
| Nur Tracks | Fokus auf LIB-01 bis LIB-05, type=tracks | ✓ |
| Tracks + Alben | PUT/DELETE /me/library für beide Typen | |

**User's choice:** Nur Tracks

| Option | Description | Selected |
|--------|-------------|----------|
| Like / Unlike | Kurz, Spotify-nah | ✓ |
| Zu Liked Songs / Aus Liked Songs entfernen | Beschreibender, Qobuz-ähnlich | |
| Claude entscheidet | Researcher prüft Passung zu bestehenden Labels | |

**User's choice:** Like / Unlike

---

## Liked-State-Check-Timing

| Option | Description | Selected |
|--------|-------------|----------|
| Kurzer TTL-Cache (60s) | 60s Cache nach erstem Check, konsistent mit CLAUDE.md Library-Cache-TTL | ✓ |
| Kein Cache | Jedes Menü-Öffnen macht frischen API-Call | |
| Session-Cache | State bleibt bis Player-Wechsel gecacht | |

**User's choice:** Kurzer TTL-Cache (60s)

| Option | Description | Selected |
|--------|-------------|----------|
| Sofort invalidieren | Cache-Löschung nach Like/Unlike, Qobuz-Pattern | ✓ |
| Optimistisch updaten | Lokales Cache-Update ohne API-Roundtrip | |

**User's choice:** Sofort invalidieren

---

## Scope-Upgrade UX

| Option | Description | Selected |
|--------|-------------|----------|
| Nur App-Config + Graceful Fallback | Scopes in Dashboard hinzufügen, Keymaster liefert automatisch, 403-Fallback | |
| cacheSchemaVersion-Bump erzwingen | Bump 2→3, Cache + Credentials löschen | |
| Claude entscheidet | Researcher klärt Keymaster-Scope-Verhalten | ✓ |

**User's choice:** Claude entscheidet
**Notes:** Technische Frage die vom Researcher geklärt werden soll — ob Keymaster-Tokens automatisch neue Scopes bekommen

---

## Feedback nach Aktion

| Option | Description | Selected |
|--------|-------------|----------|
| showBriefly + grandparent | Kurze Bestätigung, dann zurück ins vorherige Menü (Qobuz-Pattern) | ✓ |
| Inline-Label-Wechsel | Menü bleibt offen, Label wechselt | |
| Nur zurücknavigieren | Kein explizites Feedback | |

**User's choice:** showBriefly + grandparent

| Option | Description | Selected |
|--------|-------------|----------|
| Fehlermeldung im Menü | showBriefly mit Fehlertext, 403 mit Berechtigungshinweis | ✓ |
| Stille Fehler + Log | Nur Logging, kein sichtbares Feedback | |
| Claude entscheidet | Researcher/Planner bestimmen Fehlerbehandlung | |

**User's choice:** Fehlermeldung im Menü

---

## Claude's Discretion

- Scope-Upgrade-Mechanismus: Ob Keymaster-Tokens automatisch neue Scopes bekommen oder Re-Auth nötig ist
- trackInfoMenu-Registrierung: Genaues LMS API Pattern (registerInfoProvider? Callback-Signatur?)
- Cache-Key-Format für Liked-State
- Request-Body-Format für /me/library Endpoints

## Deferred Ideas

- Album-Like (PUT /me/library mit type=albums) — eigene Phase
- Connect Now-Playing Like (LIB-06) — bereits als Future Requirement erfasst
