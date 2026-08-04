---
title: "Inferenza locale su un cluster di Asrock BC250"
subtitle: "45GB di VRAM distribuita su 3 nodi con llama.cpp"
date: 2026-08-04
lastmod: 2026-08-04
draft: false
author: "Ciro Lucio Tecce"
authorLink: "https://ciroluciotecce.it"
description: "Come ho costruito un cluster di tre Asrock BC250 per l'inferenza locale di LLM: hardware ex-mining da meno di 1000 euro, 45GB di VRAM distribuita via RPC e i risultati dei benchmark di llama.cpp."
license: ""
images: []
tags: ["homelab", "llm", "llama.cpp", "inferenza locale", "bc250", "vulkan"]
categories: ["Homelab", "LLM"]
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

## Intro
Sono da sempre un appassionato di tecnologia e già qualche anno fa sono entrato nel rabbit hole dell’home labbing. Con tutto questo parlare di AI, sviluppare un interesse per l'inferenza locale credo fosse il prossimo passo naturale. 

Ma ancora prima delle sfide e dell’interesse tecnico che questo mondo porta con se, credo fortemente nel fatto che il futuro dell’AI sia locale. Sia per ragioni pratiche di sostenibilità dell’intero ecosistema AI, che oggi divora risorse in maniera decisamente non proporzionale ai ritorni effettivi che le aziende possono trarne in base al pricing, sia per un tema di democratizzazione dell’accesso a questo tipo di strumenti che non può e non dovrebbe rimanere in mano a grandi provider esterni. 

Negli ultimi mesi credo inoltre sia in crescita la consapevolezza di essere sempre più dipendenti, in un modo o nell’altro, dall’intelligenza artificiale per il supporto e l’accelerazione che ci dà nello svolgimento di moltissimi compiti della vita quotidiana, oggi principalmente in campo lavorativo. Ma non credo che siamo troppo lontani da un’applicazione sempre più pervasiva anche in ambito personale. 


Per questo ho iniziato a cercare una soluzione economica che mi permettesse di muovere i primi passi nel mondo dell’inferenza locale, sperimentando e imparando qualcosa sul deploy di questo tipo di servizi su infrastruttura distribuita. 
La prima cosa che ho fatto è stata scegliere il mio target: almeno 50GB di VRAM per poter ospitare in parallelo almeno due modelli: 
1. un modello più forte, con discrete capacità di ragionamento per il design e la supervisione di task di coding quotidiani 
2. un modello "esecutore", meno capace nel ragionamento ma più veloce nell'esecuzione di piani già predefiniti dal modello principale 

La scelta è ricaduta su Qwen3.6 27B come modello "smart" e Qwen3.6 35B-A3B come esecutore.

Impensabile orientarsi verso una scheda video unica con tutta questa VRAM: ho dovuto cercare soluzioni per dividere il carico su più schede/macchine.

La priorità era comunque limitare il budget vista la situazione attuale del mercato per l’elettronica. Questo mi ha portato ad un primo sacrificio: niente vLLM. 

vLLM si sta orientando sempre di più al mondo enterprise e di fatto oggi è lo strumento open source, per mia conoscenza, maggiormente efficace ed efficiente per il serving di modelli per casi d’uso che richiedano il supporto a più utenti in parallelo. 

Purtroppo vLLM non supporta il backend Vulkan, che è anche quello che permette di accedere a questo mondo con una spesa relativamente contenuta, essendo supportato da praticamente qualsiasi GPU di qualsiasi fascia ed età. Inoltre, se è vero che vLLM è più efficiente quando si servono più utenti in parallelo, almeno al tempo in cui scrivo offre prestazioni leggermente inferiori a llama.cpp per utente singolo. Di conseguenza, dovendo già orientarmi verso hw non di frontiera ed essendo, di fatto, io l’unico utente del servizio, ho dovuto abbandonare l’idea di usare vLLM — anche se sarebbe stato più in linea con il mio obiettivo di imparare qualcosa da portare poi nel mondo professionale come skill acquisita. 

Sul fronte hw invece, mi sono dato un budget di circa 1000 euro e devo dire che non è stato facile trovare qualcosa che potesse soddisfare la mia necessità di VRAM in questa fascia di prezzo. Saltando per brevità tutte le opzioni che ho preso in considerazione durante il percorso, passiamo alle due finaliste: 
1. AMD Mi50 32GB: scheda di qualche anno fa, senza supporto o ottimizzazione per operazioni matriciali di cui gli LLM beneficiano immensamente, ma con una banda di memoria decisamente interessante
2. un cluster di Asrock BC250: principalmente devo l'idea a [questo posto](https://www.reddit.com/r/LocalLLaMA/comments/1tj4unp/amd_bc250_and_the_search_for_cheap_compute/) che parlava dello sblocco della piena potenza di calcolo di questa scheda e del suo relativamente contenuto costo

Dopo aver confrontato entrambe le opzioni, ho scelto la seconda, ben consapevole che le prestazioni ne avrebbero risentito: per raggiungere il mio budget di RAM sarei dovuto passare per quattro nodi separati connessi via ethernet, invece che per due schede potenzialmente installabili sulla stessa macchina come sarebbe stato con la Mi50. Questa opzione era anche la più economica fra le due, cosa che probabilmente ha avuto un impatto maggiore sulla mia decisione. Optare per due Mi50 mi sarebbe costato circa il 50% in più considerando anche l'hardware a corredo che avrei dovuto acquistare.

