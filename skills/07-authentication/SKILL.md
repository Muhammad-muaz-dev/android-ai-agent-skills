---
name: authentication
description: Design and implement Android authentication flows including email/password, OAuth, token storage, refresh, logout, session recovery, and secure account state handling.
---

# 07-authentication

Keep authentication behind repository/use-case boundaries. Store tokens only in encrypted storage, never logs or plain SharedPreferences. Model session state explicitly, support refresh and logout failure paths, validate backend contracts, and keep UI screens free of credential persistence logic. Include tests for success, failure, expired-token, and offline/session restoration paths.
