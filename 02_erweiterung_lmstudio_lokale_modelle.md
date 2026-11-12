# Erweiterung: Lokale Modelle mit LM Studio statt Cloud-Provider

![line](images/banner.png)

## Einführung

In der Hauptübung haben Sie Claude Desktop als MCP-Client verwendet – ein Cloud-Modell, das von Anthropic betrieben wird. In dieser Erweiterung ersetzen Sie Claude Desktop durch **LM Studio** mit einem **lokal laufenden Sprachmodell**. Das Ziel ist zu verstehen, was sich verändert, was funktioniert – und was nicht – wenn keine Cloud-Anbindung vorhanden ist.

Diese Erweiterung ist bewusst als **Vergleichsexperiment** aufgebaut: Sie führen dieselben Aufgaben wie in der Hauptübung durch, aber mit lokalen Modellen unterschiedlicher Größe. Die Unterschiede, die Sie beobachten, sind der eigentliche Lerninhalt.

---

## Zusätzliche Lernziele

- **LM Studio** als lokalen MCP-Client einrichten und konfigurieren
- Den CML MCP-Server mit einem **lokal laufenden Modell** verbinden
- Die Auswirkungen der **Modellgröße** auf die Tool-Call-Qualität verstehen
- **Hardwaregrenzen** (VRAM, Context Window) als praktische Einschränkung erleben

---

## Zusätzliche Voraussetzungen

- **LM Studio** installiert (v0.3.x oder neuer, mit MCP-Unterstützung)
  - https://lmstudio.ai
- Mindestens **16 GB RAM** im System (für größere Modelle empfohlen: 32 GB)
- Eine dedizierte GPU ist hilfreich, aber nicht zwingend erforderlich
- **CML ab Version 2.10** mit aktiviertem Dienst **MCPServer** und **Node.js (LTS)** – beides ist aus der Hauptübung bereits vorhanden
- Der Base64-Wert für die CML-Zugangsdaten aus der Hauptübung (Schritt 2b)

---

## Schritt A: LM Studio installieren und einrichten

1. Laden Sie **LM Studio** von https://lmstudio.ai herunter und installieren Sie es.
2. Starten Sie LM Studio und öffnen Sie den Reiter **Discover** (Modelle suchen).
3. Laden Sie das Startmodell herunter – suchen Sie nach:
   ```
   Qwen2.5-3B-Instruct-Q4_K_M
   ```
   Wählen Sie den Anbieter **bartowski** oder **triangle104**. Beide sind zuverlässige Community-Anbieter für GGUF-Modelle.

> **Was bedeutet die Dateibezeichnung?**
> - `Qwen2.5` → Modellname und Version
> - `3B` → 3 Milliarden Parameter (Modellgröße)
> - `Instruct` → für Instruktionen/Chat feinabgestimmt (zwingend erforderlich – das Base-Modell versteht keine Tool-Calls)
> - `Q4_K_M` → Quantisierungsstufe: 4-Bit, gutes Gleichgewicht aus Qualität und Speicherbedarf

---

## Schritt B: Context Window korrekt konfigurieren

Dies ist der kritischste Konfigurationsschritt für die MCP-Nutzung.

1. Laden Sie das Modell in LM Studio.
2. Öffnen Sie die **Modelleinstellungen** (My Models → Qwen2.5-3B → Einstellungen).
3. Setzen Sie die **Context Length** auf **32768** (Startwert).

> **Warum ist das so wichtig?**
> Der CML MCP-Server übergibt seinen gesamten Tool-Katalog als Teil des Prompts. Dieser Katalog umfasst **mehrere zehntausend Tokens**. Der Standard-Context von LM Studio (4096 Tokens) ist viel zu klein – jeder Aufruf schlägt mit einer Fehlermeldung dieser Art fehl:
> ```
> request (38738 tokens) exceeds the available context size (4096 tokens)
> ```
> Die Fehlermeldung nennt die tatsächlich benötigte Anzahl Tokens. Stellen Sie die Context Length so ein, dass sie **deutlich über diesem Wert** liegt – der Tool-Katalog, Ihr Prompt, die Tool-Ergebnisse und die Antwort müssen gemeinsam hineinpassen. Reicht 32768 nicht aus, wählen Sie 65536.
>
> **Achtung bei kleinen Modellen:** `Qwen2.5-3B` unterstützt nativ höchstens 32k Tokens (siehe Tabelle in Schritt D). Ist der Tool-Katalog größer, stößt das 3B-Modell hier an eine harte Grenze – auch das ist ein Ergebnis des Vergleichs.

