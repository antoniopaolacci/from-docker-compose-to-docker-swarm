# from-docker-compose-to-docker-swarm
A docker customer journey... 

Iniziando ad utilizzare docker per l'esecuzione delle nostre applicazioni, abbiamo la necessità di salvare in qualche formato che non sia uno script bash di build o run, i tanti parametri che possono essere passati al comando docker run. Ben presto iniziamo a conoscere Docker Compose: la possibilità di avere un file YAML che descrive i nostri parametri di lancio, di uno o più container, di avere la visione completa dei tutti i servizi della nostra applicazione, ma non solo, anche di poterlo mettere sotto controllo di versione del codice ci fa sentire più sicuri. Appena però abbiamo bisogno di fari girare i nostri container su più nodi o macchine in un cluster ci rendiamo contro che non è lo strumento che stavamo cercando. Abbiamo bisogno di Docker Swarm per fare questo e, da tre anni ormai dalla sua nascita possiamo considerarlo production-ready.

# Docker Compose
Compose è uno strumento per definire ed eseguire applicazioni Docker multi-container. Con Compose, è possibile definire un file YAML per configurare i servizi applicativi. Nel file YAML infatti, è possibile definire più container Docker sotto il cappello “services“, sia a partire da Dockerfile che da immagini già pronte.
Docker Compose se non è nativamente disponibile quando si installa Docker Engine va installato autonomamente e automatizza certe operazioni del demone docker. Ha una propria command line interface (CLI) accessibile tramite il comando docker-compose che semplifica notevolmente la gestione di gruppo dei servizi e container definiti nel file .yml

```dockerfile
docker-compose up -d
```

Dove con *-d* si evita di rimanere attaccati al system out dei container avviati. Il controllo della console ci viene restituito solo dopo queste operazioni.

```dockerfile
Creating network "wordpress-docker_default" with the default driver
Creating volume "wordpress-docker_db_data" with default driver
Building db
Step 1/2 : FROM mysql:5.7
 ---> 98455b9624a9
Step 2/2 : COPY *.cnf /etc/mysql/conf.d/
 ---> 52440b13db7c

Successfully built 52440b13db7c
Successfully tagged poc/mysql-for-wordpress:latest
WARNING: Image for service db was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating wordpress-docker_db_1 ... done
Creating wordpress-docker_wordpress_1 ... done
```

Vediamo cosa è accaduto:

1.	Viene creato un network overlay che lega i due container e li isola da tutto il resto. Il nome è recuperato dalla cartella dove risiede il docker-compose.yml a cui viene concatenato il suffisso *default*
2.	viene creato il vomune *db_data*, sempre con prefisso il nome della cartella dove risiede il file .yml
3.	viene creata l’immagine modificata del database chiamata *poc/mysql-for-wordpress*, tramite il *Dockerfile* presente nella cartella *mysql*

4.	vengono eseguiti i container dei servizi db e wordpress esattamente in quest’ordine per via delle dipendenze indicate nel file .yml 
In questo caso, oltre al consueto prefisso wordpress-docker, notiamo un numero dopo che segue il nome del servizio: si tratta della prima istanza del servizio. La prima cosa che notiamo è che il nome della cartella che contiene il file YAML viene usata come **namespace** per la nomenclatura delle risorse create: questo approccio è interessante perché permette di rilanciare lo stesso stack di servizi da cartelle diverse senza creare conflitti.
Tutte queste operazioni sono sincrone con la CLI: finché i container non saranno avviati, non potremo riprendere il controllo della console. La CLI di compose è interessante perché permette di controllare in gruppo i container:

```dockerfile
docker-compose ps

            Name                          Command               State          Ports
--------------------------------------------------------------------------------------------
wordpress-docker_db_1          docker-entrypoint.sh mysqld      Up      3306/tcp, 33060/tcp
wordpress-docker_wordpress_1   docker-entrypoint.sh apach ...   Up      0.0.0.0:8000->80/tcp
```

