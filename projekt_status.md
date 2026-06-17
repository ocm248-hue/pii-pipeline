# Projekt-Status – pii-pipeline & Aider-Exkurs
**Stand: 14. Juni 2026 (Fortsetzung)**

---

## Erledigt: Infrastruktur

- GitHub-Repo `pii-pipeline` (privat), SSH-Key passwortlos via Keychain
- Projektstruktur: `src/`, `notebooks/`, `configs/`, `tests/`; `.gitignore` für Daten/Modelle/.DS_Store/Conda
- Google Drive Desktop gemountet (`Meine Ablage/pii-data`), Env-Var `$PROJEKT_DATA`
- Miniconda + Conda-Env `pii` (Python 3.11), PyTorch mit MPS verifiziert
- VS Code installiert (Python + Jupyter Extensions) — Notebook-IDE der Wahl statt JupyterLab
- whisper.cpp gebaut (`~/whisper.cpp`), Modelle `large-v3` + `large-v3-turbo` geladen
- SMB-Freigaben eingerichtet: `pii-pipeline` (Repo) und `~/inbox/transkripte` (Audio-Ablage), in Windows als Netzlaufwerk verbunden (Login: IP-Adresse statt Hostname, da Hostname-Auflösung unzuverlässig)
- Ollama läuft mit Custom-Profil: `OLLAMA_CONTEXT_LENGTH=32768 OLLAMA_NUM_PARALLEL=1 OLLAMA_MAX_LOADED_MODELS=1 ollama serve` (eigene SSH-Session)

## Architektur-Entscheidungen (bestätigt)

- Code → GitHub; Daten/Gewichte → Google Drive; intensiv genutzte Testdaten → lokal `~/data/`
- Drive-Mount nur in GUI/VNC-Session zuverlässig, nicht per SSH (Apple File Provider, TCC-Restriktion)
- GLiNER: komplett auf Mac (MPS), kein Colab nötig
- Transformer: Training Colab (GPU), Inferenz/Test Mac (MPS) — noch nicht begonnen
- Entity-Judge: lokal Ollama/qwen2.5 (OpenAI-kompatibler Endpoint, Client-Lib `openai`, aber 100% lokal/kein Cloud-Traffic)
- Ein Modell zur Zeit geladen (kein gleichzeitiges Laden von Coder + Judge/Summarizer — war nie anders geplant)

## Erledigt: GLiNER-Pipeline (UC1 – Anonymisierung)

- Bestehendes Notebook v8.17 (vormals Colab) auf Mac portiert: Device→MPS, LLM-Client→lokales Ollama, MPS-Fallback-Env gesetzt
- HTML-Score-Tabelle: Scrollbar-Fix im Notebook-Source eingebaut (`overflow-x:auto`-Wrapper in `create_score_table_html()`)
- Synthetische Testdatei erstellt und erfolgreich verarbeitet (PERSON/ORG/LOC/PHONE/MAIL erkannt, LLM-Adjudication lief)
- LLM-Entscheidungslogik verifiziert: `ANON`=anonymisieren, `WHITEL`=kein PII/entfernt, `REVIEW`=unsicher/wird anonymisiert+dokumentiert, `exempt:`=ohne LLM-Check direkt anonymisiert
- Alles committed & gepusht (Notebook, `requirements.txt`, `.gitignore`-Ergänzungen)

## Exkurs: Whisper + Zusammenfassung (UC2) — TEILWEISE GELÖST, OFFENE PUNKTE

### Whisper-Transkription
- Modell `large-v3` → Halluzinations-Loop bei längeren Aufnahmen (Endlosschleife gleicher Satz), auch ohne Stille an der Stelle — **ungeklärte Ursache**, nicht weiterverfolgt
- Modell `large-v3-turbo` + Parameter funktioniert grundsätzlich:
  ```
  --entropy-thold 2.4 --temperature-inc 0.2 --max-context 64
  ```