Infatti, l'inferenza multi GPU su singola macchina avrebbe anche richiesto una scheda madre e un processore con abbastanza PCIe lanes da soddisfare il fabbisogno del setup — e la relativa RAM — e, visti i prezzi di oggi, mi avrebbe lasciato senza reni. 

## Hardware e Montaggio

Prima di tutto, chiariamo di cosa stiamo parlando: la BC250 non è una GPU tradizionale da infilare in uno slot PCIe, ma una scheda madre compatta nata per il mining di criptovalute, costruita attorno all'APU AMD Cyan Skillfish — una versione depotenziata di quella che si trova dentro la PlayStation 5. In pratica, dentro ogni scheda c'è un computer completo: 6 core Zen 2 a 3.5GHz, una GPU RDNA 2 (più o meno) con 24 Compute Unit (sbloccabili fino a 40 con la mod di cui parlo nella sezione software) e 16GB di GDDR6. Quindi di fatto ogni scheda è un nodo a sé stante, con la sua CPU, la sua memoria e la sua porta di rete — non delle semplici GPU da collegare tra loro, e questo spiega molte delle scelte (e dei limiti) che vedremo.

Il punto chiave per l'inferenza è la memoria: condivisa tra CPU e GPU su un bus a 256 bit, con una banda di circa 448 GB/s. Gli LLM sono assetati di banda di memoria molto più che di pura potenza di calcolo, ed è proprio per questo che queste schede ex-mining sono diventate popolari come piattaforma economica per l'inferenza locale. Il rovescio della medaglia: la memoria va partizionata nel BIOS (io uso 512MB dedicati alla GPU e il resto come memoria unificata via GTT), lo storage si riduce a un singolo slot M.2 con due sole lane PCIe 2.0, la rete è Gigabit Ethernet e il supporto software è solo Linux — su Windows la GPU non è supportata, la codifica e decodifica video è disabilitata e l'IOMMU è rotto per costruzione. Per finire, il TDP arriva fino a 220W con un raffreddamento passivo pensato per un rack, non per un desktop: il che anticipa i problemi di temperatura di cui parlo più avanti.

Ed eccoci qui: 
1. 4 BC250 
2. 4 SSD di seconda mano presi su vinted a 10 euro l'uno 
3. un alimentatore da 1000W anch'esso di seconda mano
4. uno switch di rete unmanaged preso su aliexpress a circa 20 euro 
5. un rack custom stampato in 3D derivato modificando un labrax
6. due ventole Arctic da 120mm, in cui ho lasciato un pezzo di falange durante il montaggio di prova, per raffreddare il tutto 


{{< image src="/rack_picture.jpg" caption="Il rack stampato in 3D con le quattro BC250" >}}

N.B.: Sfortunatamente, una delle quattro schede non ha mai dato segni di vita e, mea culpa, le ho provate troppo tardi dopo l'arrivo per poter aprire una contestazione con il venditore, quindi i risultati riportati di seguito faranno riferimento ad una configurazione con soli tre nodi.

Gli SSD non sono né particolarmente capienti né particolarmente performanti. Sulla capienza, prevedo di sostituirli tutti in futuro se questo setup sopravvivrà alla prova del tempo. Oggi, praticamente ad ogni cambio modello, la porzione deployata sul singolo nodo viene inviata sequenzialmente a tutti i nodi del cluster tramite Ethernet a 1Gb; con SSD più capienti potrei usare la cache di llama.cpp in modalità RPC ed evitare di rispedire i modelli sulla rete. Per i tempi di caricamento dei modelli dall'SSD, invece, c'è poco da fare: la BC250 offre uno slot M.2 2280 PCIe 2.0 con sole 2 lane dedicate, quindi non ci si possono aspettare miracoli.
Insomma, il caricamento dei modelli non sarà un fulmine ed è qualcosa con cui dovrò convivere. 