visualizza lo stato dei container e le relative porte mappate (se ci sono). 
Quello che abbiamo visto con Docker Compose è perfetto per gestire il ciclo di build e di runtime di più servizi insieme, per il supporto allo sviluppo software è perfetto: possiamo praticamente creare e tirare su l’infrastruttura di cui abbiamo bisogno sulla nostra macchina di sviluppo.
Abbiamo bisogno di altro? Quando compose inizia ad essere un limite?
quando si ha bisogno che i servizi girino su più macchine, Compose non è in grado di controllare questa situazione. La rete che viene creata tra i servizi istanziati è una sottorete dello stesso nodo (detto host) sia esso una virtual machine o server fisico. Puro isolamento basato su *iptables* non esiste un concetto di “cluster di nodi”. Non possiamo quindi far girare WordPress su una macchina e il MySQL in un’altra gestiti da Compose, perché non si “vedrebbero” a livello di rete: i Compose delle due macchine non avrebbero niente in comune. L’unico modo per farli comunicare sarebbe esporre le porte sui rispettivi host (i nodi *fisici*, le macchine) e far dialogare i servizi tramite gli hostname o ip delle rispettive macchine, ma si perde tutta la dinamicità offerta da Docker. se si prova a scalare un servizio (ovviamente sullo stesso host perché non si può fare altrimenti) che ha il binding di una porta sull’host, lo scaling verrà impedito perché la porta è già occupata dalla prima istanza. 
In realtà, ad esempio, tramite lo stack di librerie *Netflix OSS* si può a livello programmatico eseguire più di una istanza dello stesso servizio incrementando il numero di porta, parametro di ingresso del comando docker run, registrando i servizi su un discovery service (Eureka), effettuando load-balancing (Ribbon) e discovery tramite il *service-name* definito. Verrà assegnata un’istanza di un servizio, secondo algoritmo di scheduling, che eseguirà il lavoro.

```dockerfile
docker-compose up -d --scale wordpress=2  
WARNING: The "wordpress" service specifies a port on the host. If multiple containers for this service are created on a single host, the port will clash.  
Starting wordpress-docker_wordpress_1 ... done  
Creating wordpress-docker_wordpress_2 ... error  
ERROR: for wordpress-docker_wordpress_2  Cannot start service wordpress: driver failed programming external connectivity on endpoint wordpress-docker_wordpress_2 (f19636f06cfcf461c0d175a182fa3b7f53bf0f9c430260e76320f7bfa31f9220): Bind for 0.0.0.0:8000 failed: port is already allocated 
```

Rimuoviamo l’intera applicazione costituita dai servizi e precedentemente buildata ed eseguita.

```dockerfile
docker-compose down
```

Quando si ha la necessità di controllare più servizi su più macchine, e magari scalarli su di esse, è necessario qualcosa di più, è necessario **Docker Swarm**.

# Docker Swarm

E’ lo strumento di default per la *gestione di container in un cluster di nodi Docker*.
Uno swarm (“sciame” in italiano) consiste in un insieme di nodi macchine che eseguono il Docker Engine che si comportano da manager, per la gestione del cluster stesso, e/o worker, che eseguono servizi.

1.	**nodi**: si tratta di istanze di Docker che partecipano al cluster. Solitamente, ogni istanza del motore di Docker risiede su una **macchina fisica** o **virtuale** che sia: di conseguenza si può pensare ad un nodo dello swarm come ad una macchina. I nodi possono essere di tipo *manager* o *worker* (o entrambi): più nodi possono avere il ruolo di master, ma solo uno è eletto in un certo momento a coordinare i servizi e a mantenere lo stato del cluster. Il deploy di una applicazione passa sempre da un nodo master che delega un task ad un nodo worker (o un altro master, o a se stesso) in modo che esegua il servizio richiesto. I nodi del cluster informano sempre il nodo master sullo stato dei task assegnati, in modo da mantenere sempre lo stato dei servizi come richiesto.

2.	**task** e **servizi**: abbiamo visto che un nodo master assegna il compito (task appunto) di eseguire un certo servizio su un nodo del cluster (possibilmente worker), ovvero avviare una o più istanze di un container a partire da una immagine. Come vedremo successivamente il *task*, è un concetto abbastanza trasparente, ma fondamentale, nell’uso di Swarm, se non per il fatto che tutta l’interazione tra il nodo master e la CLI è asincrona: da qua si evince che c’è qualcuno che sta facendo qualcosa (task) dopo che abbiamo eseguito un comando.
Swarm non è attivo di default, anche se è già disponibile nell’installazione di Docker (a differenza del Compose). Dobbiamo quindi solo decidere come attivarlo: per lo sviluppo, cioè sulla nostra macchina, possiamo fare un cluster di un nodo semplicemente digitando:

```dockerfile
docker swarm init
```

L’output completo del comando dà anche istruzioni su come aggiungere altri nodi al cluster (sia master che worker).
Tramite i comandi della CLI *docker swarm* e *docker node* è possibile gestire il cluster.

