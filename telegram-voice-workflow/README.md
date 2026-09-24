# Arbeitsbericht per Telegram-Sprachnachricht

Ein Monteur schickt nach dem Einsatz eine Sprachnachricht an den Firmen-Bot und bekommt
Sekunden später eine Bestätigung zurück. Dazwischen entsteht aus der formlosen Notiz ein
strukturierter Arbeitsbericht in Airtable: Kunde, ausgeführte Arbeiten, Material, Stunden,
Status, offene Punkte.

Der Gedanke dahinter ist banal: Berichte schreiben passiert abends oder gar nicht. Zwanzig
Sekunden diktieren im Auto passiert.

**Konzept- und Testprojekt** für kleinere Betriebe, kein Produktivsystem. Der Workflow lief auf
meiner eigenen n8n-Instanz gegen echte APIs, war aber nie bei einem Kunden im Einsatz — und er
ist, anders als die Auftragsannahme im Nachbarordner, ein unfertiger Stand. Was fehlt, steht
unten.

## Verbundene Dienste

| Dienst | Rolle |
|---|---|
| n8n | Ablaufsteuerung, 10 Nodes plus Dokumentation im Canvas |
| Telegram Bot API | Eingang der Nachricht, Download der Sprachdatei, Rückmeldung |
| OpenAI Whisper (`whisper-1`) | Transkription, Sprache fest auf Deutsch gesetzt |
| OpenAI GPT-4o-mini | übersetzt die Notiz in ein festes JSON-Format |
| Airtable | Ablage der Berichte |
| SMTP | Bericht an die Betriebsleitung (derzeit deaktiviert, siehe unten) |

## Datenfluss

1. **Telegram Trigger** empfängt eine Nachricht an den Bot. Der Bot wird bei `@BotFather`
   angelegt, die Monteure starten ihn einmal mit `/start`.
2. **IF-Weiche** prüft, ob eine Sprachnachricht vorliegt. Textnachrichten sind ebenfalls
   erlaubt und laufen an der Transkription vorbei direkt zu Schritt 4 — wer gerade nicht reden
   kann, tippt.
3. **Sprachdatei herunterladen und transkribieren**: die `.oga`-Datei kommt als Binärfeld vom
   Telegram-Server und geht per Multipart-Upload an die Whisper-API.
4. **Bericht-Text normalisieren** (Code-Node) führt beide Zweige zusammen, liest den
   Monteursnamen aus dem Telegram-Profil und vergibt eine Berichtsnummer. Der Node greift dafür
   nicht blind auf `$json` zu, sondern gezielt auf den Trigger-Node, und entscheidet an
   `message.voice`, aus welchem Zweig der Text kommt. An zusammenlaufenden Zweigen ist das der
   Unterschied zwischen reproduzierbar und zufällig.
5. **GPT-4o-mini strukturiert** die Notiz: Kunde und Objekt, Auftrag, durchgeführte Arbeiten,
   Material, Zeitaufwand, Status aus drei festen Werten, offene Punkte, Mängel, Zusammenfassung.
   Der Prompt verbietet ausdrücklich, Angaben zu erfinden — was nicht gesagt wurde, wird als
   „Nicht angegeben" ausgewiesen. Umgangssprache wird in Berichtssprache übersetzt, nicht
   ausgeschmückt.
6. **Airtable-Zeile schreiben**, dann **Telegram-Bestätigung** mit Berichtsnummer, Kunde,
   Stunden und Status zurück an den Monteur. Er soll sehen, *was* angekommen ist, nicht nur
   *dass* etwas angekommen ist — sonst glaubt niemand der Automatisierung.

## Offene Punkte

Der Workflow läuft, ist aber nicht fertig. Die drei ersten Punkte sind echte Datenverluste:

- Der **E-Mail-Node an die Betriebsleitung ist deaktiviert**. Er hängt mitten in der Kette
  zwischen Airtable und der Telegram-Bestätigung: der Monteur bekommt sein Häkchen, ohne dass
  im Büro jemand den Bericht gesehen hat.
- Das Feld **`Zeitaufwand` wird als feste `0`** in die Tabelle geschrieben, während alle
  Nachbarfelder Expressions sind. Das Modell extrahiert die Stunden sauber, der Wert landet nie
  in Airtable — ausgerechnet das Feld, das später die Rechnung trägt.
- Der OpenAI-Aufruf **erzwingt kein Antwortformat**. Der Prompt bittet um valides JSON, statt es
  über `response_format` vorzuschreiben; abgesichert ist das nur durch einen Fallback im
  nachgelagerten Code-Node. Die Auftragsannahme im Nachbarordner macht es mit `json_schema` und
  `strict: true` richtig.
- Kein Timeout und kein Retry auf den Netzwerk-Nodes. Kritisch ist Whisper: eine lange
  Sprachnachricht kann den Aufruf hängen lassen.
- **Keine Prüfung auf berechtigte Absender.** Ein Telegram-Trigger fühlt sich nicht wie eine
  offene URL an, ist aber genau das: jede fremde Nachricht löst einen Whisper- und einen
  GPT-Aufruf auf eigene Kosten aus und legt eine Airtable-Zeile an. Für einen echten Einsatz
  gehört eine Allowlist der erlaubten Chat-IDs davor.

## Import

`telegram-voice-workflow.json` über *Workflows → Import from File* einlesen. Danach Credentials
für Telegram, OpenAI (HTTP Header Auth) und Airtable neu anlegen und die Platzhalter
`appXXXXXXXXXXXXXX` / `tblXXXXXXXXXXXXXX` sowie die `example.com`-Adressen ersetzen. Welche
Felder die Airtable-Tabelle braucht, steht in den Notes des Airtable-Nodes.

Der Telegram-Trigger meldet seine Webhook-Adresse aktiv bei Telegram an. Nach einem Umzug auf
eine andere Instanz muss er einmal deaktiviert und wieder aktiviert werden, sonst liefert
Telegram weiter an die alte Adresse.
