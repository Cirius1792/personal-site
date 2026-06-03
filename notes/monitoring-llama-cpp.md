# Monitoraggio di llama.cpp con Prometheus e Grafana

## Appunti per il post

L'osservabilità e i dati sono la chiave per poter prendere qualsiasi decisione in maniera informata. Tutto quello che non è supportato da un numero, un grafico, un'evidenza concreta, è solo un'impressione e in quanto tale non può guidare un processo di scelta in ambito tecnico.

Da un po' di tempo uno dei mini pc nel mio server sta venendo maltrattato per far girare dei piccoli LLM a fini di sperimentazione. Nonostante l'assenza di GPU discreta, grazie a 32GB di RAM e la GPU integrata nel processore (un ryzen 5 4650G Pro) fa del suo meglio con le sue 7 CU per far girare piccoli modelli locali con backend Vulkan.

Ovviamente anche l'utente più paziente del mondo non potrebbe mai utilizzarlo in un workflow di sviluppo ad agenti, ma fa discretamente il suo lavoro quando parliamo di task batch o piccole richieste one shot per contesti non troppo grandi. Per intenderci, nella configurazione attuale, le prestazioni con Qwen3.6 35B A3B con quantizzazione a 4 bit si attestano intorno ai 70/90 token al secondo per il prefil e circa 12 token al secondo per la generazione, a seconda del contesto ovviamente. Non un fulmine di guerra, ma con un coding agent molto leggero come pi e grazie alla magia della kv cache, piccoli task sono tranquillamente gestibili.

Mentre invece non ci sono problemi di alcun tipo per l'utilizzo in workflow batch, come alcuni di quelli che ho implementato con n8n che usano le capacità multimodali di questo modello.

Insomma, niente trascendentale, ma un piccolo banco di prova per questi modelli.

Fino a poco tempo fa avevo sempre usato llama-swap per la gestione dei modelli. Per chi non lo conosce, llama-swap fa da astrattore per llama-cpp e permette la configurazione di più modelli gestendone in automatico il load e l'unload in base alle richieste, questo già molto prima che llama.cpp implementasse la sua router mode.

Tuttavia, llama-swap iniziava a darmi delle limitazioni. Primo fra tutti, era difficile gestire il versioning e non sono mai riuscito a capire la versione di llama.cpp all'interno delle immagini docker distribuite da llama-swap. Questo problema avrei potuto risolverlo buildando da me l'immagine, ma c'è anche un'altra limitazione che volevo superare: la gestione di richieste parallele su modelli differenti. A quanto pare llama-swap non riesce ad emulare la capacità di llama.cpp di gestire richieste parallele su modelli differenti per via di un suo limite di implementazione. Ovviamente nella mia configurazione attuale questa non rappresenta affatto una limitazione, difficilmente con le mie capacità di calcolo potresti gestire il carico necessario per più richieste su più modelli. Ma considerando che ho dei nuovi pezzi di hardware sulla strada per la mia porta, volevo iniziare a predispormi per uno scale up delle prestazioni, così sono passato direttamente a llama.cpp.

Eppure, da subito ho sentito la mancanza del monitoring che llama-swap offriva out of the box. Infatti l'interfaccia web di llama-swap si occupa anche di mostrare le prestazioni per ogni singola chiamata ricevuta in termini di token elaborati, velocità di prompt processing e di generazione, tempi medi per percentili etc.

Così ho cercato di replicare quella visibilità con qualcosa di più standard e dopo un po' di ricerche ho messo in piedi il meccanismo di monitoring descritto nel github gist che si basa su Prometheus e Grafana. La dashboard mostra l'andamento della velocità di prefil e di generazione per modello attivo, con la possibilità di filtrare per singoli modelli. Mostra i token processati e il tempo cumulativo di elaborazione insieme al numero di richieste concorrenti attualmente attivi e i modelli attualmente caricati in macchina.

Ci sono state delle difficoltà però. Innanzitutto, l'endpoint delle metriche che llama.cpp mette a disposizione va richiamato filtrato per modello, quindi è stato necessario aggiungere uno strato intermedio per gestire uno scraper dedicato di Prometheus.

Purtroppo al momento la soluzione non è pienamente funzionante e quindi ho dovuto per il momento metterla in pausa. Questo perché un noto bug di llama.cpp relativo all'esposizione delle metriche impedisce l'unload dei modelli attualmente caricati. In parole povere, ogni chiamata all'endpoint delle metriche resetta il timer di keep alive del modello, di fatto illudendo llama.cpp che il modello sia ancora necessario e che non possa andare in idle. Da un lato questo causa uno spreco di risorse, dall'altro impedisce di godere delle funzionalità di routing di llama.cpp, perché se il modello attualmente caricato viene percepito come in uso, questo non viene scaricato per fare posto per un altro.
