# Handoff

## Goal

Aggiungere supporto multilingua (italiano + inglese) al personal site e pushare su main.

## Progress

- ✅ Articolo completato e pubblicato in `content/posts/homelab-deploy-template.md` (draft: false)
- ✅ Due foto inserite: HP ProDesk iniziale e mini rack attuale (`static/homelab-*`)
- ✅ Diagramma Mermaid del flusso di deploy (3 workflow indipendenti in parallelo)
- ✅ Configurazione sito: `[params.author]`, `[menu]` per Posts/Tags, compatibilità Hugo 0.157
- ✅ Template overrides in `layouts/` per deprecazioni Hugo 0.156+ (`.Site.Author` → `.Site.Params.author`, `.Sites.First` → `.Sites.Default`, `.Site.IsMultiLingual` → `hugo.IsMultilingual`)
- ✅ **Multilingua**: italiano (default) + inglese
  - `config.toml`: sezione `[languages]` con `[languages.it]` e `[languages.en]`
  - `content/_index.md`: homepage in italiano
  - `content/en/_index.md`: homepage in inglese
  - `content/posts/homelab-deploy-template.md`: post in italiano (default)
  - `content/en/posts/homelab-deploy-template.md`: post tradotto in inglese
  - Language selector funzionante in header (Italiano/English)
  - Menu tradotto: "Articoli"/"Posts", "Tag"/"Tags"
  - Disclaimer/subtitle tradotti per lingua
  - OG metadata e JSON-LD corretti per lingua
- ✅ Build verificato: Hugo 0.157.0, 20 pagine IT + 16 pagine EN

## Key Decisions

- **Tono**: personale, tra pari ("altri nerd come me"), niente riferimenti diretti al lettore ("provalo", "se generi...")
- **Struttura**: four-part framework (introduzione → spiegazione → caso pratico → conclusione), con focus sull'apprendimento nella chiusa
- **Diagramma**: workflow indipendenti in parallelo, non sequenziali (corretto dopo review)
- **Compatibilità Hugo**: override locali dei partial del tema LoveIt per Hugov0.157 — i template sovrascritti sono in `layouts/` e seguono la stessa struttura del tema
- **Lingua predefinita**: italiano (defaultContentLanguage = "it") — coerenza con autore italiano e contenuto principale
- **Content directory**: `contentDir` esplicito per inglese (`content/en`) per garantire corretta risoluzione di `.Site.LanguageCode`
- **Config condivisa**: `[params.author]`, `[params.header.title]`, `[params.social]` al top level (ereditate da tutte le lingue); subtitle, disclaimer, menu in `[languages.xx.params]` e `[languages.xx.menu]`

## Files Changed

- `config.toml` — refactor multilingua: `defaultContentLanguage`, `[languages]`, menu e params per-lingua
- `content/posts/homelab-deploy-template.md` — post in italiano (167 righe, ~1000 parole) — invariato
- `content/_index.md` — tradotto in italiano
- `content/en/_index.md` — homepage in inglese (nuovo)
- `content/en/posts/homelab-deploy-template.md` — post tradotto in inglese (nuovo)
- `static/homelab-hp-prodesk.webp` — foto primo server
- `static/homelab-mini-rack.jpeg` — foto rack attuale
- `layouts/partials/init.html` — fix `.Sites.First` → `.Sites.Default`
- `layouts/partials/header.html` — fix `.Site.IsMultiLingual` → `hugo.IsMultilingual`
- `layouts/partials/head/seo.html` — fix `.Site.Author.xxx` → `.Site.Params.author.xxx`
- `layouts/partials/footer.html` — stesso fix author
- `layouts/partials/rss/item.html` — stesso fix author
- `layouts/posts/single.html` — stesso fix author
- `layouts/_default/summary.html` — stesso fix author
- `layouts/posts/rss.xml` — stesso fix author
- `layouts/taxonomy/rss.xml` — stesso fix author
- `layouts/index.rss.xml` — stesso fix author
- `layouts/shortcodes/version.html` — fix `.Site.IsMultiLingual` → `hugo.IsMultilingual`

## Current State

- Sito builda e serve correttamente su Hugo 0.157.0
- Multilingua funzionante: italiano root `/`, inglese su `/en/`
- Post disponibile in entrambe le lingue:
  - IT: `/posts/homelab-deploy-template/`
  - EN: `/en/posts/homelab-deploy-template/`
- Language selector nel menu naviga correttamente tra le versioni
- Repository template pubblico su https://github.com/Cirius1792/hs-service-template

## Blockers / Gotchas

- **Hugo 0.157 vs LoveIt theme**: il tema usa API deprecate (`.Site.Author`, `.Sites.First`, `.Site.IsMultiLingual`). Gli override in `layouts/` risolvono, ma vanno mantenuti finché il tema non aggiorna i suoi template. Se si aggiorna il submodule del tema, verificare che gli override siano ancora necessari.
- **contentDir esplicito**: per la lingua non predefinita (inglese) è necessario `contentDir = "content/en"` in `[languages.en]`. Senza, Hugo tratta `content/en/` come sezione del sito italiano invece che come directory contenuti inglesi.
- **Deploy**: per pubblicare, fare `git push` sul branch `main` (GitHub Action deploya automaticamente su Firebase). Al momento si è su `page/homelab`.

## Next Steps

1. **Push su main**: merge del branch `page/homelab` in `main` per deployare il post e il multilingua
2. (Futuro) Tradurre altri contenuti o aggiungere nuove lingue
3. (Futuro) Verificare sitemap e SEO multilingua dopo il deploy
