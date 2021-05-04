# wpk8s 

## Architettura
* La ridondanza e la scalabilità del db (con hostname db1..dbN) viene garantita tramite cluster galera (builtin nella versione di mariadb installata):
  * replica sincrona parallela a livello di riga;
  * multi-master active/active;
  * controllo, esclusione e unione dei nodi/membri automatica;
  * look and feel nativo di MySQL (i client possono contattare uno qualsiasi dei nodi che compongono il cluster senza strati ulteriori);
  * sono richieste almeno 3 istanze per la gestione di quorum e split brain e per la sopravvivenza al fail di 1 nodo;
* Gli stessi nodi db hanno a bordo i brick per un volume gluster utilizzando dai server di frontend per i dati wp-content; 
* Il db viene esposto mediante una coppia di bilanciatori active/passive (keepalived/vrrp) chiamati nello schema dbb1/2 e implementati con linux ip virtual server in modalità direct routing:
  * Il bilanciamento viene fatto in kernel space a livello trasporto e sfrutta l'algoritmo wlc (Weighted Least-Connections: nuove connessioni assegnate al server con meno connessioni attive in proporzione al peso assegnatogli);
  * La modalità direct routing è quella più prestante (il vincolo e' che bilanciatori e backend si parlino in L2, il backend viene contattato tramite bilanciatore ma risponde direttamente al chiamante);
  * Fra i 2 nodi è attiva la sincronizzazione delle tabelle di persistenza: in caso di failover viene mantenuta l'associazione fra client e server di backend.
* I frontend vengono configurati in batteria su una vm con a bordo k8s: 
  * k8s è fornito da una istanza minikube avviata con driver none e addons ingress;
    * Il driver none permette di esporre direttamente servizi ed ingress su tutti gli ip della vm;
  * Il punto di ingresso del frontend è un ingress che proxa le richieste a dei service nodePort esposti dai container docker che eseguono wordpress su apache;
  * I pod wordpress sono lanciati (in un numero configurabile) da un deployment;
    * Al deployment è associato un horizontal pod autoscaler (hpa) che replica aumenta le istanze su base cpu utilizzata fino ad un numero massimo configurabile in fase di deploy dell'infrastruttura;
  * Ogni pod monta un volume tramite persistentVolumeClaim sotto /var/www/html/wp-content;
    * Il claim è forzato ad erogare spazio utilizzando un persistentVolume gluster-pv;
    * Il volume gluster-pv è di tipo glusterfs e sfrutta una coppia service/endpoints opportunamente configurata per puntare ai server db che espongono il volume wpsharedfs;
    * Gluster così utilizzato assicura:
      * la possibilità di montare il volume in modalità readWriteMany (https://kubernetes.io/docs/concepts/storage/volumes/#glusterfs);
      * ridondanza dei dati (replicati su tutti i nodi db);
      * scalabilità (pod di piu' nodi k8s possono montare gli stessi volumi, e' possibile aumentare i nodi che espongono il volume gluster e bilanciare il carico aggiungedoli all'endpoints);
  * I pod contattano per l'accesso al db il bilanciatore lvs esterno.

![Architecture](https://github.com/tappof/wpk8s/blob/master/images/wpk8s.png)

# Bootstrap
## Tool e parametrizzazione
* L'intera infrastruttura viene costruita con vagrant e provisioning ansible; 
* E' possibile modificare alcune impostazioni in vmm/provisioning/vm-config.yml;
* La documentazione delle impostazioni e' direttamente nel file di configurazione.

### Prerequisiti e setup dell'ambiente
ambiente: debian buster + vagrant con provider libvirt 
* sudo apt-get install git vagrant sudo qemu libvirt-daemon-system libvirt-clients ebtables dnsmasq-base libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev
* sudo adduser *user* libvirt
* sudo adduser *user* libvirt-qemu
* sudo echo "*user* ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers 
* sudo echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list
* sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
* sudo apt-get update
* sudo apt-get install ansible python-jmespath
* sudo gem install ipaddress
* ansible-galaxy collection install gluster.gluster
* ansible-galaxy collection install community.mysql

## Startup
* Verificare le conf in vmm/provisioning/vm-config.yml (attenzione ai parametri legati al networking)
* Per deployare l'infrastruttura eseguire:
<pre>
cd vmm
vagrant up
</pre>

## Accesso
* Con un browser puntando a http://192.168.122.100

# Note:
* Il playbook ansible per il provisioning non e' completamente idempotente.

