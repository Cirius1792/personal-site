# Handoff

## Goal

Scrivere un nuovo post sul sito personale (ciroluciotecce.it) che racconta l'esperienza di mettere in piedi un sistema di monitoraggio per llama.cpp con Prometheus e Grafana.

## Progress

- Salvate le note dell'utente in `notes/monitoring-llama-cpp.md`
- Letto il post esistente `content/posts/homelab-deploy-template.md` per inferire tone of voice
- Lette le skill `technical-writing` e `personal-site-post`
- Copiata l'immagine della dashboard Grafana da `/home/clt/ObsidianVault/...` a `static/grafana-llm-dashboard.webp`
- Scritta versione IT: `content/posts/monitoring-llm-homelab.md`
- Scritta versione EN: `content/en/posts/monitoring-llm-homelab.md`
- Build Hugo verificata (180ms, nessun errore, `lang="it"` e `lang="en"` corretti)

## Key Decisions

- **Struttura**: usato il framework four-part del skill technical-writing (introduzione → spiegazione → caso pratico → conclusione)
- **Tono**: matchato lo stile del post esistente — conversazionale, autoironico, onesto sui limiti tecnici, con blockquote per concetti chiave
- **Bug**: incluso un `{{< admonition >}}` per segnalare lo stato ancora aperto del bug di llama.cpp, con data dell'update
- **Immagine**: rinominata da nome originale lungo a `grafana-llm-dashboard.webp` per consistenza con le convenzioni del progetto

## Files Changed

- `notes/monitoring-llama-cpp.md` — nuovo file, appunti dell'utente parola per parola
- `static/grafana-llm-dashboard.webp` — nuova immagine, copiata dalla Obsidian Vault
- `content/posts/monitoring-llm-homelab.md` — nuovo post in italiano
- `content/en/posts/monitoring-llm-homelab.md` — nuovo post in inglese

## Current State

Entrambe le versioni (IT e EN) sono scritte e compilano correttamente. Il post racconta la storia del passaggio da llama-swap a llama.cpp, la costruzione del sistema di monitoring e il bug che ha bloccato tutto. I post sono in stato `draft: false` — pronti per revisione e pubblicazione.

## Blocker / Gotchas

- Il post è già completo e compilato, ma l'utente vuole fare **refinement** nella prossima sessione
- L'immagine della dashboard è referenziata ma non se ne conosce il contenuto visivo (il modello non supporta immagini) — da verificare che sia la dashboard giusta e che la didascalia sia appropriata

## Next Steps

1. **Refinement del post** — leggere entrambe le versioni, rivedere tono, chiarezza, eventuali imprecisioni tecniche
2. **Verifica immagine** — controllare che `grafana-llm-dashboard.webp` sia la dashboard Prometheus/Grafana corretta
3. **Pubblicazione** — dopo refinement, `git add -A && git commit -m "feat: post monitoring llama.cpp" && git push`
