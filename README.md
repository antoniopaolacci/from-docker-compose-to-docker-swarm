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
#Docker Swarm
E’ lo strumento di default per la *gestione di container in un cluster di nodi Docker*.
Uno swarm (“sciame” in italiano) consiste in un insieme di nodi macchine che eseguono il Docker Engine che si comportano da manager, per la gestione del cluster stesso, e/o worker, che eseguono servizi.

1.	**nodi**: si tratta di istanze di Docker che partecipano al cluster. Solitamente, ogni istanza del motore di Docker risiede su una **macchina fisica** o **virtuale** che sia: di conseguenza si può pensare ad un nodo dello swarm come ad una macchina. I nodi possono essere di tipo *manager* o *worker* (o entrambi): più nodi possono avere il ruolo di master, ma solo uno è eletto in un certo momento a coordinare i servizi e a mantenere lo stato del cluster. Il deploy di una applicazione passa sempre da un nodo master che delega un task ad un nodo worker (o un altro master, o a se stesso) in modo che esegua il servizio richiesto. I nodi del cluster informano sempre il nodo master sullo stato dei task assegnati, in modo da mantenere sempre lo stato dei servizi come richiesto.

2.	**task** e **servizi**: abbiamo visto che un nodo master assegna il compito (task appunto) di eseguire un certo servizio su un nodo del cluster (possibilmente worker), ovvero avviare una o più istanze di un container a partire da una immagine. Come vedremo successivamente il *task*, è un concetto abbastanza trasparente, ma fondamentale, nell’uso di Swarm, se non per il fatto che tutta l’interazione tra il nodo master e la CLI è asincrona: da qua si evince che c’è qualcuno che sta facendo qualcosa (task) dopo che abbiamo eseguito un comando.
Swarm non è attivo di default, anche se è già disponibile nell’installazione di Docker (a differenza del Compose). Dobbiamo quindi solo decidere come attivarlo: per lo sviluppo, cioè sulla nostra macchina, possiamo fare un cluster di un nodo semplicemente digitando:

```dockerfile
docker swarm init
Swarm initialized: current node (keiak51s53xowk57oq24bd5yi) is now a manager.
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


