# Auftragsannahme über ein Website-Formular

Eine Anfrage aus dem Kontaktformular einer Handwerker-Website wird geprüft, gespeichert, per
Sprachmodell eingeordnet und an beide Seiten bestätigt: eine Mail mit der Dringlichkeit im
Betreff an den Betrieb, eine Eingangsbestätigung mit Auftragsnummer an den Kunden.

**Konzept- und Testprojekt** für kleinere Betriebe, kein Produktivsystem. Die zugehörige
Website ist eine Demo-Seite für einen erfundenen Betrieb („Sanitär und Heizung Wolfsbach").
Ein Unternehmen dieses Namens existiert nicht, es wurden keine echten Aufträge bearbeitet.

## Verbundene Dienste

| Dienst | Rolle |
|---|---|
| n8n | Ablaufsteuerung, 13 Nodes plus 5 Sticky Notes mit den Begründungen im Canvas |
| Webhook (n8n) | nimmt `POST /auftrag-neu` mit JSON-Body an |
| OpenAI GPT-4o-mini | ordnet die Anfrage nach Auftragsart und Dringlichkeit ein |
| Airtable | Ablage der Aufträge, zwei Schreibvorgänge pro Anfrage |
| SMTP | zwei E-Mails: an den Meister und an den Kunden |

## Datenfluss

1. **Webhook** empfängt die Formulardaten. `responseMode: responseNode` ist Voraussetzung
   dafür, dass der Workflow selbst bestimmt, wann und mit welchem Statuscode geantwortet wird —
   im Standard antwortet n8n sofort mit „Workflow was started", und die Website könnte nicht
   unterscheiden, ob der Auftrag angekommen ist.
2. **Prüfen und Normalisieren** (Code-Node): Steuerzeichen entfernen, Längen begrenzen, E-Mail
   und PLZ plausibilisieren, Honeypot-Feld auswerten, Auftragsnummer vergeben. Die Prüfung
   läuft serverseitig, obwohl das Formular im Browser ebenfalls validiert — der Webhook ist
   eine offene URL, wer sie kennt, schickt hin was er will.
3. **Switch** teilt in drei Wege: *gültig*, *Spam*, *fehlerhaft*. Ein Switch statt zweier
   IF-Nodes, damit jeder Datensatz garantiert genau einen Weg nimmt und keiner stillschweigend
   verschwindet. Jeder Weg hat seine eigene HTTP-Antwort.
4. **Airtable-Zeile anlegen** und **sofort mit 200 antworten**. Der Besucher wartet auf den
   Datenbankschreibvorgang, nicht auf das Sprachmodell.
5. **KI-Triage** (GPT-4o-mini): Auftragsart aus sechs festen Kategorien, Dringlichkeit aus
   drei Stufen, Zusammenfassung in einem Satz, Begründung, bis zu drei Rückfragen für den
   Rückruf. In den Prompt gehen nur Anliegen, Ort und Wunschtermin — Name, Telefon, E-Mail und
   Adresse bleiben draußen.
6. **Analyse nachtragen**, dann die beiden E-Mails. Beide Nodes laufen mit
   `onError: continueRegularOutput`: scheitert die Mail an den Meister, geht die an den Kunden
   trotzdem raus.

## Drei Entscheidungen, die den Workflow erklären

**Erst speichern, dann anreichern.** Der Airtable-Node steht *vor* dem OpenAI-Aufruf, und die
Website bekommt ihre Antwort, sobald die Zeile in der Tabelle steht — nicht erst nach 2 bis 8
Sekunden Modelllaufzeit und zwei E-Mails. Das Feld `Analyse` wird dabei pessimistisch mit
`Fehlgeschlagen` vorbelegt und erst bei Erfolg überschrieben; ein optimistischer Vorgabewert
könnte einen Ausfall verschleiern. Der Airtable-Node hat einen Fehlerausgang auf eine
500er-Antwort — ohne den würde bei einem Abbruch kein Respond-Node erreicht, der `fetch()` der
Website liefe in einen Timeout und der Besucher sähe minutenlang einen Ladebalken.

**KI nur für die Einordnung.** Keine Preise, keine Termine, keine generierten Kundentexte. Die
Ausgabe läuft über `json_schema` mit `strict: true` statt über `json_object`: das garantiert
nicht nur, *dass* JSON kommt, sondern *welches* — die Kategorien werden von OpenAI erzwungen,
nachträgliches Mapping entfällt. Beide Ausgänge des HTTP-Nodes laufen in denselben Code-Node:
fällt OpenAI aus oder antwortet Unsinn, bleiben Auftragsart und Dringlichkeit **leer** und die
Analyse wird als fehlgeschlagen markiert. Kein geratener Wert, der einen Notfall verdecken kann.

**Spam bekommt 200 statt 400.** Ein Bot, der einen Fehler sieht, probiert Varianten, bis er
durchkommt; einer, der Erfolg sieht, hält sich für fertig und zieht weiter. Die Antwort enthält
eine Auftragsnummer, gespeichert und versendet wird nichts. Das Honeypot-Feld heißt bewusst
nicht `website`: Passwortmanager füllen Felder mit diesem Namen auch aus, wenn sie per CSS
versteckt sind — ein echter Kunde wäre dadurch stillschweigend als Spam ausgesondert worden.

## Offene Punkte

- Der Webhook hat keine Authentifizierung, und `allowedOrigins` steht auf `*`. Für die Übung
  gewollt; produktiv gehört genau eine Domain eingetragen, dazu ein Rate Limit und ein
  kurzlebiges Token aus dem Seitenaufruf. Sonst ist der Betrieb angreifbar als Absender von
  Mails, die er nicht geschrieben hat.
- Kein `retryOnFail` auf den Netzwerk-Nodes. Ein einzelner 429 von Airtable führt direkt in die
  500er-Antwort — genau das, was das Vorziehen des Speicherschritts verhindern sollte. Retry
  und Timeout sind dabei eine Entscheidung, nicht zwei: zwei Wiederholungen verlängern die
  Antwortzeit spürbar.
- Die Auftragsnummer ist Datum plus Zufallszahl aus 9000 Werten. Bei 40 Aufträgen am Tag
  kollidieren zwei davon mit rund 8 Prozent Wahrscheinlichkeit. Airtables Autonumber löst das.
- Gmail als Versandweg ist für eine Übung in Ordnung, für einen Kunden nicht — Bestätigungen
  von einer Gmail-Adresse über einen fremden Server landen im Spam. Dafür gibt es
  Transaktionsversender.

## Import

`order-intake-workflow.json` über *Workflows → Import from File* einlesen. Danach Credentials
für OpenAI (HTTP Header Auth), Airtable und SMTP neu anlegen und die Platzhalter
`appXXXXXXXXXXXXXX` / `tblXXXXXXXXXXXXXX` sowie die `example.com`-Adressen ersetzen. Der
Webhook-Pfad `auftrag-neu` bleibt, der Hostname hängt an der eigenen n8n-Instanz.
