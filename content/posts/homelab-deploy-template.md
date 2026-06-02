---
title: "Da SSH a git push: come ho automatizzato i deploy sul mio homelab"
subtitle: "Un template Cookiecutter per trasformare git push in container running — senza toccare un terminale"
date: 2026-06-02
lastmod: 2026-06-02
draft: false
author: "Ciro Lucio Tecce"
authorLink: "https://ciroluciotecce.it"
description: "Come ho automatizzato il deploy di servizi sul mio homelab usando Cookiecutter, cruft, e CI/CD."
license: ""
images: []

tags: ["homelab", "devops", "automation", "cookiecutter"]
categories: ["Homelab", "DevOps"]

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
---

## Introduzione — il pesce rosso e il mini rack

Un paio di anni fa sono caduto nel rabbit hole dell'homelabbing. Da buon nerd quale sono, non ho saputo resistere alla tentazione di costruire il mio piccolo server personale. Sono sempre stato affascinato dalle configurazioni compatte e silenziose — non necessariamente le più potenti, ma le più adatte allo scopo, così dopo aver letto decine e decine di post su [r/homelab](https://www.reddit.com/r/homelab/) e guardato più o meno ogni video della serie [TyniMiniMicro](https://www.servethehome.com/introducing-project-tinyminimicro-home-lab-revolution/) mi sono finalmente deciso.

Ho iniziato con un HP ProDesk 600 G3, un vecchio PC preso per quattro soldi su eBay, il giusto compromesso fra costo, prestazioni e consumi. Insieme al computer ho comprato sei hard disk di seconda mano da 2.5″ e una scheda di espansione SATA. Dopo qualche ora al CAD e un po' di pazienza, ho stampato tutto il necessario per far stare i dischi dentro il case originale.

{{< image src="/homelab-hp-prodesk.webp" caption="Il mio primo homelab — HP ProDesk 600 G3 con 6 hard disk da 2.5″ stampati in 3D" >}}

Da allora il setup si è evoluto. L'HP è andato in pensione, due nuovi mini PC sono entrati in servizio e qualche decina di ore di stampa più tardi, hanno trovato casa in un mini rack dedicato [questo](https://www.youtube.com/watch?v=ZTmmEv9irbM). 
Non solo le macchine si sono moltiplicate, ma anche il numero dei servizi deployati al loro interno, e con essi anche la complessità nel gestirli.

All'inizio era semplice: SSH sulla macchina, creavo il docker-compose.yml a mano, configuravo il reverse proxy, creavo i file di configurazione, lottavo un po' con i permessi e via, il servizio era su. Niente di complicato, finchè qualcosa non andava storto: **un errore di configurazione e tornare indietro non era possibile**. Dovevo sperare di aver salvato una versione funzionante da qualche parte. Spostare un servizio da una macchina all'altra? Un'impresa.

{{< image src="/homelab-mini-rack.jpeg" caption="Il setup attuale — mini rack con due nodi, reverse proxy e rete dedicata" >}}

Così ho deciso di unire le mie due passioni: l'homelab e la continuous delivery (sono uno dei firmatari di [minimumcd.org](https://minimumcd.org/#signatories)). Dopotutto, perchè perdere mezz'ora per fare a mano qualcosa che puoi automatizzare in decine e decine di ore?!?
Scherzi a parte, per me è stata un'interessante esperienza formativa e dopotutto è proprio questo che muove chi decide di costruirsi un piccolo homelab, non è il risparmio, spesso non è neanche davvero la privacy che ti spinge, ma il desiderio di sperimentare e di imparare qualcosa di nuovo. 

---

## Spiegazione — l'infrastruttura in sintesi

Prima di parlare della soluzione, faccio un passo indietro per dare un po' di contesto. Oggi la mia infrastruttura è fatta così:

