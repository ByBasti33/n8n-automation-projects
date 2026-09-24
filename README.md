# n8n-automation-projects

Zwei n8n-Workflows für kleine Handwerks- und Dienstleistungsbetriebe. Beides Konzept- und
Testprojekte: sie liefen auf meiner eigenen n8n-Instanz gegen echte APIs, aber nie bei einem
Kunden im Einsatz.

## Zur Person

Sebastian Wolfkamp, BWL-Student an der TH Wildau. Ich baue Automatisierungen mit n8n und
Claude Code und arbeite darauf hin, daraus eine eigene Agentur für AI-Automatisierung im
Mittelstand zu machen. Der Ausgangspunkt ist eine schlichte Beobachtung: kleine Betriebe
verlieren Zeit an Bürokram, der sich in klar abgegrenzten Schritten abnehmen lässt — Anfragen
annehmen, Berichte erfassen, Eingänge sortieren und weiterleiten.

## Die zwei Workflows

| Projekt | Auslöser | Was passiert |
|---|---|---|
| [order-intake-workflow](order-intake-workflow/) | `POST` aus einem Website-Formular | Serverseitige Prüfung → Airtable → GPT-4o-mini ordnet Art und Dringlichkeit ein → E-Mail an Betrieb und Kunde |
| [telegram-voice-workflow](telegram-voice-workflow/) | Telegram-Nachricht eines Monteurs | Sprachnachricht → Whisper-Transkription → GPT-4o-mini strukturiert einen Arbeitsbericht → Airtable → Bestätigung per Telegram |

Stand: die Auftragsannahme ist vollständig und getestet, der Telegram-Workflow ein
funktionierender Entwurf mit offenen Punkten. Beide READMEs benennen sie einzeln.

Zum Lesen ist die Auftragsannahme der interessantere Teil — dort stecken die Entscheidungen zu
Fehlerbehandlung, Antwortzeiten und der Frage, was ein Sprachmodell überhaupt tun darf.

## Eingesetzte Dienste

n8n (Webhook, Code, Switch, HTTP Request, Respond to Webhook) · OpenAI Whisper (`whisper-1`)
und GPT-4o-mini · Airtable als Datenablage · Telegram Bot API · SMTP für den Mailversand.

## Zwei Prinzipien, die beide Workflows teilen

**Erst speichern, dann anreichern.** Die Rohdaten gehen in die Tabelle und die Antwort geht
zurück, *bevor* ein Sprachmodell läuft. Fällt die KI danach aus, ist die Anfrage trotzdem
erfasst und als unvollständig markiert, statt verloren.

**Keine KI für Preise, Termine oder Kundentexte.** Das Modell ordnet ein und fasst zusammen,
nichts weiter. Kundenmails sind feste Templates. Eine erfundene Zahl, die als Angebot beim
Kunden landet, ist ein Haftungsrisiko und kein Feature.

## Zu den JSON-Dateien

Es sind Exporte aus n8n, einlesbar über *Workflows → Import from File*. Sie sind anonymisiert:
Airtable-Base- und Tabellen-IDs, E-Mail-Adressen und Credential-Verweise stehen als Platzhalter
drin. Zugangsdaten enthalten n8n-Exporte ohnehin nie — nach dem Import müssen die Credentials
in jedem Node mit Schloss-Symbol neu ausgewählt werden.

Die Workflows sind durchgehend deutsch beschriftet, einschließlich der Sticky Notes im Canvas.
Dort stehen die Begründungen zu den einzelnen Schritten.