- `-otxt` ohne `--no-timestamps`: korrekt vollständig, aber **keine Interpunktion/Großschreibung** (reiner Wortfluss)
- `-otxt` mit `--no-timestamps`: gute Interpunktion/Großschreibung, ABER **Informationsverlust** (nachweislich fehlende Sätze trotz hörbarem Inhalt) — **NICHT AKZEPTABEL als finale Lösung**
- **OFFEN / NÄCHSTER TEST:** `-osrt` (Untertitel-Format mit Timestamps) auf Vollständigkeit prüfen, dann Timestamps per Python-Nachbearbeitung entfernen → Hypothese: vollständig UND sauber
- **Anforderung für Aider-Spezifikation:** Vollständigkeit hat oberste Priorität, kein Informationsverlust akzeptabel

### Zusammenfassung/Protokoll (qwen2.5:32b)
- Modellwahl bestätigt: `qwen2.5:32b`, 32k Kontext ausreichend (Test-Transkript 41.000 Zeichen ≈ 10.000 Tokens)
- **Erkenntnis:** Prompt mit Begriff „Zusammenfassung" führt zu starker Kompression (z.B. ganzes Gespräch in 4 Sätzen) → **Informationsverlust nicht akzeptabel**
- **Lösung (Ansatz, noch nicht final verifiziert):** Prompt umformulieren auf „detailliertes/vollständiges Protokoll" statt „Zusammenfassung", explizite Vollständigkeits-Anweisung, ggf. `num_predict` erhöhen
- Conversational-Test-Setup (Python-Script `/tmp/chat.py`, Chat-Loop gegen Ollama-API) eingerichtet für iteratives Prompt-Tuning
- Quantisierung von qwen2.5:32b für diese Aufgabe noch nicht final entschieden (Diskussion offen)

## Aider / Coding-Agent — Konzeptklärung (noch nicht umgesetzt)

### Modellentscheidung
- Für Aider/Coding: **qwen3-coder:30b**, NICHT qwen2.5 (Verwechslungsgefahr — zwei verschiedene Modelle für zwei verschiedene Rollen!)
- Architektur: MoE (30B total, ~3B aktiv pro Token)
- Quantisierungsentscheidung: **Q8_0** (~32 GB) gewählt als Ziel — passt sicher in 64 GB inkl. KV-Cache-Puffer; FP16 (~61 GB) zu riskant, lässt keinen Platz für Kontext
- Noch zu pullen: `ollama pull qwen3-coder:30b-a3b-q8_0` (exakten Tag-Namen nach Pull in `ollama list` verifizieren)
- Hinweis: `qwen3:30b` (Basis-Modell, NICHT qwen3-coder) hat bekanntes Problem mit unkontrollierbarem Thinking-Modus über API — betrifft qwen3-coder vermutlich nicht (separates Modell, kein Thinking-Modus)

### Rollenverständnis Aider ↔ LLM (geklärt)
- Aider = Orchestrator: Kontext/Repo-Map, Prompt-Bau, Diff-Parsing, Dateisystem-Schreiben, Git-Commit, optionaler Test/Lint-Loop
- LLM = Code-Generator, kennt kein Git/Dateisystem
- Aider trifft KEINE fachliche Qualitätsbewertung und KEINE Freigabe-Entscheidung — das bleibt beim Menschen
- Tests werden nicht automatisch von Aider erfunden, nur ausgeführt wenn konfiguriert (`--auto-test`, `--lint-cmd`, `--test-cmd`)

### Vereinbarte Leitplanken für Aider-Einsatz (Vorschlag, noch zu bestätigen)
1. Kleine, klar spezifizierte Einzelschritte statt großer Aufgaben
2. Test-Driven-Ansatz wo möglich (Tests vorgeben statt vom LLM erfinden lassen)
3. `--auto-test` an echte Tests koppeln
4. Jeder Commit wird von dir selbst reviewt vor Weiterarbeit

