---
name: local-database
description: Design and implement Android local persistence with Room, DataStore, migrations, caching, offline-first sync, and repository boundaries.
---

# 08-local-database

Use Room for relational structured data and DataStore for small key-value preferences. Define entities separately from domain models, use DAOs as data access only, add migrations for schema changes, and keep queries observable where UI must react. Avoid destructive migrations in production unless explicitly accepted. Test DAOs, migrations, and repository cache behavior.
