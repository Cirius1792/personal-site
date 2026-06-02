# Da SSH a git push: come ho automatizzato i deploy sul mio homelab

> Un template Cookiecutter per trasformare "git push" in "container running" — senza toccare un terminale.

---

## Introduzione — il pesce rosso e il mini rack

Un paio di anni fa sono caduto nel rabbit hole dell'homelabbing. Da buon nerd quale sono, non ho saputo resistere alla tentazione di costruire il mio piccolo server personale. Sono sempre stato affascinato dalle configurazioni compatte e silenziose — non necessariamente le più potenti, ma le più adatte allo scopo. Quando ho scoperto r/homelab mi sono finalmente deciso.

Il mio primo "server" è stato un HP ProDesk 600 G3. Un vecchio PC preso per quattro soldi su eBay, ma il giusto compromesso fra costo, prestazioni e consumi. Insieme al computer ho comprato sei hard disk di seconda mano da 2.5" e una scheda di espansione SATA. Dopo qualche ora al CAD e un po' di pazienza, ho stampato tutto il necessario per far stare i dischi dentro il case originale.

{{< image src="/homelab-hp-prodesk.webp" caption="Il mio primo homelab — HP ProDesk 600 G3 con 6 hard disk da 2.5\" stampati in 3D" >}}

Da allora il setup si è evoluto. L'HP è andato in pensione, due nuovi mini PC sono entrati in servizio in un mini rack dedicato. E con l'aumentare dei servizi, è aumentata anche la complessità.

All'inizio era semplice: SSH sulla macchina, creavo il docker-compose.yml a mano, configuravo il reverse proxy. Ogni volta era una piccola cerimonia. Ma dopo il quinto o sesto servizio, qualcosa è cambiato: **un errore di configurazione e tornare indietro non era possibile**. Dovevo sperare di aver salvato una versione funzionante da qualche parte. Spostare un servizio da una macchina all'altra? Un'impresa.

{{< image src="/homelab-mini-rack.jpeg" caption="Il setup attuale — mini rack con due nodi, reverse proxy e rete dedicata" >}}

Così ho deciso di unire le mie due passioni: l'homelab e la continuous delivery. Sono uno dei firmatari di [minimumcd.org](https://minimumcd.org/#signatories), e credo che le pratiche di CI/CD non valgano solo per le startup, ma anche per il server che hai in cantina.

---

## Spiegazione — l'infrastruttura in sintesi

Prima di parlare della soluzione, faccio un passo indietro per dare un po' di contesto. Oggi la mia infrastruttura è fatta così:

1. **Due macchine** in totale, entrambe configurate automaticamente tramite script Ansible
2. **Servizi su Docker** orchestrati tramite Portainer, installato su entrambe le macchine
3. **Un reverse proxy** (SWAG) su una delle due macchine, esposto verso l'esterno per tutto ciò che deve essere raggiungibile da fuori rete
4. **Homepage** — una dashboard che mostra tutti i servizi, aggiornata automaticamente

> **Portainer** è un'interfaccia grafica per Docker. **SWAG** è un container LinuxServer che include nginx, Certbot (certificati SSL), Fail2ban, e una struttura pre-configurata per proxy inverso. **Homepage** è un aggregatore di servizi con discovery automatica via label Docker.

L'infrastruttura funzionava già bene. Il problema non era *cosa* faceva girare i servizi, ma *come* li mettevo sopra.

---

## Caso pratico — il flusso di deploy

Ecco cosa succede oggi quando voglio lanciare un nuovo servizio:

1. Apro un terminale
2. Eseguo `cruft create https://github.com/Cirius1792/hs-service-template.git`
3. Rispondo a poche domande: nome del servizio, gruppo, icona, dominio
4. Modifico il `docker-compose.yml` generato con l'immagine del servizio
5. Compilo le variabili d'ambiente in `stack.env`
6. Faccio `git push` su main

Il resto è automatico. Ecco il flusso:

{{< mermaid >}}
flowchart TD
    A["📦 git push"] --> B{"Modifica docker-compose.yml<br/>o stack.env?"}
    B -->|Sì| C["🧪 docker compose config<br/>—quiet"]
    B -->|No| D{"Modifica swag/<br/>proxy-confs/ ?"}
    C --> E{Valido?}
    E -->|No| F["❌ Pipeline fallisce"]
    E -->|Sì| G["☸️ Portainer API<br/>Crea/aggiorna stack"]
    G --> H["🔄 Webhook redeploy"]
    
    D -->|Sì| I["📤 SCP → Host remoto<br/>Copia config SWAG"]
    D -->|No| J{"Modifica configurations/<br/>data/ ?"}
    I --> K["🧪 nginx -t<br/>dentro container SWAG"]
    K --> L{Valido?}
    L -->|No| M["↩️ Rollback backup"]
    L -->|Sì| N["🔄 Ricarica nginx"]
    
    J -->|Sì| O["📤 SCP per ogni file<br/>configurations/data/"]
    O --> P{Errori?}
    P -->|Sì| Q["↩️ Rollback tutti i file"]
    P -->|No| R["⚡ Redeploy Portainer<br/>(se serve)"]
    
    N --> S["✅ Deploy completato"]
    H --> S
    R --> S
    
    S --> T["🏠 Homepage auto-discovery<br/>via label Docker"]

    style A fill:#e1f5fe,stroke:#0288d1
    style F fill:#ffebee,stroke:#c62828
    style S fill:#e8f5e9,stroke:#2e7d32
    style T fill:#f3e5f5,stroke:#7b1fa2
{{< /mermaid >}}

### Come funziona il template

Il cuore della soluzione è un **template Cookiecutter**, gestito con [cruft](https://cruft.github.io/cruft/) per mantenere i repository generati sincronizzati con gli aggiornamenti del template.

Quando generi un nuovo progetto, ottieni un repository già strutturato con:

- **docker-compose.yml** — definizione del servizio con label per Homepage (gruppo, icona, URL)
- **stack.env** — variabili d'ambiente del servizio (TZ, PUID, PGID...)
- **Config SWAG** — configurazione nginx per il reverse proxy (opzionale)
- **Cartella configurations/** — file di configurazione del servizio, versionati, con mappa delle destinazioni remote
- **5 workflow CI/CD** già pronti:
  - **`deploy-stack.yml`** — valida il compose, crea/aggiorna lo stack in Portainer
  - **`deploy-swag-config.yml`** — carica la configurazione nginx sull'host SWAG via SSH, la valida con `nginx -t`, e ricarica il servizio
  - **`deploy-configurations.yml`** — distribuisce i file di configurazione sulle macchine target
  - **`cruft-update.yml`** — ogni settimana controlla se il template si è evoluto e apre una PR
- **Renovate** configurato — quando una nuova versione di un servizio è disponibile, apre automaticamente una PR. Io devo solo revisionare e approvare.

> **Git è la source of truth.** Ogni commit è un punto di restore. Ogni push è un deploy. Sbagli qualcosa? Torna al commit precedente e ripeti.

### Cosa ottieni in concreto

- **Niente più SSH** per installare un servizio
- **Rollback immediato**: configurazione sbagliata → `git revert` → push → il deploy precedente è di nuovo attivo
- **Spostare un servizio** da una macchina all'altra? Cambi il secret `PORTAINER_URL` e pushi
- **Aggiornamenti automatici** con Renovate: quando una nuova versione è disponibile, arriva una PR
- **Il template si aggiorna da solo** con cruft: se migliori il template, tutti i servizi esistenti ricevono una PR

Tutti i dettagli tecnici — secrets, variabili d'ambiente, comandi SSH, formati dei file — li trovi nel [README del repository](https://github.com/Cirius1792/hs-service-template). Qui voglio solo darti un'idea di cosa è possibile.

---

## Conclusione

Quando ho iniziato, configurare un servizio significava 30 minuti di SSH, copia manuale di file, e una preghiera silenziosa che tutto funzionasse al primo colpo. Oggi è un `cruft create`, qualche modifica al compose, e un push.

I principi che mi porto a casa sono tre:

1. **Git è la source of truth.** Non il server, non un backup dimenticato in una cartella. Ogni commit è versionato, ogni errore è reversibile.
2. **L'automazione paga.** Investire un pomeriggio nel template mi ha fatto risparmiare ore ogni volta che lancio un nuovo servizio.
3. **Le pratiche DevOps non sono solo per le aziende.** CI/CD, infrastructure as code, semantic release — funzionano anche per il server che hai in cantina. O in uno scaffale del soggiorno.

Se anche tu hai un homelab e vuoi smettere di configurare servizi a mano, [dai un'occhiata al repository](https://github.com/Cirius1792/hs-service-template). È pubblico, il README spiega tutto nei dettagli, e se hai domande o suggerimenti — sono tutt'orecchi.

Il repository è ancora un work in progress, ma il cuore della pipeline è solido e lo uso ogni giorno. Se ti sembra utile, provalo. Se hai idee per migliorarlo, parliamone.
