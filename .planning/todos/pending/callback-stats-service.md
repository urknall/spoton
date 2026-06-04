---
title: Callback-Webseite für anonyme Install-Statistik
priority: low
source: user-idea
created: 2026-06-04
context: Phase 9 UAT Diskussion
depends_on: Extended Quota bei Spotify, PKCE-Auth-Flow
---

Idee: Redirect URI auf eigene Webseite setzen (z.B. https://spoton.example.com/callback),
die bei PKCE-Auth als Callback dient und anonyme Nutzungsstatistik erfasst.

Voraussetzungen:
- PKCE-Auth-Flow als Alternative/Ergänzung zu Keymaster (Keymaster triggert keinen Redirect)
- Extended Quota bei Spotify (Developer Mode = max 5 User)
- DSGVO-konforme Datenerfassung (Consent, Privacy Policy)

Pragmatischer Einstieg: Callback-Seite zeigt "Authentifizierung erfolgreich" + anonymer Counter serverseitig.

Zeithorizont: Jenseits v1.x.
