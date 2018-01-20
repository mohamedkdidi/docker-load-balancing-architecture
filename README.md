# Exécution de WordPress hautement disponible avec MySQL sur Kubernetes

[![Build Status](https://travis-ci.org/IBM/Scalable-WordPress-deployment-on-Kubernetes.svg?branch=master)](https://travis-ci.org/IBM/Scalable-WordPress-deployment-on-Kubernetes)


# Scalable WordPress deployment en Kubernetes Cluster

WordPress est une plate-forme populaire pour l'édition et la publication de contenu pour le web. Dans ce tutoriel, je vais vous montrer comment construire un déploiement WordPress hautement disponible (HA) en utilisant Kubernetes.

WordPress se compose de deux composants principaux: le serveur PHP WordPress et une base de données pour stocker les informations utilisateur, les publications et les données du site. Nous devons faire en sorte que ces deux HA soient tolérants aux pannes pour toute l'application.

L'exécution de services HA peut être difficile lorsque le matériel et les adresses changent; rester debout est difficile. Avec Kubernetes et ses puissants composants réseau, nous pouvons déployer un site HA WordPress et une base de données MySQL sans taper une seule adresse IP (presque).

Dans ce tutoriel, je vais vous montrer comment créer des classes de stockage, des services, des cartes de configuration et des ensembles dans Kubernetes; exécuter HA MySQL; et raccordez un cluster HA WordPress au service de base de données. Si vous n'avez pas encore de cluster Kubernetes, vous pouvez facilement en créer un sur Amazon, Google ou Azure, ou en utilisant  Rancher Kubernetes Engine (RKE)  sur n'importe quel serveur.

## Aperçu de l'architecture

Je vais maintenant vous présenter un aperçu des technologies que nous allons utiliser et de leurs fonctions:

Stockage pour les fichiers d'application WordPress: NFS avec un support de disque persistant GCE
Cluster de base de données: MySQL avec xtrabackup pour la parité
Niveau de l'application: une image WordPress DockerHub montée sur le stockage NFS
Équilibrage de charge et mise en réseau: équilibreurs de charge basés sur Kubernetes et mise en réseau de service
L'architecture est organisée comme indiqué ci-dessous:

- [WordPress (Latest)](https://hub.docker.com/_/wordpress/)
- [MySQL (5.6)](https://hub.docker.com/_/mysql/)
- [Kubernetes Clusters](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov)

Diagramme

![kube-wordpress](https://i.imgur.com/Urk4o79.png)

## Création de classes de stockage, de services et de cartes de configuration dans Kubernetes


Dans Kubernetes, les ensembles avec état offrent un moyen de définir l'ordre d'initialisation du module. Nous utiliserons un ensemble avec état pour MySQL, car il garantit que nos nœuds de données ont suffisamment de temps pour répliquer les enregistrements des pods précédents lors de la rotation. La façon dont nous configurons cet ensemble d'états permettra au maître MySQL de tourner avant n'importe lequel des esclaves, de sorte que le clonage peut se produire directement de maître à esclave quand nous passons à l'échelle supérieure.

Pour commencer, nous devons créer une classe de stockage de volume persistante et une carte de configuration pour appliquer les configurations maître et esclave selon les besoins.

Nous utilisons des volumes persistants pour que les données de nos bases de données ne soient pas liées à des modules spécifiques du cluster. Cette méthode protège la base de données contre la perte de données en cas de perte du module maître MySQL. Lorsqu'un module maître est perdu, il peut se reconnecter aux xtrabackupesclaves sur les noeuds esclaves et répliquer les données de l'esclave vers le maître. La réplication MySQL gère la réplication maître à esclave mais xtrabackupgère la réplication descendante esclave à maître.

Pour allouer dynamiquement des volumes persistants, nous créons la classe de stockage suivante à l'aide de disques persistants GCE. Cependant, Kubernetes offre une variété de fournisseurs de stockage de volume persistant:


```
# storage-class.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  zone: us-central1-a
```

Créez la classe et déployer avec cette commande:  $ kubectl create -f storage-class.yaml.

Ensuite, nous allons créer la configuration, qui spécifie quelques variables à définir dans les fichiers de configuration MySQL. Ces différentes configurations sont sélectionnées par les pods elles-mêmes, mais elles nous offrent un moyen pratique de gérer les variables de configuration potentielles.

Créez un fichier YAML nommé mysql-configmap.yamlpour gérer cette configuration comme suit:

```
# mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Apply this config only on the master.
    [mysqld]
    log-bin
    skip-host-cache
    skip-name-resolve
  slave.cnf: |
    # Apply this config only on slaves.
    [mysqld]
    skip-host-cache
    skip-name-resolve
```

Créer et déployer le ConfigMap avec cette commande:  $ kubectl create -f mysql-configmap.yaml.

Ensuite, nous voulons configurer le service de sorte que les pods MySQL puissent se parler entre eux et que nos pods WordPress puissent parler à MySQL en utilisant  mysql-services.yaml. Cela permet également un équilibreur de charge de service pour le service MySQL.

```
# mysql-services.yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
  ```
  
Avec cette déclaration de service, nous jetons les bases d'un cluster d'écriture multiple et de plusieurs lectures d'instances MySQL. Cette configuration est nécessaire car chaque instance de WordPress peut potentiellement écrire dans la base de données, de sorte que chaque nœud doit être prêt à lire et à écrire.

Pour créer les services ci-dessus, exécutez la commande suivante:

```$ kubectl create -f mysql-services.yaml|```

Nous avons créé la classe de stockage de revendications de volume qui va remettre des disques persistants à tous les pods qui les demandent, nous avons configuré la configuration qui définit quelques variables dans les fichiers de configuration MySQL, et nous avons configuré un réseau ... service de niveau qui va charger les demandes d'équilibrage aux serveurs MySQL. Tout ceci est juste un cadre pour les ensembles avec état, où les serveurs MySQL fonctionnent réellement, que nous explorerons ensuite.

## Configuration de MySQL avec des ensembles avec état

Dans cette section, nous allons écrire la configuration YAML pour une instance MySQL en utilisant un ensemble avec état.

Définissons notre ensemble dynamique:

1. Créez trois pods et enregistrez-les dans le service MySQL.
2. Définissez le modèle suivant pour chaque pod:
3. Créez un conteneur d'initialisation pour le serveur MySQL maître nommé  init-mysql.
  . Utilisez l'  mysql:5.7 image pour ce conteneur.
  . Exécutez un script bash à configurer xtrabackup.
  . Montez deux nouveaux volumes pour la configuration et la configuration.

4. Créez un conteneur d'initialisation pour le serveur MySQL maître nommé  clone-mysql.
  . Utilisez l'image de Google Cloud Registry  xtrabackup:1.0 pour ce conteneur.
  . Exécutez un script bash pour cloner l'existant xtrabackupsdu pair précédent.
  . Montez deux nouveaux volumes pour les données et la configuration.
  . Ce conteneur héberge efficacement les données clonées afin que les nouveaux conteneurs esclaves puissent les récupérer.
5. Créez les conteneurs principaux pour les serveurs MySQL esclaves.
  . Créez un conteneur esclave MySQL et configurez-le pour vous connecter au maître MySQL.
  . Créez un xtrabackupconteneur esclave et configurez-le pour vous connecter au maître xtrabackup.
6. Créez un modèle de revendication de volume pour décrire chaque volume à créer en tant que disque persistant de 10 Go.

La configuration suivante définit le comportement des maîtres et des esclaves de notre cluster MySQL, offrant une configuration bash qui exécute le client esclave et assure le bon fonctionnement d'un maître avant le clonage. Les esclaves et les maîtres ont chacun leur propre volume de 10 Go qu'ils demandent à partir de la classe de stockage de volume persistante que nous avons définie précédemment.

```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on master (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing slave.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from master. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

Enregistrer ce fichier sous  mysql-statefulset.yaml. Tapez  kubectl create -f mysql-statefulset.yaml et laissez Kubernetes déployer votre base de données.

Maintenant, quand vous appelez  $ kubectl get pods, vous devriez voir trois pods en train de tourner ou prêts à recevoir deux conteneurs.

Le pod principal est noté  mysql-0 et les esclaves suivent  mysql-1 et  mysql-2.

Donnez quelques minutes aux pods pour vous assurer que le  xtrabackup service est correctement synchronisé entre les pods, puis passez au déploiement WordPress.

Vous pouvez vérifier les journaux des conteneurs individuels pour confirmer qu'aucun message d'erreur n'est lancé. Pour ce faire, exécutez $ kubectl logs -f -c <container_name>

Le xtrabackupconteneur principal doit afficher les deux connexions des esclaves et aucune erreur ne doit être visible dans les journaux.

# Déployer WordPress hautement disponible

La dernière étape de cette procédure consiste à déployer nos modules WordPress sur le cluster. Pour ce faire, nous voulons définir un service pour WordPress et un déploiement.

Pour que WordPress soit HA, nous voulons que chaque conteneur qui exécute le serveur soit entièrement remplaçable, ce qui signifie que nous pouvons en terminer un et en créer un autre sans modifier la disponibilité des données ou des services. Nous voulons également tolérer au moins un conteneur ayant échoué, ayant un conteneur redondant là pour prendre le relais.

WordPress stocke les données importantes du site dans le répertoire de l'application  /var/www/html. Pour deux instances de WordPress pour servir le même site, ce dossier doit contenir des données identiques.

Lors de l'exécution de WordPress dans HA, nous devons partager les  /var/www/html dossiers entre les instances, nous allons donc définir un service NFS qui sera le point de montage pour ces volumes.

La configuration suivante configure les services NFS. J'ai fourni la version anglaise ordinaire ci-dessous:

> Définissez un contrôleur de réplication pour le serveur NFS qui garantira l'exécution d'au moins une instance du serveur NFS à tout moment.
Définissez une revendication de volume persistante pour créer notre disque NFS partagé en tant que disque persistant GCE de 200 Go.
> Définissez un contrôleur de réplication pour le serveur NFS qui garantira l'exécution d'au moins une instance du serveur NFS à tout moment.
> Ouvrez les ports 2049, 20048 et 111 dans le conteneur pour rendre le partage NFS accessible.
> Utilisez l' volume-nfs:0.8 image du registre Google Cloud  pour le serveur NFS.
> Définissez un service pour le serveur NFS pour gérer le routage d'adresse IP.
> Autoriser les ports nécessaires à travers ce pare-feu de service.


Déployer le serveur NFS en utilisant  $ kubectl create -f nfs.yaml.

Maintenant, nous devons courir  $ kubectl describe services nfs-server pour gagner l'adresse IP à utiliser ci-dessous.

Remarque: À l'avenir, nous serons en mesure de relier ces deux éléments en utilisant les noms de service, mais pour l'instant, vous devez coder en dur l'adresse IP.

```
# wordpress.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 20G
  accessModes:
    - ReadWriteMany
  nfs:
    # FIXME: use the right IP
    server: <IP of the NFS Service>
    path: "/"

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 20G

---

apiVersion: apps/v1beta1 # for versions before 1.8.0 use apps/v1beta1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.9-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          value: ""
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
            claimName: nfs
            
```
Nous avons maintenant créé une revendication de volume persistante qui correspond au service NFS que nous avons créé précédemment. Il attache ensuite le volume au pod WordPress à la  /var/www/html racine, où WordPress est installé. Cela préserve toutes les installations et tous les environnements dans les modules WordPress du cluster. Avec cette configuration, nous pouvons lancer et démonter n'importe quel nœud WordPress et les données resteront. Étant donné que le service NFS utilise constamment le volume physique, il conserve le volume et ne le recycle pas ou ne le répartit pas correctement.

Déployez les instances WordPress en utilisant  $ kubectl create -f wordpress.yaml.

Le déploiement par défaut n'exécute qu'une seule instance de WordPress, alors n'hésitez pas à augmenter le nombre d'instances WordPress en utilisant  $ kubectl scale --replicas=<number of replicas> deployment/wordpress.

Pour obtenir l'adresse de l'équilibreur de charge du service WordPress, tapez  $ kubectl get services wordpress et saisissez le  EXTERNAL-IP champ du résultat pour naviguer vers WordPress.

## Tests de résilience

OK, maintenant que nous avons déployé nos services, commençons à les démolir pour voir à quel point notre architecture HA gère un certain chaos. Dans cette approche, le seul point de défaillance restant est le service NFS (pour les raisons expliquées dans la Conclusion). Vous devriez pouvoir démontrer en testant l'échec de tout autre service pour voir comment l'application répond. J'ai commencé avec trois répliques du service WordPress et un maître et deux esclaves sur le service MySQL.

Tout d'abord, tuons tous les nœuds WordPress sauf un et voyons comment l'application réagit:

$ kubectl scale --replicas=1 deployment/wordpress

Maintenant, nous devrions voir une baisse du nombre de pods pour le déploiement de WordPress.

$ kubectl get pods

Nous devrions voir que les pods WordPress fonctionnent seulement  1/1 maintenant.

Lorsque vous atteignez l'adresse IP du service WordPress, nous verrons le même site et la même base de données qu'auparavant.

Pour agrandir, nous pouvons utiliser  $ kubectl scale --replicas=3 deployment/wordpress.

Nous verrons à nouveau que les données sont conservées dans les trois instances.

Pour tester le StatefulSet de MySQL, nous pouvons réduire le nombre de réplicas en utilisant ce qui suit:

$ kubectl scale statefulsets mysql --replicas=1

Nous verrons une perte des deux esclaves dans cette instance et, en cas de perte du maître à ce moment, les données qu'il contient seront conservées sur le disque persistant GCE. Cependant, nous devrons récupérer manuellement les données du disque.

Si les trois nœuds MySQL ne fonctionnent plus, vous ne pourrez pas répliquer lorsque de nouveaux nœuds apparaîtront. Cependant, si un nœud maître tombe en panne, un nouveau maître sera lancé et via  xtrabackup, il se repeuplera avec les données d'un esclave. Par conséquent, je ne recommande jamais d'exécuter avec un facteur de réplication de moins de trois lors de l'exécution de bases de données de production.

Pour conclure, parlons de quelques meilleures solutions pour vos données d'état, car Kubernetes n'est pas vraiment conçu pour l'état.



# License
[MIT](LICENSE)