### Review-Pipeline-Konzept (beschlossen, Umsetzung offen)
Vergleichstest in zwei Schritten:
- **Option A (zuerst):** Lokaler Reviewer — generierter Code von `qwen2.5:32b-q8` (oder `qwen2.5:72b-q4`, Kontext-Stabilität noch zu testen) analysieren lassen (Bugs, Sicherheit, Stil, Verbesserungen)
- **Option B (danach):** Cloud-Reviewer — derselbe Code/Diff an Claude Sonnet, gleicher Prompt für Vergleichbarkeit
- **Vergleich:** Übereinstimmungen/Unterschiede zwischen lokalem und Cloud-Review auswerten
- **OFFEN:** einheitlicher Review-Prompt (Kriterien Korrektheit/Sicherheit/Wartbarkeit/Performance/Stil) muss noch formuliert werden, bevor Schritt A/B starten

### Ursprüngliche Aufgabenideen (Eignung geprüft, Empfehlung: NICHT als Erstaufgabe)
- Aufgabe 1 „Lokales RAG-System mit UI + Autostart": als ZU KOMPLEX für ersten Aider-Test eingestuft (Multi-Komponenten: Vektor-DB, Embedding, LLM-Anbindung, UI, Prozessorchestrierung)
- Aufgabe 2 „Watcher → Whisper → Protokoll → RAG": hängt von Aufgabe 1 ab UND von noch ungelösten Fachfragen (Whisper-Vollständigkeit, Protokoll-Prompt) — daher noch nicht spezifizierbar
- **Empfehlung (noch nicht final entschieden):** Erster Aider-Test sollte kleinerer, klar spezifizierter Baustein sein, z.B. das Whisper-Nachbearbeitungs-Script (SRT→sauberer Text ohne Verlust), NACHDEM die Fachfrage selbst gelöst ist

---

## Offene Todos (priorisiert)

### Unmittelbar als Nächstes
- [ ] `-osrt`-Test für Whisper-Vollständigkeit durchführen + Timestamp-Stripping-Script
- [ ] Protokoll-Prompt (qwen2.5:32b) auf Vollständigkeit finalisieren und verifizieren
- [ ] Einheitlichen Review-Prompt für Aider-Code-Review formulieren (Kriterien festlegen)
- [ ] `qwen3-coder:30b-a3b-q8_0` pullen, in Aider konfigurieren
- [ ] Ersten Aider-Task sauber spezifizieren (kleiner Baustein, NICHT die RAG-Aufgabe direkt)

### Phase 1 GLiNER (Rest)
- [ ] Entity-Judge benchmarken: qwen2.5:14b vs. 32b
- [ ] Test mit längerem Dokument (Ziel 50 Seiten)

### UC2 Zusammenfassung/Übersetzung (laufend, s.o.)
- [ ] Watcher-Mechanismus für `~/inbox/transkripte` (Polling, Stabilitäts-Check „Datei fertig geschrieben")
- [ ] Output-Konvention (wohin mit Ergebnis, Archivierung Original)
- [ ] Sprechererkennung: bewusst zurückgestellt (whisperx/pyannote als Optionen identifiziert, nicht entschieden ob nötig)

### Phase 2 – Synthetische Trainingsdaten (UC4) — nicht begonnen
### Phase 3 – Transformer-Training (Colab) — nicht begonnen, inkl. Colab-PAT-Setup
### Phase 5 – RAG (UC3) — Konzept zurückgestellt bis Aider-Eignung geklärt; ursprüngliche Aufgabenidee s.o.

### Technische Hygiene
- [ ] `requirements.txt` regelmäßig aktuell halten (`pip freeze`)
- [ ] Notebook nach jedem Fix committen (Disziplin)
- [ ] Whisper-Parameter + Findings ins Cheatsheet übernehmen (bisher nur in diesem Status-Dokument)
