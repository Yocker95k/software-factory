# Stack-Pack: SQLite + Drizzle ORM

Gilt, wenn `docs/stack.md` unter „Stack-Packs" `sqlite` nennt. Für Apps mit wenigen Nutzern, die lokal oder auf einem einzelnen Server laufen. Die Datenbank ist eine Datei, es gibt keinen Cloud-Dienst.

## Datenbank
- Treiber `better-sqlite3`, ORM **Drizzle** (`drizzle-orm`, `drizzle-kit`). Schema in `src/db/schema.ts`, Zugriff nur über `src/db/client.ts`.
- Pfad der Datei aus `DATABASE_URL` (z. B. `file:./data/app.db`). `data/` steht in `.gitignore`, echte Daten kommen nie ins Repo.
- Schema-Änderungen **nur als Migration**: `npx drizzle-kit generate` erzeugt `drizzle/<nr>_<name>.sql`, die Datei wird mit committet. Migrationen laufen beim Start der App (`migrate()` in `src/db/client.ts`), nie von Hand.
- Beim Öffnen `PRAGMA journal_mode = WAL` und `PRAGMA foreign_keys = ON` setzen.
- Fremdschlüssel mit bewusstem `onDelete`. Indizes auf Spalten, nach denen gefiltert oder sortiert wird.
- Geldbeträge als Ganzzahl in Cent, Zeitpunkte als ISO-8601-Text in UTC. Kein `REAL` für Geld.
- Mehrere zusammengehörige Schreibvorgänge in einer Transaktion (`db.transaction`).
- Listen-Abfragen immer mit `limit`. Keine N+1-Abfragen.

## Zugriff aus der App
- Datenbankzugriff nur serverseitig (Route Handlers, Server Actions, Server Components), nie im Client-Bundle.
- Fachlogik liegt nicht in Drizzle-Abfragen verstreut, sondern in Funktionen unter `src/lib/` (bzw. Repository-Modulen), die Tests direkt aufrufen können.

## Tests in der Factory
- Jeder Test bekommt eine **eigene frische Datenbank** (`:memory:` oder Datei im Temp-Ordner), auf die alle Migrationen angewendet werden. Kein Test teilt Daten mit einem anderen.
- Testdaten über Seed-Funktionen in `src/db/seed/` (deterministisch, ohne Zufall und ohne „jetzt“, Zeit wird übergeben).
- E2E-Tests (Playwright) starten die App mit einer eigenen `DATABASE_URL` im Temp-Ordner und einem Seed. Ein Reset-Endpunkt existiert nur, wenn `NODE_ENV=test`.
- Im PR vermerken, welche Migrationen neu sind.

## Befehle (zusätzlich zu `.factory/setup.json` des Frameworks)
- `npm run db:generate` (`drizzle-kit generate`) · `npm run db:seed` (Beispieldaten für `dev` und Vorschau)
- `dev` und `preview` müssen mit leerer Datenbank starten können: migrieren, dann Seed, wenn die Datenbank leer ist.