```dockerfile
docker swarm init
Swarm initialized: current node (aj0oz066oahxnksl6ni08nytv) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2u0sfs9ls2opuqcp4iq3kk8whdz1h067xbxrhxcjfzjmlbec55-d8nejlhfbatomd4hta4epv6lh 192.168.65.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

# Docker stacks

Nel gergo Swarm, l’insieme dei servizi che possiamo deployare si chiama *stack*, cioè pila di servizi. La cosa scaltra di Swarm è che usa i file YAML di Docker Compose, quindi il passaggio a Swarm è molto semplice: basta riusare lo stesso identico file *.yml*


```dockerfile
docker stack deploy -c docker-compose.yml wp
Ignoring unsupported options: build, restart

Creating network wp_default
Creating service wp_db
Creating service wp_wordpress
```

1. Il comando stack deploy vuole per forza il riferimento al file YAML e il nome dello stack da creare. Il nome è usato come prefisso per le risorse create: il servizio WordPress, il database e la network che li isola, in analogia a cosa accadeva per il nome della cartella in cui risiedeva il file *.yml* del precedente comando docker-compose.
In un cluster swarm formati da più nodi, questa *network* sarebbe accessibile anche **cross-nodo** se i due container fossero su nodi diversi.

2. *Le direttive build e restart sono state ignorate.* Questo significa che non posso eseguire le build delle immagini con swarm su tutto il cluster. **Swarm è un ambiente di runtime: qualcun altro deve essere responsabile di fornirgli le immagini già pronte.**

La CLI ha restituito subito il controllo, significa che qualcuno sta facendo qualcosa (ricordate i task?) in background (magari su un altra macchina nel caso multi-nodo). L’interazione è quindi asincrona: come si fa a vedere cosa sta succedendo? Il comando ci dà una panoramica dello stato dei task di wp: abbiamo quindi un task per ogni istanza (per questo il numerino in fondo al nome) di container che implementa il servizio richiesto. 

# Docker services

In Docker Compose si parla di servizi: ci appaiono come container (e le sue repliche) gestite dal Docker Compose.
In Swarm questo concetto è reso più forte ed assume un ruolo centrale, diventando di fatto l’unità di deployment.
Quando infatti si deploya una applicazione, in realtà si chiede (al **master**) di creare un servizio, definito nel file YAML (immagine da usare, porte esposte, overlay di rete,...). Dalla richiesta si *schedulano* i task che istanziano i container nel cluster. E' possibile verificare lo stato dello stack con il comando:

```dockerfile

docker stack services wp
ID                  NAME                MODE                REPLICAS            IMAGE                            PORTS
bo6cdv2jwnqz        wp_db               replicated          1/1                 poc/mysql-for-wordpress:latest
klsaic86xu39        wp_wordpress        replicated          1/1                 wordpress:latest                 *:8000->80/tcp
```

Notiamo che abbiamo 1/1 replica rispettivamente del database e di WordPress (corrispondenti ai task), nonché il binding della porta 80 di quest’ultimo sulla 8000 dell’host (nodo fisico), mentre il servizio database non è esposto sull'host.
Adesso siamo in condizioni di poter scalare il container di WordPress a due repliche senza avere problemi:

```dockerfile
docker service scale wp_wordpress=2
wp_wordpress scaled to 2
overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged
```
 Come ha fatto questa volta a funzionare? Funziona perché **il binding della porta è fatto a livello di servizio e non a livello di container**
 
Il servizio quindi rappresenta l’unità non solo *logica*, ma anche fisica, perché è lui responsabile di fare da punto di accesso e da bilanciatore alle funzionalità dei container che *proxa*. Vediamo quindi chi *implementa* questi servizi:
 
 ```dockerfile