Le quattro schede sono montate dentro un rack stampato in 3D, la cui struttura è quella di un LabRax [LabRax 10" Server Rack - Bolted Version - 5U](https://makerworld.com/en/models/1464819-lab-rax-10-server-rack-bolted-version-5u). Ho modificato e stampato due traverse più lunghe per ospitarle: altrimenti non sarebbero mai entrate in un rack da 10" tradizionale. 
Le schede sono state montate verticalmente una a fianco dell'altra in gruppi da due, ospitate da due bracket custom disegnate da me con le mie terribili skill di modellazione 3D dopo diverse iterazioni (fallimentari) e misure sbagliate.

{{< image src="/cad.png" caption="Il bracket custom per il montaggio verticale delle schede" >}} 
Il montaggio finale non è dei più semplici e deve avvenire in un preciso ordine: le schede, ad esempio, vanno montate prima di chiudere la parte superiore del rack, per avere margine di manovra nell'inserimento e nel serraggio delle viti di fissaggio. 
[Qui](https://www.printables.com/model/1799662-4-bc250-10-rack-mount) potete trovare tutto il necessario e le istruzioni di montaggio.

### Temperature
La parte che mi spaventava di più erano le temperature: queste schede sono ben lungi dall'essere energeticamente efficienti, dopotutto erano destinate all'utilizzo in server veri e propri, con un raffreddamento degno di questo nome. Io, ingenuamente, speravo di poterne raffreddare due con una sola ventola Arctic da 120mm piazzata sul retro a metà fra una scheda e l'altra. 
Povero sciocco. 
Ma nonostante questo — passando alla versione PRO della stessa ventola, che raggiunge 3000rpm contro i 1800 della precedente, e progettando e stampando un fan duct terribile ma, almeno in apparenza, minimamente efficace, disegnato al meglio delle mie possibilità — sono riuscito ad avere temperature abbastanza accettabili per il mio caso d'uso. 
Quindi nel setup attuale una sola ventola al 100% riesce a raffreddare due schede. La scheda di destra (guardando frontalmente il rack) riceve probabilmente la maggior parte del flusso d'aria e mantiene temperature di sei o sette gradi inferiori rispetto alla sinistra. 
Questo porta al seguente comportamento termico: 
1. Idle: scheda rx 47 gradi, scheda sx 55 gradi
2. Load per inferenza distribuita: scheda rx 66 gradi, scheda sx 75 gradi
3. Load per inferenza su singolo nodo: ~85 gradi

{{< image src="/dashboard_1_node.png" caption="Dashboard temperature — singolo nodo" >}}

{{< image src="/dashboard_2_nodes.png" caption="Dashboard temperature — due nodi" >}}

Ho riportato i risultati sia nel caso dell'inferenza distribuita che su singolo nodo perché nel primo caso, essendo il parallelismo implementato a livello di layer del modello in questa configurazione, le schede non rimangono attive in maniera continuativa durante tutto il tempo del task, ma si attivano a turno passando il carico da una scheda all'altra in loop. Di fatto questo dà modo alle schede di raffreddarsi fra un turno e l'altro e complessivamente le temperature sono mediamente più basse. 
Questo non succede per l'inferenza su singolo nodo: il carico è continuativo e le schede arrivano decisamente al limite della capacità di dissipazione del mio piccolo rack. 

In sostanza: risultato non eccezionale, ma le schede non raggiungono la temperatura di fusione del nocciolo. Quindi, per il momento: Missione Compiuta! Ben consapevole che esiste ancora un significativo margine di miglioramento. 

## Software e Configurazione 
La maggior parte delle informazioni che mi sono state utili per mettere a punto la configurazione del cluster vengono da un'ottima documentazione che raccoglie le principali guide e istruzioni per questa scheda: [BC250](https://elektricm.github.io/amd-bc250-docs/)

Le principali configurazioni applicate sono: 
1. Installazione di Ubuntu server (non Fedora, che sarebbe stata la scelta migliore, ma io sono più familiare con Ubuntu) tramite custom iso headless preparata con Cubit
2. Installazione del kernel 7.x (built-in con Ubuntu 26.04+)
3. Sblocco dei 40 CU su tutte e tre le schede 
4. Impostazione VRAM a 512MB per lasciare quanta più RAM disponibile al sistema
5. Configurazione del GTT a 15GB per permettere il massimo uso della memoria unificata da parte di llama.cpp
6. Installazione dei driver MESA come indicato dalla documentazione menzionata sopra 
7. Installazione di Docker e deploy di llama.cpp compilato con i flag RPC e Vulkan attivi 

Tutti i nodi sono usati solo come worker: un'istanza di llama-swap su una macchina dedicata orchestra l'allocazione dei modelli e la distribuzione sui nodi.

L'inferenza distribuita viene realizzata tramite layer parallelism, meno oneroso in termini di overhead di rete quando la connessione avviene tramite una rete standard e senza l'utilizzo di tecniche più sofisticate come, ad esempio schede RDMA dedicate.
In modalità RPC con layer parallelism llama.cpp non replica il modello su tutti i nodi, ma lo taglia a fette: ogni nodo si occupa di un sottoinsieme dei layer della rete. Ad ogni passo di generazione i nodi devono però scambiarsi via rete i dati ai confini tra le fette, e questo traffico si paga a ogni token prodotto. Più nodi aggiungi, più coordinazione e sincronizzazione introduci — è il motivo per cui, come vedremo, la configurazione a due nodi risulta più veloce di quella a tre.

Per semplificare il lavoro, tutta la configurazione sulle macchine è stata applicata tramite script Ansible. 

In particolare, il ruolo per lo sblocco delle 40 CU è stato riscritto integralmente (grazie AI!) perché lo script originale non funziona nativamente su Ubuntu. 

 <details>
 <summary>Ruolo Ansible per lo sblocco dei 40 CU</summary>

```yaml
---
# 40 CU Unlock — Inference Cluster Worker Node
# Upstream-aligned build workflow, with Ansible-owned enable/reboot:
#   ./scripts/bc250-enable-40cu.sh build
#   write /etc/modprobe.d/bc250-40cu.conf
#   ansible.builtin.reboot

- name: 40CU | Skip — role is disabled (cu40_enabled = false)
  ansible.builtin.debug:
    msg: "40 CU Unlock is disabled. Set cu40_enabled=true to enable."
  when: not (cu40_enabled | bool)
  tags: [40cu]

- name: 40CU | Converge unlock state
  when: cu40_enabled | bool
  tags: [40cu]
  block:
    - name: 40CU | Compute kernel source paths
      ansible.builtin.set_fact:
        _cu40_kernel_source_version: "{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_dir: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_compat_link: "/usr/src/linux-{{ ansible_kernel | regex_replace('[-+].*$', '') }}"
        _cu40_kernel_source_amdgpu_check: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c"
        _cu40_kernel_source_tree_check: "/usr/src/linux-source-{{ ansible_kernel | regex_replace('[-+].*$', '') }}/drivers/gpu/drm/amd/amdkfd/Makefile"

    - name: 40CU | Install role dependencies
      ansible.builtin.apt:
        name:
          - build-essential
          - zstd
          - git
          - curl
          - pciutils
          - vulkan-tools
          - mesa-vulkan-drivers
          - "linux-headers-{{ ansible_kernel }}"
          - "linux-source-{{ _cu40_kernel_source_version }}"
        state: present
        update_cache: true
        cache_valid_time: 3600
      become: true

    - name: 40CU | Prepare kernel source tree for upstream build script
      ansible.builtin.shell:
        cmd: |
          set -euo pipefail
          mkdir -p {{ _cu40_kernel_source_dir | quote }}
          for tarball in \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.xz" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.bz2" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.gz"; do
            if [ -f "$tarball" ]; then
              tar xf "$tarball" \
                -C {{ _cu40_kernel_source_dir | quote }} \
                --strip-components=1 \
                --wildcards \
                '*/drivers/gpu/drm/amd/*'
              test -f {{ _cu40_kernel_source_amdgpu_check | quote }}
              test -f {{ _cu40_kernel_source_tree_check | quote }}
              exit 0
            fi
          done
          echo "Cannot find linux-source tarball for {{ _cu40_kernel_source_version }} under /usr/src" >&2
          exit 1
        executable: /bin/bash
        creates: "{{ _cu40_kernel_source_tree_check }}"
      become: true

    - name: 40CU | Check kernel source compatibility link
      ansible.builtin.stat:
        path: "{{ _cu40_kernel_source_compat_link }}"
      register: _cu40_kernel_source_compat_link_stat
      become: true

    - name: 40CU | Create kernel source compatibility link for upstream script
      ansible.builtin.file:
        src: "{{ _cu40_kernel_source_dir }}"
        dest: "{{ _cu40_kernel_source_compat_link }}"
        state: link
        force: true
      become: true
      when: >-
        (not _cu40_kernel_source_compat_link_stat.stat.exists)
        or (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Check existing non-link compatibility source tree
      ansible.builtin.stat:
        path: "{{ _cu40_kernel_source_compat_link }}/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c"
      register: _cu40_kernel_source_compat_tree_stat
      become: true
      when:
        - _cu40_kernel_source_compat_link_stat.stat.exists
        - not (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Assert existing compatibility path is a usable source tree
      ansible.builtin.assert:
        that:
          - _cu40_kernel_source_compat_tree_stat.stat.exists
        fail_msg: >-
          {{ _cu40_kernel_source_compat_link }} exists but is neither a symlink
          managed by this role nor a usable kernel source tree containing
          drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c.
      when:
        - _cu40_kernel_source_compat_link_stat.stat.exists
        - not (_cu40_kernel_source_compat_link_stat.stat.islnk | default(false))

    - name: 40CU | Preflight — detect BC250 PCI device
      ansible.builtin.command:
        cmd: lspci -nn
      register: _cu40_lspci
      changed_when: false
      become: true

    - name: 40CU | Assert BC250 PCI ID 13fe is present
      ansible.builtin.assert:
        that:
          - _cu40_lspci.stdout is search('(?i)13fe')
        fail_msg: >-
          BC250 PCI ID 13fe not found in lspci output. Aborting 40 CU unlock.
        success_msg: "BC250 PCI ID detected (13fe)."

    - name: 40CU | Clone/update upstream repository
      ansible.builtin.git:
        repo: "{{ cu40_repo }}"
        dest: "{{ cu40_clone_dir }}"
        version: "{{ cu40_repo_version }}"
        update: true
        force: true
        depth: 1
      become: true

    - name: 40CU | Verify upstream script exists and is executable
      ansible.builtin.stat:
        path: "{{ cu40_script_path }}"
      register: _cu40_script_stat
      become: true

    - name: 40CU | Assert upstream script is ready
      ansible.builtin.assert:
        that:
          - _cu40_script_stat.stat.exists
          - _cu40_script_stat.stat.executable
        fail_msg: "Upstream script {{ cu40_script_path }} missing or not executable."

    - name: 40CU | Apply upstream script sentinel compatibility fix
      ansible.builtin.replace:
        path: "{{ cu40_script_path }}"
        regexp: "grep -q 'bc250-cc-clear'"
        replace: "grep -q 'bc250-40cu-enable'"
      become: true

    - name: 40CU | Check governor service is active
      ansible.builtin.command:
        cmd: "systemctl is-active {{ cu40_governor_service }}"
      register: _cu40_governor_active
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Read governor config
      ansible.builtin.slurp:
        src: "{{ cu40_governor_config_path }}"
      register: _cu40_governor_config
      become: true

    - name: 40CU | Decode governor config
      ansible.builtin.set_fact:
        _cu40_governor_config_content: "{{ _cu40_governor_config.content | b64decode }}"

    - name: 40CU | Assert governor profile is capped for 40 CU unlock
      ansible.builtin.assert:
        that:
          - _cu40_governor_active.rc == 0
          - _cu40_governor_config_content is search('(?m)^max\s*=\s*1500\s*$')
          - (_cu40_governor_config_content | regex_findall('(?m)^\[\[safe-points\]\]\s*$') | length) == 3
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*350\s*\n\s*voltage\s*=\s*700')
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*1500\s*\n\s*voltage\s*=\s*900')
          - _cu40_governor_config_content is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*2000\s*\n\s*voltage\s*=\s*1000')
        fail_msg: >-
          Governor safety profile is not in the required 40 CU state.
          Expected max=1500 and safe-points 350/700 + 1500/900 + 2000/1000.

    - name: 40CU | Check if current-kernel amdgpu module is patched
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Ensure build state directory exists
      ansible.builtin.file:
        path: /var/lib/bc250-40cu-unlock
        state: directory
        owner: root
        group: root
        mode: "0755"
      become: true

    - name: 40CU | Check role build marker for current kernel
      ansible.builtin.stat:
        path: "/var/lib/bc250-40cu-unlock/{{ ansible_kernel }}.build-ok"
      register: _cu40_build_marker
      become: true

    - name: 40CU | Check persistent upstream config contains bc250_cc_write_mode=3
      ansible.builtin.shell:
        cmd: >-
          test -f {{ cu40_conf_path | quote }}
          && grep -Eq 'bc250_cc_write_mode=3' {{ cu40_conf_path | quote }}
      register: _cu40_config_enabled
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Check Vulkan/RADV reports num_cu = 40
      ansible.builtin.shell:
        cmd: RADV_DEBUG=info vulkaninfo --summary 2>&1
      register: _cu40_vulkan_summary
      changed_when: false
      failed_when: false
      become: true
      when: cu40_verify_vulkan | bool

    - name: 40CU | Set Vulkan state fact
      ansible.builtin.set_fact:
        _cu40_vulkan_num_cu_40: >-
          {{
            (cu40_verify_vulkan | bool)
            | ternary(
                ((_cu40_vulkan_summary.stdout | default('')) is search('num_cu\s*=\s*40')),
                true
              )
          }}

    - name: 40CU | Compute whether upstream build is required
      ansible.builtin.set_fact:
        _cu40_build_required: >-
          {{
            (_cu40_module_patched.rc != 0)
            or ((not (_cu40_build_marker.stat.exists | default(false)))
                and (not (_cu40_vulkan_num_cu_40 | bool)))
          }}

    - name: 40CU | Reset prepared kernel source tree before rebuild
      ansible.builtin.shell:
        cmd: |
          set -euo pipefail
          for tarball in \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.xz" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.bz2" \
            "/usr/src/linux-source-{{ _cu40_kernel_source_version }}.tar.gz"; do
            if [ -f "$tarball" ]; then
              tar xf "$tarball" \
                -C {{ _cu40_kernel_source_dir | quote }} \
                --strip-components=1 \
                --wildcards \
                '*/drivers/gpu/drm/amd/*'
              test -f {{ _cu40_kernel_source_amdgpu_check | quote }}
              test -f {{ _cu40_kernel_source_tree_check | quote }}
              exit 0
            fi
          done
          echo "Cannot find linux-source tarball for {{ _cu40_kernel_source_version }} under /usr/src" >&2
          exit 1
        executable: /bin/bash
      become: true
      when: _cu40_build_required | bool

    - name: 40CU | Build patched module (upstream build)
      ansible.builtin.command:
        cmd: "{{ cu40_script_path }} build"
        chdir: "{{ cu40_clone_dir }}"
      environment:
        MAKEFLAGS: "-j{{ cu40_build_jobs }}"
      register: _cu40_build
      become: true
      when: _cu40_build_required | bool

    - name: 40CU | Mark whether build executed
      ansible.builtin.set_fact:
        _cu40_build_ran: "{{ _cu40_build is defined and _cu40_build is changed }}"

    - name: 40CU | Compute whether enable/reboot is required
      ansible.builtin.set_fact:
        _cu40_enable_required: >-
          {{
            (_cu40_build_ran | bool)
            or (_cu40_config_enabled.rc != 0)
            or (not (_cu40_vulkan_num_cu_40 | bool))
          }}

    - name: 40CU | Verify installed module is patched before enabling
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched_before_enable
      changed_when: false
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Assert installed module can accept 40 CU parameter
      ansible.builtin.assert:
        that:
          - _cu40_module_patched_before_enable.rc == 0
        fail_msg: >-
          Cannot enable 40 CU mode because the installed amdgpu module does not
          expose bc250_cc_write_mode. The upstream build/install step did not
          leave a usable patched module for the current kernel.
      when: _cu40_enable_required | bool

    - name: 40CU | Manage persistent 40 CU modprobe config
      ansible.builtin.copy:
        dest: "{{ cu40_conf_path }}"
        content: |
          # BC-250 40 CU re-enablement
          # Managed by Ansible — 40cu-unlock role
          options amdgpu bc250_cc_write_mode=3
        owner: root
        group: root
        mode: "0644"
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Reboot to load patched amdgpu with 40 CU mode
      ansible.builtin.reboot:
        msg: "Rebooting to load patched amdgpu with bc250_cc_write_mode=3."
        pre_reboot_delay: "{{ cu40_reboot_pre_delay }}"
        reboot_timeout: "{{ cu40_reboot_wait_timeout }}"
        post_reboot_delay: "{{ cu40_reboot_wait_delay }}"
        reboot_command: "{{ (cu40_reboot_command | length > 0) | ternary(cu40_reboot_command, omit) }}"
      become: true
      when: _cu40_enable_required | bool

    - name: 40CU | Re-check module patched state (post-reconnect)
      ansible.builtin.shell:
        cmd: modinfo amdgpu 2>/dev/null | grep -q bc250_cc_write_mode
      register: _cu40_module_patched_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-check persistent config (post-reconnect)
      ansible.builtin.shell:
        cmd: >-
          test -f {{ cu40_conf_path | quote }}
          && grep -Eq 'bc250_cc_write_mode=3' {{ cu40_conf_path | quote }}
      register: _cu40_config_enabled_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-check Vulkan/RADV state (post-reconnect)
      ansible.builtin.shell:
        cmd: RADV_DEBUG=info vulkaninfo --summary 2>&1
      register: _cu40_vulkan_summary_post
      changed_when: false
      failed_when: false
      become: true
      when: cu40_verify_vulkan | bool

    - name: 40CU | Re-check governor service state (post-reconnect)
      ansible.builtin.command:
        cmd: "systemctl is-active {{ cu40_governor_service }}"
      register: _cu40_governor_active_post
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Re-read governor config (post-reconnect)
      ansible.builtin.slurp:
        src: "{{ cu40_governor_config_path }}"
      register: _cu40_governor_config_post
      become: true

    - name: 40CU | Decode governor config (post-reconnect)
      ansible.builtin.set_fact:
        _cu40_governor_config_content_post: "{{ _cu40_governor_config_post.content | b64decode }}"
        _cu40_vulkan_num_cu_40_post: >-
          {{
            (cu40_verify_vulkan | bool)
            | ternary(
                ((_cu40_vulkan_summary_post.stdout | default('')) is search('num_cu\s*=\s*40')),
                true
              )
          }}

    - name: 40CU | Assert final desired state
      ansible.builtin.assert:
        that:
          - _cu40_module_patched_post.rc == 0
          - _cu40_config_enabled_post.rc == 0
          - (not (cu40_strict_verify | bool)) or (_cu40_vulkan_num_cu_40_post | bool)
          - _cu40_governor_active_post.rc == 0
          - _cu40_governor_config_content_post is search('(?m)^max\s*=\s*1500\s*$')
          - (_cu40_governor_config_content_post | regex_findall('(?m)^\[\[safe-points\]\]\s*$') | length) == 3
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*350\s*\n\s*voltage\s*=\s*700')
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*1500\s*\n\s*voltage\s*=\s*900')
          - _cu40_governor_config_content_post is search('(?ms)\[\[safe-points\]\]\s*frequency\s*=\s*2000\s*\n\s*voltage\s*=\s*1000')
        fail_msg: >-
          Final 40 CU verification failed.
          Required: persistent config bc250_cc_write_mode=3, Vulkan num_cu = 40,
          governor service active and capped profile present.

    - name: 40CU | Write role build marker for current kernel
      ansible.builtin.copy:
        dest: "/var/lib/bc250-40cu-unlock/{{ ansible_kernel }}.build-ok"
        content: |
          kernel={{ ansible_kernel }}
          script={{ cu40_script_path }}
          repo={{ cu40_repo }}
          version={{ cu40_repo_version }}
        owner: root
        group: root
        mode: "0644"
      become: true
      when: _cu40_build is defined and _cu40_build is changed

    - name: 40CU | Diagnostic — runtime bc250_cc_write_mode sysfs
      ansible.builtin.shell:
        cmd: cat /sys/module/amdgpu/parameters/bc250_cc_write_mode 2>/dev/null || echo unavailable
      register: _cu40_diag_sysfs_mode
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — last active_cu_number log line
      ansible.builtin.shell:
        cmd: dmesg 2>/dev/null | grep 'active_cu_number' | tail -1 || true
      register: _cu40_diag_active_cu
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — recent bc250-40cu log lines
      ansible.builtin.shell:
        cmd: dmesg 2>/dev/null | grep 'bc250-40cu' | tail -n 8 || true
      register: _cu40_diag_patch_logs
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Diagnostic — upstream status output
      ansible.builtin.command:
        cmd: "{{ cu40_script_path }} status"
        chdir: "{{ cu40_clone_dir }}"
      register: _cu40_diag_status
      changed_when: false
      failed_when: false
      become: true

    - name: 40CU | Show diagnostics summary
      ansible.builtin.debug:
        msg:
          - "sysfs bc250_cc_write_mode: {{ _cu40_diag_sysfs_mode.stdout | default('n/a') | trim }}"
          - "dmesg active_cu_number: {{ _cu40_diag_active_cu.stdout | default('n/a') | trim }}"
          - "dmesg bc250-40cu lines:\n{{ _cu40_diag_patch_logs.stdout | default('n/a') }}"
          - "upstream status:\n{{ _cu40_diag_status.stdout | default('n/a') }}"

    - name: 40CU | Warn when Vulkan verification is disabled (emergency mode)
      ansible.builtin.debug:
        msg: >-
          WARNING: cu40_verify_vulkan=false. Vulkan/RADV num_cu verification is
          skipped in this run. Use only for emergency diagnostics.
      when: not (cu40_verify_vulkan | bool)
```
 </details>

## Prestazioni

La domanda a cui tutto questo doveva rispondere è una sola: valeva la pena? Per capirlo ho fatto girare una serie di benchmark con llama-bench, confrontando la configurazione a due nodi con quella a tre su quattro modelli: Gemma 4 12B, Gemma 4 26B-A4B, Qwen 3.6 35B-A3B e Qwen 3.6 27B, tutti in quantizzazione UD-Q4_K_XL (la variante "XL" tiene i tensori di alto livello in precisione maggiore).

Il setup di test: backend Vulkan con RPC per la distribuzione multi-nodo, tutti i layer offloadati sulle schede (`-ngl 99`), batch size 4096 e micro-batch 2048. Per ogni modello ho misurato due fasi — prompt processing (512 token) e generazione dei token (128 token) — ciascuna a due profondità di contesto: a freddo (nessun token prefillato, `-d 0`) e a contesto pieno (4096 token prefillati, `-d 4096`). Tutto tramite lo script `bench.sh`, riportato in fondo alla sezione.

La configurazione attuale usa il *layer parallelism* gestito automaticamente da llama.cpp in base alla dimensione del modello e alla capacità dei nodi: ogni GPU elabora un sottoinsieme dei layer del modello, comunicando con gli altri nodi ad ogni passo. Questo è un dettaglio importante perché spiega il risultato principale: **la configurazione a due nodi è più veloce di quella a tre in tutte le 16 comparazioni**, ma questo era un risultato atteso. L'inferenza distribuita aggiunge strutturalmente un overhead di rete — ogni coordinazione tra i nodi costa — e il terzo nodo porta più parallelismo ma anche più traffico RPC e più sincronizzazione. Il fatto che il degrado sia più marcato sulla generazione dei token che sul prompt processing è dovuto alla natura autoregressiva dell'inferenza: il prompt viene valutato tutto in una singola run di calcolo (quindi l'overhead di rete si paga una volta sola), mentre la generazione produce un token alla volta, richiedendo coordinazione continua tra i nodi ad ogni passo.

### La profondità del contesto

Il primo risultato riguarda la profondità del contesto, che ha un effetto opposto sulle due fasi. Elaborare un prompt con la cache già piena a 4096 token è più lento — normale, l'attenzione deve lavorare su una finestra già popolata — ma la generazione dei token a profondità 4096 diventa più veloce, probabilmente per via del beneficio della potenza di calcolo aggiuntiva rispetto all'overhead introdotto. I grafici sotto mostrano la variazione percentuale rispetto alla profondità 0.

{{< image src="/three-node-depth-comparison.png" caption="Effetto della profondità del contesto su tre nodi" >}}

{{< image src="/two-node-depth-comparison.png" caption="Effetto della profondità del contesto su due nodi" >}}

| Modello | pp512 (3 nodi) | tg128 (3 nodi) | pp512 (2 nodi) | tg128 (2 nodi) |
|---|---|---|---|---|
| Gemma 4 12B | -14.0% | +0.7% | -14.2% | +22.6% |
| Gemma 4 26B-A4B | -22.3% | -1.9% | -24.8% | +13.0% |
| Qwen 3.6 35B-A3B | -19.8% | +16.2% | -18.8% | +28.9% |
| Qwen 3.6 27B | -10.0% | +10.8% | -10.3% | +12.4% |
| **Media** | **-16.5%** | **+6.5%** | **-17.0%** | **+19.2%** |

In media il prompt processing perde circa il 16-17% con il contesto pieno su entrambe le configurazioni, mentre la generazione ne beneficia: +19.2% su due nodi, ma solo +6.5% su tre. E su tre nodi il comportamento non è nemmeno uniforme: Gemma 4 12B resta praticamente invariata, Gemma 4 26B-A4B diventa addirittura più lenta.

{{< image src="/long_run.png" caption="Scaling delle prestazioni al crescere della profondità di contesto" >}}

Il risultato non è una sorpresa, ma la conferma di quanto ci si poteva aspettare dal layer parallelism su Ethernet a 1 Gb: aggiungere un terzo nodo non ha portato throughput, ha portato overhead. I grafici e la tabella seguenti lo mostrano in dettaglio.

{{< image src="/two-vs-three-node-comparison.png" caption="Confronto due vs tre nodi" >}}

La tabella seguente mostra la variazione percentuale passando da due a tre nodi (valori negativi = tre nodi più lenti).

| Modello | Profondità | pp512 | tg128 |
|---|---|---|---|
| Gemma 4 12B | 0 | -9.4% | -8.1% |
| Gemma 4 12B | 4096 | -9.1% | -24.5% |
| Gemma 4 26B-A4B | 0 | -9.8% | -32.7% |
| Gemma 4 26B-A4B | 4096 | -6.9% | -41.6% |
| Qwen 3.6 35B-A3B | 0 | -8.7% | -18.2% |
| Qwen 3.6 35B-A3B | 4096 | -9.8% | -26.2% |
| Qwen 3.6 27B | 0 | -3.3% | -4.6% |
| Qwen 3.6 27B | 4096 | -3.0% | -6.0% |
| **Media** | | **-7.5%** | **-20.2%** |

In media il prompt processing perde il 7.5% e la generazione il 20.2%, con un picco del -41.6% per la generazione di Gemma 4 26B-A4B a contesto pieno. 

### Cosa significa in pratica

Con questo hardware e questa configurazione, la distribuzione su due nodi di **Qwen 3.6 35B-A3B** ha prestazioni sufficienti per l'utilizzo quotidiano come piccoli task di coding, soprattutto grazie alla velocità di prompt processing — che in questo tipo di task, a mio avviso, ha un ruolo particolarmente importante. Ricordiamo che l'obiettivo di questo cluster era proprio quello di poter deployare Qwen 3.6 35B su due delle quattro schede e Qwen 3.6 27B sulle altre due: direi che il primo obiettivo può dirsi raggiunto.

Purtroppo i risultati meno rosei sono quelli della variante densa da 27B. La velocità di generazione dei token è a mio avviso sufficiente per l'utilizzo quotidiano, ma la velocità di prompt processing non può dirsi soddisfacente. Basti pensare che solo il primo messaggio di un coding agent può arrivare a contenere fra gli 8K e i 30K token, e ovviamente al crescere del contesto la velocità di processing decade in maniera considerevole, portando ad attese non tollerabili per un utilizzo quotidiano.

Per non parlare del fatto che il deploy di questi due modelli con una finestra di contesto soddisfacente (fra i 60 e gli 80K token) non è possibile sulle sole tre schede rimaste per il mio setup.

### Lo script di benchmark

Ecco `bench.sh`, lo script con cui ho estratto i valori: lancia llama-bench dentro il container dell'inference engine, con i worker del cluster raggiunti via RPC.

<details>
<summary>bench.sh</summary>

```bash
#!/bin/bash

set -o pipefail

LOGFILE="llama-bench-$(date +%Y%m%d-%H%M%S).log"

docker run --rm \
  --network host \
  --user "$(id -u):$(id -g)" \
  -v "/opt/data/llm/models:/models:ro" \
  --entrypoint /app/llama-bench \
  my-gitea-host/clt/llama-cpp-rpc:latest \
  -rpc 192.168.31.50:50052,192.168.31.52:50052,192.168.31.53:50052 \
  -m /models/gemma/qat/gemma-4-12B-it-qat-UD-Q4_K_XL.gguf \
  -m /models/gemma/qat/gemma-4-26B-A4B-it-qat-UD-Q4_K_XL.gguf \
  -m /models/qwen/3.6/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf \
  -m /models/qwen/3.6/Qwen3.6-27B-UD-Q4_K_XL.gguf \
  -d 0,4096 \
  -ngl 99 \
  -b 4096 \
  -ub 2048 \
  --progress \
  2>&1 | tee "$LOGFILE"

exit ${PIPESTATUS[0]}
```

</details>

## Conclusioni

L'obiettivo del progetto era semplice ma non scontato da raggiungere: con un budget intorno ai 1000 euro, portarsi a casa almeno 50GB di VRAM distribuita per servire in parallelo due modelli — uno "smart" con discrete capacità di ragionamento per i task di coding ed un esecutore più veloce — e nel farlo imparare qualcosa sul deploy di servizi di inferenza su infrastruttura distribuita. La scelta di un cluster di Asrock BC250 si è rivelata l'opzione più economica fra quelle in grado di soddisfare il requisito di memoria, ma non la migliore in termini di prestazioni.

**Cosa ha funzionato.** Purtroppo, a causa del nodo prematuramente scomparso, ho raggiunto solo 45GB di VRAM invece dei 60 teoricamente possibili. Qwen 3.6 35B-A3B distribuito su due nodi ha prestazioni sufficienti per l'utilizzo quotidiano, soprattutto grazie alla velocità di prompt processing, fondamentale nei task di coding. Hanno funzionato anche lo sblocco delle 40 CU, la gestione delle temperature (accettabile, seppur vicina al limite) e la riproducibilità dell'intero setup tramite Ansible, che rende ogni nodo ricostruibile dallo stato del repository.

**Cosa ha funzionato meno.** La variante densa Qwen 3.6 27B delude sul prompt processing: con i contesti reali di un coding agent, che possono superare i 30K token, l'attesa diventa intollerabile. E sulle tre schede rimaste dopo la scheda morta all'arrivo non è possibile ospitare entrambi i modelli con una finestra di contesto soddisfacente. Si confermano poi i limiti strutturali emersi nei benchmark: la rete fa sì che il terzo nodo aggiunga overhead invece di throughput, e il caricamento dei modelli resta lento per via dello storage.
Inoltre, lo stack software attuale — llama-swap davanti a llama.cpp — non sfrutta la VRAM nel migliore dei modi: l'hosting parallelo di due modelli che teoricamente ci starebbero, anche se un po' stretti, diventa difficile. Spesso mi sono imbattuto in problemi di OOM durante i test più intensivi di parallelismo.

Valeva la pena? Non ne sono sicuro: onestamente non so se terrò questo setup. Per meno di 1000 euro ho un cluster di tre nodi che, anche se non per l'inferenza distribuita, potrebbe sempre essere riutilizzato per altri calcoli distribuiti — parliamo di tre Single Board Computer, non di tre schede video. Non so ancora bene cosa farci in futuro; per il momento è un'aggiunta anche esteticamente piacevole al mio setup. È stata l'occasione per un'esperienza diretta sui compromessi dell'inferenza distribuita che non avrei potuto ottenere altrimenti, e ho comunque imparato qualcosa durante il percorso: per il momento, sono soddisfatto. 