1. **Due macchine** in totale, entrambe configurate automaticamente tramite script Ansible
2. **Servizi su Docker** orchestrati tramite Portainer, installato su entrambe le macchine
3. **Un reverse proxy** ([SWAG](https://docs.linuxserver.io/general/swag/)) su una delle due macchine, esposto verso l'esterno per tutto ciò che deve essere raggiungibile da fuori rete
4. **[Homepage](https://gethomepage.dev/)** — una dashboard che mostra tutti i servizi, aggiornata automaticamente

Più un'altra decina di servizi che per brevità non menzionerò. 

> **Portainer** è un'interfaccia grafica per Docker. **SWAG** è un container LinuxServer che include nginx, Certbot (certificati SSL), Fail2ban, e una struttura pre-configurata per proxy inverso. **Homepage** è un aggregatore di servizi con discovery automatica via label Docker.

L'infrastruttura funzionava già bene. Il problema non era *cosa* faceva girare i servizi, ma *come* li mettevo in piedi.

---

## Caso pratico — il flusso di deploy

Ecco cosa succede oggi quando voglio lanciare un nuovo servizio:

1. Apro un terminale
2. Eseguo `cruft create https://github.com/Cirius1792/hs-service-template.git`
3. Rispondo a poche domande: nome del servizio, gruppo, icona, dominio
4. Modifico il `docker-compose.yml` generato con l'immagine del servizio
5. Compilo le variabili d'ambiente in `stack.env`
6. Faccio `git push` su main, i repository sono salvati su una istanza di [Gitea](https://about.gitea.com/) direttamente su una delle due macchine

Il resto è automatico. Ecco il flusso:

{{< mermaid >}}
flowchart TD
    A["📦 git push"] --> B["Workflow: <b>Deploy Stack</b>"]
    A --> C["Workflow: <b>Deploy SWAG Config</b>"]
    A --> D["Workflow: <b>Deploy Config Files</b>"]

    B --> B1{"File rilevanti<br/>modificati?"}
    B1 -->|Sì| B2["🧪 docker compose config<br/>—quiet"]
    B1 -->|No| B3["✅ Skippato"]
    B2 --> B4{Valido?}
    B4 -->|No| B5["❌ Fallisce"]
    B4 -->|Sì| B6["☸️ Portainer API<br/>Crea/aggiorna stack"]
    B6 --> B7["🔄 Webhook redeploy"]
    B7 --> B8["✅ Completato"]
    B3 --> B8

    C --> C1{"File rilevanti<br/>modificati?"}
    C1 -->|Sì| C2["📤 SCP → Host remoto<br/>Copia config SWAG"]
    C1 -->|No| C3["✅ Skippato"]
    C2 --> C4["🧪 nginx -t<br/>dentro container SWAG"]
    C4 --> C5{Valido?}
    C5 -->|No| C6["↩️ Rollback backup"]
    C5 -->|Sì| C7["🔄 Ricarica nginx"]
    C7 --> C8["✅ Completato"]
    C6 --> C8
    C3 --> C8

    D --> D1{"File rilevanti<br/>modificati?"}
    D1 -->|Sì| D2["📤 SCP per ogni file<br/>configurations/data/"]
    D1 -->|No| D3["✅ Skippato"]
    D2 --> D4{Errori?}
    D4 -->|Sì| D5["↩️ Rollback tutti i file"]
    D4 -->|No| D6["⚡ Redeploy Portainer<br/>(se necessario)"]
    D6 --> D7["✅ Completato"]
    D5 --> D7
    D3 --> D7

    B8 --> E["🏠 Homepage auto-discovery<br/>via label Docker"]
    C8 --> E
    D7 --> E

    style A fill:#e1f5fe,stroke:#0288d1
    style B fill:#fff3e0,stroke:#e65100
    style C fill:#e8f5e9,stroke:#2e7d32
    style D fill:#fce4ec,stroke:#c62828
    style B5 fill:#ffebee,stroke:#c62828
    style B8 fill:#e8f5e9,stroke:#2e7d32
    style C8 fill:#e8f5e9,stroke:#2e7d32
    style D7 fill:#e8f5e9,stroke:#2e7d32
    style E fill:#f3e5f5,stroke:#7b1fa2
{{< /mermaid >}}

### Come funziona il template

Il cuore della soluzione è un **template Cookiecutter**, gestito con [cruft](https://cruft.github.io/cruft/) per mantenere i repository generati sincronizzati con gli aggiornamenti del template.

Quando viene generato un nuovo progetto, si ottiene un repository già strutturato con:

- **docker-compose.yml** — definizione del servizio con label per Homepage (gruppo, icona, URL)
- **stack.env** — variabili d'ambiente del servizio (TZ, PUID, PGID...)
- **Config SWAG** — configurazione nginx per il reverse proxy (opzionale)
- **Cartella configurations/** — file di configurazione del servizio, versionati, con mappa delle destinazioni remote
- **5 workflow CI/CD** già pronti:
  - **`deploy-stack.yml`** — valida il compose, crea/aggiorna lo stack in Portainer
  - **`deploy-swag-config.yml`** — carica la configurazione nginx sull'host SWAG via SSH, la valida con `nginx -t`, e ricarica il servizio
  - **`deploy-configurations.yml`** — distribuisce i file di configurazione sulle macchine target
  - **`cruft-update.yml`** — ogni settimana controlla se c'è un aggiornamento al template e apre una PR con le modifiche già applicate
- **Renovate** configurato — quando una nuova versione di un servizio è disponibile, apre automaticamente una PR. Io devo solo revisionare e approvare.

> **Git è la source of truth.** Ogni commit è un punto di restore. Ogni push è un deploy. Sbagli qualcosa? Torna al commit precedente e ripeti.

### Benefici nel concreto

- **Niente più SSH** per installare un servizio
- **Rollback immediato**: configurazione sbagliata → `git revert` → push → il deploy precedente è di nuovo attivo
- **Spostare un servizio** da una macchina all'altra? Cambi il secret `PORTAINER_ENVIRONMENT_ID` e pushi
- **Aggiornamenti automatici** con Renovate: quando una nuova versione è disponibile, arriva una PR
- **Il template si aggiorna da solo** con cruft: se migliori il template, tutti i servizi esistenti ricevono una PR

Tutti i dettagli tecnici — secrets, variabili d'ambiente, comandi SSH, formati dei file — sono nel [README del repository](https://github.com/Cirius1792/hs-service-template). 

---

## Conclusione

Quando ho iniziato, configurare un servizio significava 30 minuti di SSH, copia manuale di file, e una preghiera silenziosa che tutto funzionasse al primo colpo. Oggi è un `cruft create`, qualche modifica al compose, e un push.

Alla fine di questo percorso, quello che mi resta non è solo un template funzionante o delle pipeline che fanno il lavoro sporco al posto mio. È la conferma che avere un piccolo server nell'armadio, in cantina o nascosto in qualche scaffale —  dove sbagliare senza conseguenze, dove provare uno stack tecnologico diverso quando si ha tempo, dove ogni problema è un'occasione per capire qualcosa in più — è una delle esperienze formative più efficaci che ci siano.

Quel vecchio HP ProDesk, con baie per i dischi stampate in 3D e ventole fissate con un po' di colla, non era solo un server. Era un banco di prova dove ho potuto sporcarmi le mani con Ansible per la prima volta, dove ho capito come funziona davvero un reverse proxy e la gestione dei certificati, dove ho visto con i miei occhi la differenza tra una pipeline CI/CD ben fatta e ho imparato cosa funziona e cosa no, come fare le cose e, soprattutto, come non farle. Ogni problema incontrato — un container che non parte, un certificato scaduto, un file di configurazione dimenticato, un servizio irraggiungibile — è stato una lezione.

Il template è [pubblico](https://github.com/Cirius1792/hs-service-template), pensato per funzionare su Gitea ma adattabile a GitHub con poche modifiche. Magari non serve a tutti, magari il setup di qualcun altro è completamente diverso. Ma se c'è una cosa che ho imparato è che vale sempre la pena di investire tempo per imparare qualcosa di nuovo, anche solo per il gusto di farlo. Anche se la soluzione finale è un "work in progress". Come del resto lo è qualsiasi homelab che si rispetti.