> **Hinweis zur Hardware:** Eine größere Context Length erhöht den Speicherbedarf deutlich. Wenn Ihr System wenig VRAM hat, lagert LM Studio automatisch mehr Berechnungen auf die CPU aus. Das Modell wird langsamer, bleibt aber funktionsfähig.

---

## Schritt C: CML MCP-Server in LM Studio konfigurieren

LM Studio nutzt dasselbe Konfigurationsformat wie Claude Desktop. Sie verbinden sich also mit demselben, in CML 2.10 integrierten MCP-Server – nur der MCP-Client (und damit das Sprachmodell) ist ein anderer.

1. Öffnen Sie in LM Studio rechts den Reiter **Program** → **Install** → **Edit mcp.json**.
2. Tragen Sie die Konfiguration aus der Hauptübung (Schritt 3) ein. Sie können den Block aus Ihrer `claude_desktop_config.json` unverändert übernehmen:

```json
{
  "mcpServers": {
    "Cisco Modeling Labs (CML)": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://<CML-IP>/mcp",
        "--header",
        "X-Authorization:${CML_AUTH_HEADER}"
      ],
      "env": {
        "CML_AUTH_HEADER": "Basic <base64_encoded_cml_credentials>",
        "NODE_TLS_REJECT_UNAUTHORIZED": "0"
      }
    }
  }
}
```

   > Ersetzen Sie `<CML-IP>` und `<base64_encoded_cml_credentials>` durch Ihre Werte aus der Hauptübung. CLI-Befehle auf den Geräten (`send_cli_command`) funktionieren damit ebenfalls; zu abweichenden Gerätepasswörtern siehe Hauptübung, Schritt 2c.

3. Speichern Sie die Datei. LM Studio startet den MCP-Server automatisch.
4. Prüfen Sie im Chat unter **Integrations**, ob **Cisco Modeling Labs (CML)** erscheint, und aktivieren Sie den Eintrag. Die Liste der Tools des Servers wird dort angezeigt.

> **Tipp zur Fehlerbehebung:** Erscheint der Server nicht oder meldet einen Fehler, prüfen Sie zuerst, ob dieselbe Konfiguration in Claude Desktop funktioniert. Typische Ursachen sind ein fehlendes Node.js (`npx` nicht gefunden), ein falscher Base64-Wert (401) oder ein nicht aktivierter Dienst **MCPServer** in CML.

---

## Schritt D: Vergleichstest mit drei Modellgrößen

Führen Sie die folgenden zwei Aufgaben mit **drei verschiedenen Modellen** durch und notieren Sie Ihre Beobachtungen in der Tabelle am Ende dieses Abschnitts.

### Die drei Modelle

| Modell | Größe | VRAM-Bedarf (Q4_K_M) | Context-Fähigkeit |
|---|---|---|---|
| `Qwen2.5-3B-Instruct-Q4_K_M` | 3B | ~2,5 GB | 32k (nativ) |
| `Qwen2.5-7B-Instruct-Q4_K_M` | 7B | ~5,0 GB | 128k (nativ) |
| `Qwen2.5-14B-Instruct-Q4_K_M` | 14B | ~9,5 GB | 128k (nativ) |


> **Modelle herunterladen:** Suchen Sie in LM Studio nach dem jeweiligen Modellnamen und wählen Sie - wenn möglich - jeweils die `Q4_K_M`-Variante.

### Aufgabe D1: Bestehendes Lab abfragen (lesende Operation)

Geben Sie im LM Studio Chat ein:

```
List all existing labs in Cisco Modeling Labs.
```

Beobachten Sie: Gibt das Modell eine korrekte Liste zurück? Wie lange dauert die Antwort?

### Aufgabe D2: Leeres Lab erstellen (schreibende Operation)

Geben Sie im LM Studio Chat ein:

```
Create an empty lab in Cisco Modeling Labs with the title "TestLab-[IhrName]",
with empty nodes list, empty links list and empty interfaces list.
```

