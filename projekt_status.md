# Projekt-Status – pii-pipeline
**Stand: 14. Juni 2026**

---

## Erledigtes

### Infrastruktur
- GitHub-Konto + privates Repo `pii-pipeline` angelegt
- SSH-Key (Mac) bei GitHub hinterlegt, passwortlos via Keychain
- Projektstruktur im Repo: `src/`, `notebooks/`, `configs/`, `tests/`
- `.gitignore` für Daten, Modelle, `.DS_Store`, Conda/venv, Notebook-Output
- Google Drive Desktop auf Mac installiert, gemountet (`Meine Ablage/pii-data`)
- Env-Var `$PROJEKT_DATA` gesetzt (`.zshrc`)
- Miniconda installiert, Conda-Env `pii` (Python 3.11) angelegt
- PyTorch mit MPS-Support installiert und verifiziert (`MPS: True`)
- VS Code + Jupyter/Python Extensions installiert

### Architektur-Entscheidungen
- Code → GitHub (Single Source of Truth)
- Daten/Modellgewichte → Google Drive (`Meine Ablage/pii-data`)
- Intensiv genutzte Testdaten → lokal `~/data/`
- Drive-Mount nur in GUI-Session (VNC) zuverlässig nutzbar, nicht per SSH
- GLiNER: komplett auf Mac (kein Colab, kein CUDA nötig)
- Transformer-Training: Colab (GPU); Inferenz/Test: Mac (MPS)
- Entity-Judge: lokal Ollama/qwen2.5 (OpenAI-kompatibler Endpoint)
- Aider: bewusst ausgeklammert, separater Test geplant

### GLiNER-Pipeline (UC1 – Anonymisierung)
- Bestehendes Notebook `v8.17` (von Colab) auf Mac portiert:
  - Device: CUDA → MPS-Fallback
  - LLM-Client: OpenAI Cloud → lokales Ollama (`qwen2.5:14b`, umschaltbar via `ENTITY_JUDGE_MODEL`)
  - `PYTORCH_ENABLE_MPS_FALLBACK=1` gesetzt
  - Input-Pfad: `/Users/olivermaus/data/gliner_input/`
  - HTML-Scrollbar-Fix in `create_score_table_html()`
- Notebook in Repo committed: `notebooks/gliner_anonymization_v8_17_mac.ipynb`
- `requirements.txt` generiert und committed
- Synthetische Testdatei (1 Seite, Deutsch, alle 5 PII-Typen) erstellt
- Erster erfolgreicher Testlauf auf Mac inkl. LLM-Adjudication durchgeführt

### Erkannte PII-Typen (aktiv)
`PERSON`, `ORG`, `LOC`, `PHONE`, `MAIL`

### LLM-Adjudication (Entity-Judge)
- Entscheidungen: `ANON` / `WHITEL` / `REVIEW`
- `ANON` = anonymisieren; `WHITEL` = kein PII, wird entfernt; `REVIEW` = unsicher, wird anonymisiert + dokumentiert
- `exempt:` = direkt anonymisiert ohne LLM-Check

---

## Offene Todos

### Kurzfristig
- [ ] Entity-Judge benchmarken: `qwen2.5:14b` vs. `qwen2.5:32b` (Qualität + Geschwindigkeit)
- [ ] GLiNER mit längerem Dokument testen (Ziel: 50 Seiten)
- [ ] Aider separat testen (anderer Kontext, nicht pii-pipeline)

### Phase 2 – Synthetische Trainingsdaten (UC4)
- [ ] Konzept: Format, Umfang, Labelstruktur
- [ ] Generierung (LLM-basiert, Mac)

### Phase 3 – Transformer
- [ ] Modellarchitektur festlegen
- [ ] Colab-Anbindung: GitHub PAT (Fine-grained, read-only) + `git clone` + Drive-Mount
- [ ] Training (Colab/GPU) → Gewichte → Drive → Inferenz/Test Mac (MPS)
- [ ] MPS-Parität prüfen (`PYTORCH_ENABLE_MPS_FALLBACK=1`)

### Phase 4 – Zusammenfassung + Übersetzung (UC2)
- [ ] Noch nicht begonnen

### Phase 5 – RAG (UC3)
- [ ] Noch nicht begonnen

### Technisches
- [ ] `.gitignore` um `.DS_Store` + `gliner_output/` erweitert (heute erledigt, noch pushen)
- [ ] Notebook nach jedem größeren Fix committen (Disziplin)
