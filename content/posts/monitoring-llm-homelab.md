---
title: "La dashboard che quasi funzionava: monitoring di llama.cpp con Prometheus e Grafana"
subtitle: "Osservabilità su un homelab che arranca, tra bug, metriche, e un modello che non vuole mai andare in pensione"
date: 2026-06-03
lastmod: 2026-06-03
draft: false
author: "Ciro Lucio Tecce"
authorLink: "https://ciroluciotecce.it"
description: "Come ho cercato di mettere in piedi un sistema di monitoring per llama.cpp con Prometheus e Grafana — e cosa ho imparato quando un bug di llama.cpp ha mandato tutto all'aria."
license: ""
images: []

tags: ["llm", "homelab", "monitoring", "llama.cpp", "grafana", "prometheus"]
categories: ["Homelab", "LLM", "DevOps"]

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
---

## Introduzione — il mini PC che non si arrende (e il problema di guardarlo)

Da qualche mese uno dei mini PC nel mio homelab sta venendo letteralmente maltrattato. Gli faccio fare cose per cui non è stato progettato: caricare LLM, processare prompt, generare testo e analizzare immagini. Roba da PC serio, con GPU discreta insomma, ma lui non lo sa e funziona lo stesso.

Eppure, questo piccolo Ryzen 5 4650G Pro con 32GB di RAM e la sua GPU integrata da 7 CU se la cava. Non è un fulmine di guerra — per darvi un'idea, con Qwen3.6 35B A3B in quantizzazione a 4 bit viaggia sui 70-80 token al secondo in prefill e circa 12 token al secondo in generazione. Non proprio da gridare al miracolo, ma è veramente incredibile pensare che una macchina del genere possa far girare un modello con capacità perfettamente sufficienti per piccoli task nel campo dell'automazione domestica, ad esempio.