docker ps
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS              PORTS                 NAMES
35b70e25a1f3        wordpress:latest                 "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        80/tcp                wp_wordpress.2.xhpc5jebprigkrhlf742f5272
5aaa08bd0ff9        poc/mysql-for-wordpress:latest   "docker-entrypoint.s…"   22 minutes ago      Up 22 minutes       3306/tcp, 33060/tcp   wp_db.1.uvxb7yhbdu1v36lri90lp2zr8
f0f07649a62e        wordpress:latest                 "docker-entrypoint.s…"   22 minutes ago      Up 22 minutes       80/tcp                wp_wordpress.1.tlobb6t4vnxogbpehwwrr6cyd
```

Con il classico docker ps possiamo vedere quali container stanno girando sul nodo in cui siamo connessi: vediamo infatti che la porta non viene esposta a livello di container; possiamo scalarlo tutte le volte che vogliamo: sarà responsabilità del servizio rendere raggiungibili tutti i container indipendentemente dal nodo in cui sono.
Quest’ultima considerazione fa sorgere una domanda: in un contesto reale, è necessario riuscire a far arrivare il traffico da fuori fin dentro il cluster Swarm, per cui è molto probabile che ci sarà un bilanciatore davanti. Come si fa quindi a sapere su quale nodo è stato fatto il binding della porta 8000 in modo da poter raggiungere il punto di ingresso del nostro applicativo. La soluzione che adotta Swarm (ma non solo, anche Kubernetes si comporta così) è chiama *Routing Mesh*: **il binding della porta viene fatto su tutti i nodi del cluster, anche se fisicamente non sta girando un task in quel nodo.**

![image](https://github.com/antoniopaolacci/from-docker-compose-to-docker-swarm/blob/master/routing-mesh-swarm.png)

Portare l'intero stack down della nostra applicazione:

```dockerfile
 docker stack rm stack-demo
 ```
Portare il nostro Docker Engine fuori dallo swarm, utilizzare:

```dockerfile
 docker swarm leave --force
 ```
 
# Docker Swarm View Log

Per visualizzare errori o seguire il log di un service:

```dockerfile
 docker service logs --follow <stack-name>_<service-name>
 ```

# Update Service e Docker Registry

Immaginiamo di dover aggiornare un’immagine del nostro applicativo: abbiamo bisogno di eseguire nuovamente la build dell’immagine.

Su questo Docker Swarm non ci può aiutare: abbiamo infatti detto che è un ambiente di runtime, possibilmente multi-nodo.
Per lavorare bene dovremmo avere un **Docker Registry** su cui deployare le nostre immagini (magari generate da tool di continuous integration come Jenkins) così che ogni task (inviato dallo swarm) potrà scaricarsi l’ultima versione.

Dopo aver aggiornato l’immagine, aggiorniamo ora il servizio, forzandolo:

 ```dockerfile
docker service update --force wp_db
wp_db
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

Controllando i task, ci accorgiamo che è terminato quello precedente e ne è stato avviato uno nuovo subito dopo (questo perché l'*update-order* è di default a **stop-first**, si può verificare con *docker service inspect --pretty wp_db*). A differenza del Docker Compose però, Swarm non sostituisce il vecchio container con quello nuovo all’aggiornamento del servizio, ma ne crea uno nuovo a fianco, seguendo la policy dell’ *update-order*. Per fare un vero e proprio rolling update senza downtime, è possibile specificare l’order-update a **start-first**: in un dato momento ci saranno così contemporaneamente sia il nuovo task che quello vecchio, prima di essere interrotto.

Per eseguire container in modalità swarm, non dovremmo effettuare la build individualmente su ogni nodo dello swarm. Invece tu effettui il build dell'immagine una sola volta, tipicamente su un *continuous integration* server (CI), effettuare il push su un registry server (spesso in hosting locale, oppure puoi utilizzare *docker hub*), e specificare l'immagine dentro il file *.yml* con un parametro di specifica compose *image: * per ogni servizio definito.

Il parametro di specifica compose _"container_name: "_ non è supportato in quanto comprometterebbe la possibilità di ridimensionare o eseguire aggiornamenti (il nome di un contenitore deve essere univoco nel motore docker). Lascia che lo sciame denomini i contenitori e faccia riferimento alla tua app sulla rete docker in base al nome del servizio.

Il parametro di specifica compose _"depends_on: "_ non è supportato poiché i contenitori possono essere avviati su nodi diversi e il ripristino degli aggiornamenti / il ripristino degli errori potrebbe rimuovere alcuni contenitori che forniscono un servizio dopo l'avvio dell'app. Docker può ritentare l'app in errore fino all'avvio dell'altro servizio oppure, preferibilmente, configurare un punto di accesso che attende la disponibilità delle dipendenze con un tipo di ping per un minuto o due.

Il parametro di specifica compose _"deploy: "_ indica allo Swarm come distribuire e aggiornare il servizio, incluso quante repliche, vincoli su dove eseguire, limiti e requisiti di memoria e CPU e quanto velocemente aggiornare il servizio, definire una politica di riavvio, anche se quest'ultima si sconsiglia di farlo perché i docker engine riavviano i container in conflitto con le modalità di deploy dello swarm.

