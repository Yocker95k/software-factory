# Stack-Pack: Claude in der App (Anthropic SDK)

Gilt, wenn `docs/stack.md` unter „Stack-Packs" `claude-api` nennt. Für Apps, die zur Laufzeit Claude aufrufen: Recherche, Extraktion aus Texten und Dokumenten, Bewertungen, Chat.

## Anbindung
- Offizielles SDK **`@anthropic-ai/sdk`** (Messages API). Keine OpenAI-kompatiblen Shims, kein eigener HTTP-Client.
- Schlüssel aus `ANTHROPIC_API_KEY` (`.env.local`, nie im Code, nie im Client). Das Modell steht in einer Konstante oder in `CLAUDE_MODEL`, Standard `claude-opus-5-5`.
- Aufrufe nur serverseitig. Antworten, die länger dauern können (Recherche, Chat), werden gestreamt, die UI zeigt Fortschritt.
- Recherche im Web über die serverseitigen Tools `web_search_20260209` und `web_fetch_20260209`. `max_uses` immer setzen.
- Strukturierte Ergebnisse (extrahierte Felder, Schätzungen) über Structured Outputs (`output_config.format`) mit einem Schema, das auch im Code als Zod-Schema existiert. Jede Antwort wird gegen das Schema geprüft, bevor sie gespeichert wird.
- Vor SDK-Code die aktuelle Doku bzw. die Typen in `node_modules/@anthropic-ai/sdk` lesen. API-Namen nicht raten.
- `stop_reason` immer prüfen (`refusal`, `max_tokens`), bevor der Inhalt verwendet wird. Fehler mit den typisierten Fehlerklassen des SDK behandeln.

## Architektur
- Claude steckt hinter einem **Port** (Interface) in `src/lib/ai/`, z. B. `extractListing(text)`, `estimateRenovation(input)`, `research(topic)`. Fachlogik kennt nur den Port, nie das SDK.
- Prompts liegen als Dateien bzw. Konstanten in `src/lib/ai/prompts/`, versioniert. Jede gespeicherte KI-Ausgabe speichert Modell, Prompt-Version und Zeitpunkt mit.
- **Zahlen rechnet der Code, nicht Claude.** Claude liefert Einschätzungen mit Begründung und Quelle (z. B. Kostenspannen), Summen, Gewichtungen und Scores berechnet deterministischer Code.
- Behauptungen mit Quelle: Wo Claude Fakten liefert, gehört die Quellen-URL bzw. Fundstelle zum Ergebnis. Ohne Quelle wird es als Annahme gekennzeichnet.

## Tests in der Factory
- **Kein echter Claude-Aufruf in Tests, Checks oder der Vorschau.** Es gibt einen **Fake-Adapter** für den Port mit festen Antworten (Fixtures unter `src/lib/ai/fixtures/`). Er wird über `AI_PROVIDER=fake` gewählt. `.factory/setup.json` setzt `AI_PROVIDER=fake` für `dev`, `preview` und Tests.
- Tests prüfen: Schema-Validierung, Verhalten bei `refusal`/Fehlern/ungültigem JSON, und dass Ergebnisse korrekt gespeichert und verrechnet werden.
- Prompt-Qualität (trifft die Extraktion, sind Schätzungen plausibel) ist kein Unit-Test. Dafür gibt es optional ein Eval-Skript (`npm run eval`), das nur von Hand mit echtem Schlüssel läuft.

## Kosten und Datenschutz
- Kein Aufruf ohne Nutzeraktion. Keine automatischen Hintergrund-Läufe ohne ausdrückliche Spec.
- Nur die Daten an Claude schicken, die für die Aufgabe nötig sind.