> **Wichtig:** Bei kleinen Modellen müssen Sie die gewünschte Struktur (leere Listen) **explizit im Prompt nennen**. Größere Modelle ergänzen diese Pflichtfelder automatisch – kleinere Modelle nicht.

Überprüfen Sie anschließend in der CML-Weboberfläche, ob das Labor erstellt wurde.

### Beobachtungsprotokoll

Füllen Sie die Tabelle während des Tests aus:

| Kriterium | Qwen2.5-3B | Qwen2.5-7B | Qwen2.5-14B |
|---|---|---|---|
| Labs abfragen: funktioniert? | | | |
| Lab erstellen: funktioniert ohne Hinweis auf leere Listen? | | | |
| Lab erstellen: funktioniert mit explizitem Prompt? | | | | 
| Antwortzeit (geschätzt) | | | | 
| Auffälligkeiten / Fehlermeldungen | | | |

---

## Reflexion: Drei Ebenen der Problemanalyse

Die Erfahrungen aus dieser Erweiterung lassen sich auf drei Ebenen einordnen. Diskutieren Sie die folgenden Fragen:

### Ebene 1 – Infrastruktur: Context Window und Hardware

- Warum schlägt der MCP-Aufruf mit dem Standard-Context (4096 Tokens) fehl, obwohl der eigentliche Prompt nur wenige Wörter hat?
- Welche Konsequenz hat ein größeres Context Window für den VRAM-Bedarf? Was passiert, wenn der VRAM nicht ausreicht?
- In welchen Unterrichtsszenarien ist eine lokale Infrastruktur (kein Internet, datenschutzsensible Umgebungen) ein entscheidender Vorteil gegenüber Cloud-Modellen?

### Ebene 2 – Tool-Routing: Präzision des Prompts

- Warum findet das Modell manchmal keinen passenden Tool-Namen, obwohl der MCP-Server die Tools korrekt registriert hat?
- Welchen Unterschied macht es, ob Sie im Prompt `"create a lab"` oder `"create an empty lab with empty nodes list, empty links list and empty interfaces list"` schreiben?
- Was sagt das über die Rolle des **Prompt Engineerings** beim Einsatz von KI-Tools im Unterricht aus?

### Ebene 3 – Modellkompetenz: Größe und Fähigkeit

- Welche Aufgaben konnte das 3B-Modell zuverlässig ausführen, welche nicht?
- Ab welcher Modellgröße haben Sie eine spürbare Verbesserung beim Erstellen von Labs (schreibende Operationen) beobachtet?
- Warum ist die Modellgröße gerade beim **Tool Calling mit komplexen API-Schemas** besonders relevant – mehr als z.B. beim einfachen Beantworten von Fragen?
- Wie würden Sie als Lehrkraft entscheiden, welches Modell für welchen Anwendungsfall ausreicht?

---

## Vergleich: Cloud-Modell vs. lokales Modell

| Kriterium | Claude Desktop (Cloud) | LM Studio (lokal) |
|---|---|---|
| Einrichtungsaufwand | Gering | Mittel |
| Datenschutz | Daten verlassen das Netz | Alles bleibt lokal |
| Internetabhängigkeit | Ja | Nein |
| API-Kompetenz (Tool Calling) | Sehr hoch | Abhängig von Modellgröße |
| Kosten im Dauerbetrieb | Nutzungsabhängig | Einmalige Hardware |
| Geeignet für Schulungsumgebungen ohne Internet | Nein | Ja |

---

## Aufräumarbeiten (Erweiterung)

- Beenden und löschen Sie alle während dieser Erweiterung erstellten Test-Labs in CML.
- Entladen Sie die Modelle in LM Studio, um VRAM freizugeben: **Chat → Eject the model from memory**.

---

## Autoren und Urheberrecht
- Erweiterung erstellt auf Basis der Hauptübung von: Kareem Iskander
- Erweiterung: Michael Lotter
- Datum: 02/2026, aktualisiert 11/2026 (Anpassung an CML 2.10 mit integriertem MCP-Server und mcp-remote)
- Version: v1.2

![line](images/banner.png)
<p align="center">
<a href="01_lab_cml_mcp_lab.md"><img src="images/previous.png" width="150px"></a>
<a href="README.md"><img src="images/next.png" width="150px"></a>
</p>