Con un coding agent leggero come [pi](https://github.com/EarendilWorks/pi) e grazie alla magia della KV cache, piccoli task si gestiscono. Anche per workflow batch più strutturati — tipo quelli che ho implementato con n8n per sfruttare le capacità multimodali del modello — nessun c'è problema e svolge egregiamente il suo lavoro.

Una delle cose che particolarmente mi interessa del mio setup, è l'osservabilità e la capacità di monitorare come, a fronte di diverse configurazioni, feature e modelli, variano le prestazioni del sistema. In un panorama così in rapida evoluzione, dove bisogna destreggiarsi fra diversi tipi di speculative decoding, gestione della quantizzazione dei modelli, quantizzazione della cache e la gestione dell'attenzione, avere la capacità di osservare l'impatto sulle prestazioni e la qualità del modello delle diverse configurazioni è fondamentale: **senza dati, sei cieco**.

> L'osservabilità e i dati sono la chiave per poter prendere qualsiasi decisione in maniera informata. Tutto quello che non è supportato da un numero, un grafico, un'evidenza concreta, è solo un'impressione. E in quanto tale, non può guidare un processo di scelta in ambito tecnico.

Questo post è la storia di come ho cercato di rendere visibile quello che succede dentro llama.cpp. Una storia fatta di metriche, dashboard, Prometheus, Grafana — e di un bug che, almeno per ora, mi ha costretto a mettere tutto in pausa.

---

## Spiegazione — da llama-swap a llama.cpp (e la visibilità perduta)

Fino a poco tempo fa usavo [llama-swap](https://github.com/mut-ex/llama-swap) per gestire i modelli. Per chi non lo conosce, llama-swap è un layer di astrazione sopra llama.cpp che permette di configurare più modelli e gestirne automaticamente il caricamento e lo scaricamento in base alle richieste. Tutto questo molto prima che llama.cpp implementasse la sua router mode nativa.

llama-swap aveva un grande pregio: un'interfaccia web che mostrava le prestazioni di ogni singola chiamata — token elaborati, velocità di prompt processing, velocità di generazione, tempi medi per percentili. Tutto bello, tutto visibile, tutto out of the box.

Ma col tempo ho iniziato a scontrarmi con i suoi limiti. Il primo: il versioning. Non sono mai riuscito a capire esattamente quale versione di llama.cpp fosse impacchettata nelle immagini Docker distribuite da llama-swap. Certo, avrei potuto buildare l'immagine da solo, ma c'era un'altra limitazione più sostanziale: **la gestione delle richieste parallele su modelli differenti**. Per via di un limite implementativo, llama-swap non riesce a emulare la capacità nativa di llama.cpp di gestire più richieste su modelli diversi contemporaneamente.

Nella mia configurazione attuale questa non è una limitazione — con le mie capacità di calcolo, difficilmente riuscirei a saturare anche un solo modello. Ma ho dei nuovi pezzi di hardware in arrivo, e volevo prepararmi per uno scale up.

Così sono passato direttamente a llama.cpp.

E da subito ho sentito la mancanza del monitoring che llama-swap offriva. Tutta quella visibilità evaporata in un colpo solo.

---

## Caso pratico — Prometheus, Grafana, e un bug che non ti aspetti

### La ricerca della visibilità perduta

Così mi sono messo al lavoro. L'obiettivo era semplice: replicare quello che avevo con llama-swap, ma con strumenti standard. Prometheus per la raccolta, Grafana per la visualizzazione. Tutto quello che ho poi documentato in [un gist GitHub](https://gist.github.com/4ae126689db857cc8c2c26f1425c2692.git).

La dashboard finale — almeno nelle intenzioni — doveva mostrare:

- L'andamento della velocità di prefill e generazione per modello attivo, con filtro per modello
- I token processati e il tempo cumulativo di elaborazione
- Il numero di richieste concorrenti attualmente attive
- I modelli caricati in memoria

{{< image src="/grafana-llm-dashboard.webp" caption="La dashboard Grafana — quando funzionava, era bellissima" >}}

### Il problema dello scraper

La prima difficoltà è arrivata subito. llama.cpp espone un endpoint `/metrics`, ma va chiamato filtrato per modello. Questo significa che non puoi semplicemente puntare un Prometheus vanilla su quell'endpoint e aspettarti che funzioni. Serve uno strato intermedio — uno scraper dedicato che gestisca la logica di filtro per modello.

Niente di insormontabile, ma è stato il primo segnale che la strada non sarebbe stata tutta in discesa.

### Il bug che ha fermato tutto

Purtroppo, dopo aver messo in piedi tutta l'infrastruttura — Prometheus che raccoglie, Grafana che visualizza, la dashboard che finalmente prende vita — mi sono scontrato con un muro.

**Un bug noto di llama.cpp** relativo all'esposizione delle metriche impedisce lo scaricamento dei modelli caricati.

In parole povere: ogni chiamata all'endpoint `/metrics` resetta il timer di keep alive del modello. llama.cpp pensa che il modello sia ancora in uso, e quindi non lo scarica quando potrebbe andare in idle.

Le conseguenze sono due, entrambe dolorose:

1. **spreco di risorse** — il modello resta in memoria anche quando nessuno lo sta usando
2. **il routing non funziona** — se il modello caricato è percepito come in uso, llama.cpp non lo scarica per fare posto a un altro modello quando arriva una richiesta diversa

{{< admonition type=warning title="Update 03/06/2026" >}}
Al momento in cui scrivo, questo bug è ancora aperto e sto monitorando gli sviluppi sul repository di llama.cpp. Per ora ho disabilitato lo scraper Prometheus e sono tornato a un monitoring manuale e sporadico — il classico sguardo furtivo a `htop` mentre un modello sta girando.
{{< /admonition >}}

Così, per ora, la soluzione è in pausa. La dashboard è lì, i dati ci sarebbero, ma non posso usarli senza mandare in crash la gestione della memoria del mio piccolo server.

---

## Conclusione — anche una sconfitta è un dato

Alla fine di questa storia, quello che mi porto a casa non è una dashboard funzionante. È un'altra cosa.

Ho imparato che l'osservabilità non è un optional — è il fondamento su cui si basano le decisioni tecniche. Ho imparato che llama.cpp è un progetto fantastico, in evoluzione rapidissima, ma che come ogni software in fase di crescita ha le sue crepe. E ho imparato che anche un esperimento fallito produce dati preziosi — sulla tecnologia, sui limiti dell'hardware, e su cosa serve davvero per far funzionare un sistema.

La dashboard tornerà. Quando il bug sarà risolto, quando avrò il nuovo hardware, o quando troverò un workaround. Ma nel frattempo, questo post rimane — una fotografia di un momento in cui ho cercato di rendere visibile l'invisibile, e ci sono quasi riuscito.

> Alla fine, anche una sconfitta è un dato. E se c'è una cosa che ho imparato dal mio homelab, è che ogni problema è un'occasione per capire qualcosa in più.

Il setup completo è documentato nel [gist](https://gist.github.com/4ae126689db857cc8c2c26f1425c2692.git) — se qualcuno vuole seguire la stessa strada (o aiutarmi a trovare una soluzione al bug), è tutto lì.
