# Phase 11: Track History Metadata - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-04
**Phase:** 11-track-history-metadata
**Areas discussed:** Connect-Track Identität, Cache-Miss Re-Fetch, Connect→Browse Replay

---

## Connect-Track Identität

### Persistence-Strategie

| Option | Description | Selected |
|--------|-------------|----------|
| Im Metadata-Cache | Beim Setzen von pluginData('info') zusätzlich den gleichen Datensatz unter spoton_meta_ + md5(connect-URL) cachen. Enthält cover, title, artist UND die echte spotify:track:ID. Wiederverwendet bestehende Cache-Infrastruktur. | ✓ |
| Separater Mapping-Cache | Eigener Cache-Key (z.B. spoton_connect_map_ + md5(connect-URL)) der nur spotify:track:ID speichert. Schlanker, aber zweiter Lookup bei History-Zugriff nötig. | |
| URL umschreiben | Connect-URL in der LMS-Playlist direkt zu spotify://track:ID umschreiben wenn Track-ID bekannt. Radikal, könnte Connect-Mode-Erkennung brechen. | |

**User's choice:** Im Metadata-Cache
**Notes:** Wiederverwendung der bestehenden Cache-Infrastruktur bevorzugt.

### TTL

| Option | Description | Selected |
|--------|-------------|----------|
| 3600s (wie Browse) | Gleicher TTL wie Browse-Metadata. Konsistent, aber nach 1h wieder weg. | |
| 86400s (24h) | 'Was lief gestern Abend?' funktioniert. Spotify Artwork-URLs halten typischerweise länger als 24h. | |
| 604800s (7 Tage) | Maximale Abdeckung für 'was lief letzte Woche?'. Größerer Cache-Footprint, aber LMS-Cache ist ohnehin klein. | ✓ |

**User's choice:** 604800s (7 Tage)
**Notes:** Maximale History-Abdeckung gewünscht.

### Browse TTL Angleichung

| Option | Description | Selected |
|--------|-------------|----------|
| Auch 7 Tage | Einheitlich: alle Track-Metadata 7 Tage. History funktioniert konsistent für Browse und Connect. | ✓ |
| Browse bei 3600s lassen | Browse-Tracks werden häufiger frisch geladen. Nur Connect braucht den langen TTL. | |

**User's choice:** Auch 7 Tage
**Notes:** Einheitlicher TTL für konsistentes History-Verhalten.

---

## Cache-Miss Re-Fetch

### Re-Fetch Strategie

| Option | Description | Selected |
|--------|-------------|----------|
| Async + Placeholder | Sofort Minimal-Metadata zurückgeben, async API-Call starten, Cache füllen, LMS via 'newmetadata' Notification refreshen. | ✓ |
| Nur Placeholder, kein Re-Fetch | Cache-Miss = Pech. Generisches Icon. HIST-04 wäre nicht erfüllt. | |
| Sync Re-Fetch (Blocking) | SimpleSyncHTTP im getMetadataFor-Kontext. Blockiert LMS Event-Loop. | |

**User's choice:** Async + Placeholder
**Notes:** Kurz generisches Icon, dann Refresh via newmetadata Notification.

### Connect Re-Fetch

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, mit Fallback-Feld | Cache-Eintrag enthält 'spotifyUri' Feld. Bei Cache-Miss: Track-ID aus abgelaufenem Eintrag extrahieren. | ✓ |
| Nur Browse Re-Fetch | Re-Fetch nur für Browse-URLs. Connect-Tracks nach Cache-Expiry = endgültig verloren. | |

**User's choice:** Ja, mit Fallback-Feld
**Notes:** Connect-History soll auch nach Cache-Expiry wiederherstellbar sein.

### Throttle

| Option | Description | Selected |
|--------|-------------|----------|
| Ja, ein Request pro Track | Laufender Re-Fetch pro Track-URL tracken via Package-Hash. Verhindert Doppel-Fetches. | ✓ |
| Kein Throttle | Jeder Cache-Miss feuert sofort. Bei 20 abgelaufenen Tracks = 20 gleichzeitige API-Calls. | |
| Claude entscheidet | Implementierungsdetail. | |

**User's choice:** Ja, ein Request pro Track
**Notes:** Rate-Limit-schonend.

---

## Connect→Browse Replay

### Replay-Verhalten

| Option | Description | Selected |
|--------|-------------|----------|
| Transparent übersetzen | spotifyUri aus Cache wird als Browse-URL verfügbar gemacht. Connect-Tracks sind abspielbar. | ✓ |
| Nur Metadaten, kein Replay | History zeigt Artwork korrekt, aber Connect-Tracks nicht erneut abspielbar. | |
| Claude entscheidet | Implementierungsdetail. | |

**User's choice:** Transparent übersetzen
**Notes:** —

### Technische Integration

| Option | Description | Selected |
|--------|-------------|----------|
| Via getMetadataFor + Redirect | getMetadataFor gibt 'playUrl' Feld zurück. ProtocolHandler leitet intern um. | |
| ProtocolHandler::canDirectStream | canDirectStream() für connect-URLs gibt Browse-URL zurück. | |
| Claude entscheidet | Hauptsache Connect-Tracks aus der History sind abspielbar via Browse-Pipeline. | ✓ |

**User's choice:** Claude entscheidet
**Notes:** Technische Integration ist Implementierungsdetail.

### Visuelle Markierung

| Option | Description | Selected |
|--------|-------------|----------|
| Unsichtbar übersetzen | Kein Unterschied für den User. History-Track sieht aus wie Browse-Track. | ✓ |
| Visuell markieren | Kleiner Hinweis im Metadata (type bleibt 'Spotify Connect'). | |

**User's choice:** Unsichtbar übersetzen
**Notes:** Nahtlose Experience bevorzugt.

---

## Claude's Discretion

- Technische Integration der URL-Translation (getMetadataFor Redirect, canDirectStream, oder anderer Mechanismus)
- Placeholder-Inhalt bei Cache-Miss
- Exakte Struktur des Debounce-Hash
- Ob DontStopTheMusic.pm TTL-Bump auch betrifft

## Deferred Ideas

None — discussion stayed within phase scope.
