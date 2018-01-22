# Exécution de WordPress hautement disponible avec MySQL sur Kubernetes

[![Build Status](https://travis-ci.org/IBM/Scalable-WordPress-deployment-on-Kubernetes.svg?branch=master)](https://travis-ci.org/IBM/Scalable-WordPress-deployment-on-Kubernetes)

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

![kube-wordpress](https://comtechies.com/wp-content/uploads/2016/04/kubernetes-cluster.png)

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

# Déploiements de microservices sur Kubernetes avec Rancher

Vous pouvez consulter la documentation de Kubernetes pour savoir comment lancer un cluster Kubernetes dans différents environnements. Dans ce post, je vais me concentrer sur le lancement de la distribution de Kubernetes  par Rancher en tant qu'environnement au sein de la plateforme de gestion de conteneurs Rancher . Nous allons commencer par configurer un serveur Rancher comme décrit ici et sélectionner Environnement / Défaut> Gérer les environnements> Ajouter un environnement . Sélectionnez Kubernetes dans les options Container Orchestration et créez votre environnement. Maintenant, sélectionnez Infrastructure> Hôtes>  Ajouter un hôte et lancer quelques nœuds pour que Kubernetes fonctionne. Remarque: nous recommandons d'ajouter au moins 3 hôtes, qui exécuteront le conteneur de l'agent Rancher. Une fois que les hôtes arrivent, vous devriez voir l'écran suivant, et dans quelques minutes votre cluster devrait être prêt et prêt.


Il y a beaucoup d'avantages à faire fonctionner Kubernetes au sein de Rancher. Généralement, cela facilite considérablement le déploiement et la gestion pour les utilisateurs et l'équipe informatique. Rancher implémente automatiquement une implémentation HA de etcd pour le backend Kubernetes et déploie tous les services nécessaires sur tous les hôtes que vous ajoutez dans cet environnement. Il configure les contrôles d'accès et peut facilement se connecter à l'infrastructure LDAP et AD existante. Rancher implémente également automatiquement des services de mise en réseau de conteneurs et d'équilibrage de charge pour Kubernetes. En utilisant Rancher, vous devriez avoir une implémentation HA de Kubernetes en quelques minutes.


![rancher1](http://cdn.rancher.com/wp-content/uploads/2016/08/13055117/Screen-Shot-2016-07-16-at-8.31.03-AM.png)


# NAMESPACES
Maintenant que notre cluster est en cours d'exécution, passons à quelques ressources de base de Kubernetes. Vous pouvez accéder au cluster Kubernetes directement via l'interface CLI kubectl ou via l'interface utilisateur Rancher. La couche de gestion des accès de Rancher contrôle qui peut accéder au cluster. Vous devez donc générer une clé API à partir de l'interface utilisateur Rancher avant d'accéder à la CLI.

La première ressource de Kubernetes que nous allons examiner est celle des espaces de noms. Dans un espace de noms donné, toutes les ressources doivent avoir des noms uniques. En outre, les étiquettes utilisées pour lier des ressources sont limitées à un seul espace de noms. C'est pourquoi les espaces de noms peuvent être très utiles pour créer des environnements isolés sur le même cluster Kubernetes. Par exemple, vous pouvez créer un environnement alpha, bêta et de production pour votre application afin de pouvoir tester les dernières modifications sans affecter les utilisateurs réels. Pour créer un espace de noms, copiez le texte suivant dans un fichier appelé namespace.yaml et exécutez la kubectl create -f namespace.yaml commande pour créer un espace de noms appelé minprojet.

```
kind: Namespace
apiVersion: v1
metadata:
  name: minprojet
  labels:
    name: minprojet
```

Vous pouvez également créer, afficher et sélectionner des espaces de noms à partir de l'interface utilisateur de Rancher en utilisant le menu Namespace de la barre de menus supérieure. 

![rancher2](http://cdn.rancher.com/wp-content/uploads/2016/08/13023604/Screen-Shot-2016-08-13-at-5.35.48-AM-265x300.png)

Vous pouvez utiliser la commande suivante pour définir l'espace de noms dans les interactions CLI à l'aide de kubectl:

```
$ kubectl config set-context Kubernetes --namespace=minprojet.
```

Pour vérifier que le contexte a été défini pour le moment, utilisez la commande config view et vérifiez que la sortie correspond à l'espace de noms attendu.

```
$ kubectl config view | grep namespace command
namespace: minprojet

```
PODS
Maintenant que nous avons défini nos espaces de noms, commençons à créer des ressources. La première ressource que nous allons examiner est un Pod. Un groupe d'un ou plusieurs conteneurs est appelé par Kubernetes comme un pod. Les conteneurs d'un pod sont déployés, démarrés, arrêtés et répliqués en tant que groupe. Il ne peut y avoir qu'un seul pod d'un type donné sur chaque hôte, et tous les conteneurs du pod sont exécutés sur le même hôte. Les pods partagent un espace de noms réseau et peuvent se contacter via le domaine localhost. Les pods sont l'unité de base de la mise à l'échelle et ne peuvent pas s'étendre sur plusieurs hôtes. Il est donc idéal de les rendre aussi proches que possible de la charge de travail unique. Cela éliminera les effets secondaires de la mise à l'échelle d'un pod vers le haut ou vers le bas, tout en veillant à ne pas créer de pods trop gourmands en ressources pour nos hôtes sous-jacents.

Définissons un pod très simple nommé mywebservice qui a un conteneur dans sa spécification web-1-10 utilisant l' image du conteneur nginx et exposant le port 80. Ajoutez le texte suivant dans un fichier appelé pod.yaml.

```
apiVersion: v1
kind: Pod
metadata:
  name: mywebservice
spec:
  containers:
  - name: web-1-10
    image: nginx:1.10
    ports:
    - containerPort: 80
    
```

Exécutez la commande kubectl create pour créer votre pod. Si vous définissez votre espace de noms ci-dessus à l'aide de la commande set-context, les pods seront créés dans l'espace de noms spécifié. Vous pouvez vérifier l'état de votre pod en exécutant la commande get pods . Une fois que vous avez terminé, nous pouvons supprimer le pod en exécutant la commande kubectl delete.

```
$ kubectl create -f ./pod.yaml
pod "mywebservice" created
$ kubectl get pods
NAME           READY     STATUS    RESTARTS   AGE
mywebservice   1/1       Running   0          37s
$ kubectl delete -f pod.yaml
pod "mywebservice" deleted

```



Vous devriez également pouvoir voir votre pod dans l'interface utilisateur de Rancher en sélectionnant Kubernetes> Pods dans la barre de menu du haut.

![rancher2](http://cdn.rancher.com/wp-content/uploads/2016/08/13025234/Screen-Shot-2016-08-13-at-5.52.20-AM.png)

# REPLICA SETS

Les Replica Sets, comme son nom l'indique, définissent le nombre de répliques de chaque pod en cours d'exécution. Ils surveillent également et veillent à ce que le nombre requis de dosettes fonctionne, en remplaçant les dosettes qui meurent. Notez que les jeux de réplicas remplacent les contrôleurs de réplication. Toutefois, dans la plupart des cas d'utilisation, vous n'utiliserez pas les jeux de réplicas directement, mais utiliserez plutôt des déploiements . Les déploiements enveloppent les jeux de réplicas et ajoutent la fonctionnalité pour faire des mises à jour tournantes vers votre application.

# DÉPLOIEMENTS
Les déploiements sont un mécanisme déclaratif pour gérer les mises à jour tournantes de votre application. Dans cet esprit, définissons notre premier déploiement en utilisant la définition de pod ci-dessus. La seule différence est que nous supprimons le paramètre name, car le nom de notre conteneur sera généré automatiquement par le déploiement. Le texte ci-dessous montre la configuration de notre déploiement. copiez-le dans un fichier appelé deployment.yaml.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mywebservice-deployment
spec:
  replicas: 2 # We want two pods for this deployment
  template:
    metadata:
      labels:
        app: mywebservice
    spec:
      containers:
      - name: web-1-10
        image: nginx:1.10
        ports:
        - containerPort: 80
```

Exécutez la commande kubectl create pour créer votre pod. Si vous définissez votre espace de noms ci-dessus à l'aide de la commande set-context, les pods seront créés dans l'espace de noms spécifié. Vous pouvez vérifier l'état de votre pod en exécutant la commande get pods . Une fois que vous avez terminé, nous pouvons supprimer le pod en exécutant la commande kubectl delete.

```
$ kubectl create -f ./deployment.yaml
deployment "mywebservice-deployment" create
$ kubectl get deployments
NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
mywebservice-deployment   2         2         2            2           7m

```

Vous pouvez obtenir des détails sur votre déploiement à l'aide de la commande de déploiement describe. L'un des éléments utiles générés par la commande describe est un ensemble d'événements. Un exemple tronqué de la sortie de la commande describe est illustré ci-dessous. Actuellement, votre déploiement ne doit comporter qu'un seul événement avec le message suivant:Scaled up replica set ... to 2.



```
$ kubectl describe deployment mywebservice
Name:                  mywebservice-deployment
Namespace:             beta
CreationTimestamp:     Sat, 20 Jan 2018 06:26:44 -0400
Labels:                app=mywebservice
.....
..... Scaled up replica set mywebservice-deployment-3208086093 to 2

```

# SCALING DEPLOYMENTS
Vous pouvez modifier l'échelle du déploiement en mettant à jour le fichier deployment.yaml précédemment pour remplacer les répliques: 2 avec des  répliques: 3 et exécutez la commande apply illustrée ci-dessous. Si vous exécutez la commande describe déploiement à nouveau , vous verrez un deuxième événement avec le message: Scaled up replica set mywebservice-deployment-3208086093 to 3.

```
$ kubectl apply -f deployment.yaml
deployment "mywebservice-deployment" configured

```

# UPDATING DEPLOYMENTS
Vous pouvez également utiliser la commande apply pour mettre à jour votre application en modifiant la version de l'image. Modifiez le fichier deployment.yaml précédemment pour remplacer l' image: nginx: 1.10 à image: nginx: 1.11 et exécutez la commande kubectl apply. Si vous exécutez à nouveau la commande describe deployment, vous verrez de nouveaux événements dont les messages sont affichés ci-dessous. Vous pouvez voir comment le nouveau déploiement (2303032576) a été mis à l'échelle et l'ancien déploiement (3208086093) a été réduit et les étapes en cours. Le nombre total de pods sur les deux déploiements est maintenu constant mais les pods sont progressivement déplacés de l'ancien vers le nouveau. Cela nous permet d'exécuter des déploiements en charge sans interruption de service.

```
Scaled up replica set mywebservice-deployment-2303032576 to 1
Scaled down replica set mywebservice-deployment-3208086093 to 2
Scaled up replica set mywebservice-deployment-2303032576 to 2
Scaled down replica set mywebservice-deployment-3208086093 to 1
Scaled up replica set mywebservice-deployment-2303032576 to 3
Scaled down replica set mywebservice-deployment-3208086093 to 0

```
Si, pendant ou après le déploiement, vous vous rendez compte que quelque chose ne va pas et que le déploiement a causé des problèmes, vous pouvez utiliser la commande de déploiement pour annuler votre changement de déploiement. Cela appliquera l'opération inverse à celle ci-dessus et ramènera la charge à la version précédente du conteneur.

```
$ kubectl rollout undo deployment/mywebservice-deployment
deployment "mywebservice-deployment" rolled back

```

# HEALTH CHECK
Avec les déploiements, nous avons vu comment faire évoluer notre service de haut en bas, ainsi que comment faire les déploiements eux-mêmes. Toutefois, lors de l'exécution de services en production, il est également important de surveiller et de remplacer les instances de service en temps réel. Kubernetes fournit des contrôles de santé pour résoudre ce problème. Mettez à jour le fichier deployment.yaml en ajoutant une configuration livenessProbe dans la section spec. Il existe trois types de sondes de vivacité, http , tcp et exec de conteneur. Les deux premiers vérifieront si Kubernetes est capable d'établir une connexion http ou tcp avec le port spécifié. Le conteneur exec probe exécute une commande spécifiée à l'intérieur du conteneur et affirme un code de réponse nul. Dans l'extrait ci-dessous, nous utilisons la sonde http pour émettre une requête GET au port 80 à l'URL racine.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mywebservice-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: mywebservice
    spec:
      containers:
      - name: web-1-11
        image: nginx:1.11
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          timeoutSeconds: 1
          
```

Si vous recréez votre déploiement avec le déploiement helthcheck supplémentaire et exécutez describe, vous devriez voir que Kubernetes vous indique maintenant que 3 de vos réplicas ne sont pas disponibles. Si vous exécutez à nouveau la description après la période initiale de 30 secondes, vous verrez que les réplicas sont maintenant marqués comme disponibles. C'est un bon moyen de s'assurer que vos conteneurs sont sains et de laisser à votre application le temps d'apparaître avant que Kubernetes commence à router le trafic vers elle.

```
$ kubectl create -f deployment.yaml
deployment "mywebservice-deployment" created
$ kubectl describe deployment mywebservice
...
Replicas: 3 updated | 3 total | 0 available | 3 unavailable

```

# UN SERVICE
Maintenant que nous avons un déploiement surveillé et évolutif qui peut être mis à jour en charge, il est temps d'exposer le service à de vrais utilisateurs. Copiez le texte suivant dans un fichier appelé service.yaml. Chaque nœud de votre cluster expose un port qui peut router le trafic vers les répliques à l'aide du proxy Kube.

```
apiVersion: v1
kind: Service
metadata:
  name: mywebservice
  labels:
    run: mywebservice
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: mywebservice
    
```

Avec le fichier service.yaml, nous créons un service à l'aide de la commande create, puis nous pouvons rechercher le NodePort à l'aide de la commande describe service. Par exemple, dans mon service, je peux accéder à l'application sur le port 31673 sur l'un de mes nœuds d'agent Kubernetes / Rancher. Kubernetes acheminera automatiquement le trafic vers les nœuds disponibles si les nœuds sont mis à l'échelle de haut en bas, deviennent malsains ou sont relancés.

```
$ kubectl create -f service.yaml
service "mywebservice" created
$ kubectl describe service mywebservice | grep NodePort
NodePort:              http       31673/TCP

```


## Volumes persistants
Avant de lancer notre application go-auth, nous devons configurer une base de données pour la connexion. Avant de configurer un serveur de base de données dans Kubernetes, nous devons lui fournir un volume de stockage persistant. Cela aidera à rendre l'état de la base de données persistant à travers les redémarrages de la base de données et dans la migration de la mémoire lorsque les conteneurs sont déplacés d'un hôte à un autre. La liste des types de volumes persistants actuellement pris en charge est répertoriée ci-dessous:


> GCEPersistentDisk
> AWSElasticBlockStore
> AzureFile
> FC (Fibre Channel)
> NFS
> iSCSI
> RBD (Ceph Block Device)
> CephFS
> Cinder (stockage de bloc OpenStack)
> Glusterfs
> VsphereVolume
> HostPath (les tests ne fonctionneront pas dans les clusters multihost)

Nous allons utiliser des volumes basés sur NFS, car NFS est omniprésent dans les systèmes de stockage réseau. Si vous n'avez pas de serveur NFS à portée de main, vous pouvez utiliser Amazon Elastic File Store pour configurer rapidement un volume NFS montable. Une fois que vous avez votre volume NFS (ou volume EFS), vous pouvez configurer un volume persistant dans Kubernetes en utilisant la spécification suivante. Nous spécifions le nom d'hôte ou l'adresse IP de notre serveur NFS / EFS, 1 Go de stockage avec lecture, écriture de nombreux mode d'accès.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: us-east-1a.fs-f604cbbf.efs.us-east-1.amazonaws.com
    path: "/"
    
```    
Une fois que vous avez créé votre volume à l'aide de kubectl create -f persistent-volume.yaml,  vous pouvez utiliser la commande suivante pour lister votre volume nouvellement créé:

```
$kubectl get pv
NAME                CAPACITY   ACCESSMODES   STATUS      CLAIM     REASON    AGE
mysql-volume          1Gi         RWX      Available                         29s

```


Réclamation de volume persistante
Maintenant que nous avons notre volume, nous pouvons créer une revendication de volume persistante en utilisant la spécification ci-dessous. Une revendication de volume persistante réserve notre volume persistant et peut ensuite être montée dans un volume de conteneur. Les spécifications que nous fournissons pour notre demande sont utilisées pour faire correspondre les volumes persistants disponibles et les lier s'ils sont trouvés. Par exemple, nous avons spécifié que nous ne voulons qu'un volume ReadWriteMany avec au moins 1 Go de stockage disponible:

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-claim
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
      
```
Nous pouvons voir si notre revendication a été capable de se lier à un volume en utilisant la commande ci-dessous:

```
$kubectl get pvc
NAME          STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
mysql-claim   Bound     nfs       1Gi       RWX           13s

```

## Secrets Management
Avant de commencer à utiliser notre volume et notre revendication persistants dans un conteneur MySQL, nous devons également trouver comment obtenir un secret, tel qu'un mot de passe de base de données, dans les modules Kubernetes. Heureusement, Kubernetes fournit un système de gestion des secrets à cette fin. Pour créer un secret géré pour le mot de passe de la base de données, créez un fichier appelé password.txt et ajoutez votre mot de passe en texte brut ici. Assurez-vous qu'il n'y a pas de caractères de nouvelle ligne dans ce fichier car ils feront partie du secret. Une fois que vous avez créé votre fichier de mot de passe, utilisez la commande suivante pour stocker votre secret dans Kubernetes:

```
$kubectl create secret generic mysql-pass --from-file=password.txt
secret "mysql-pass" created
```
Vous pouvez regarder une liste de tous les secrets actuels en utilisant la commande suivante:

```
$kubectl get secret
NAME         TYPE      DATA      AGE
mysql-pass   Opaque    1         3m
```

## Déploiement MySQL

Maintenant nous avons toutes les pièces requises, nous pouvons configurer notre déploiement MySQL en utilisant la spécification ci-dessous. Quelques points intéressants à noter: dans la spécification, nous utilisons la stratégie recréer, ce qui signifie qu'une mise à jour du déploiement supprimera tous les conteneurs et les créera à nouveau plutôt que d'utiliser un déploiement par roulement. Ceci est nécessaire car nous voulons seulement qu'un conteneur MySQL accède au volume persistant. Cependant, cela signifie également qu'il y aura des temps d'arrêt si nous redéployons notre base de données. Deuxièmement, nous utilisons les valeurs valueFrom et secretKeyRefparamètres pour injecter le secret que nous avons créé plus tôt dans notre conteneur en tant que variable d'environnement. Enfin, notez dans la section des ports que nous pouvons nommer notre port et dans les conteneurs en aval nous ferons référence au port par son nom, et non sa valeur. Cela nous permet de changer le port dans les déploiements futurs sans avoir à mettre à jour nos conteneurs en aval.


```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: go-auth-mysql
  labels:
    app: go-auth
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: go-auth
        component: mysql
    spec:
      containers:
      - image: mysql:5.6
      name: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysql-pass
            key: password.txt
      ports:
      - containerPort: 3306
        name: mysql
      volumeMounts:
      - name: mysql-persistent-storage
        mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-claim
```

## Service MySQL

Une fois que nous avons un déploiement MySQL, nous devons y attacher une interface de service afin qu'elle soit accessible aux autres services de notre application. Pour créer un service, nous pouvons utiliser la spécification suivante. Notez que nous pourrions spécifier une IP de cluster dans cette spécification si nous voulions lier statiquement notre couche d'application à ce service de base de données. Cependant, nous utiliserons les mécanismes de découverte de service dans Kubernetes pour éviter les adresses IP codées en dur.

```
apiVersion: v1
kind: Service
metadata:
  name: go-auth-mysql
  labels:
    app: go-auth
    component: mysql
spec:
  ports:
  - port: 3306
  selector:
    app: go-auth
    component: mysql
  ClusterIP: 10.43.204.178

```

Dans Kubernetes, la découverte de service est disponible via les variables d'environnement de style de lien Docker. Tous les services d'un cluster sont visibles par tous les conteneurs / pod du cluster. Kubernetes utilise les règles des tables IP pour rediriger les demandes de service vers le proxy Kube qui, à son tour, achemine les hôtes et les pods avec le service requis. Par exemple, si vous utilisez l' exécutable kubectl CONTAINER_NAME bash dans un conteneur et que vous exécutez env, vous pouvez voir les variables de liaison de service comme indiqué ci-dessous. Nous utiliserons cette configuration pour connecter notre application Web go-auth à la base de données.

```
$env | grep GO_AUTH_MYSQL_SERVICE
GO_AUTH_MYSQL_SERVICE_PORT=3306
GO_AUTH_MYSQL_SERVICE_HOST=10.43.204.178
```

Go Auth Déploiement
Maintenant que nous avons enfin notre base de données exposée, nous pouvons enfin faire apparaître notre couche Web. Nous utiliserons la spécification ci-dessous pour notre couche Web. Nous allons utiliser l'image usman / go-auth-kubernetes , qui utilise un script d'initialisation pour ajouter le service de base de données Cluster IP à / etc / hosts. Si vous utilisez le DNS ajouter dans Kubernetes, vous pouvez passer cette étape. Nous utilisons également la fonctionnalité de gestion des secrets de Kubernetes pour monter le secret mysql-pass dans le conteneur. En utilisant le paramètre args , nous spécifions l'argument db-host en tant qu'hôte mysql que nous installons dans / etc / hosts. De plus, nous spécifions db-password-filepour que notre application se connecte au cluster mysql. Nous utilisons également l'élément livenessProbe pour surveiller notre conteneur de service Web. Si le processus a des problèmes, Kubernetes détectera la panne et remplacera le pod automatiquement.

 
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: go-auth-web
spec:
  replicas: 2 # We want two pods for this deployment
  template:
    metadata:
      labels:
        app: go-auth
        component: web
    spec:
      containers:
      - name: go-auth-web
        image: usman/go-auth-kubernetes
        ports:
        - containerPort: 8080
        args:
          - "-l debug"
          - run
          - "--db-host mysql"
          - "--db-user root"
          - "--db-password-file /etc/db/password.txt"
          - "-p 8080"
        volumeMounts:
        - name: mysql-password-volume
          mountPath: "/etc/db"
          readOnly: true
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          timeoutSeconds: 1
      volumes:
      - name: mysql-password-volume
        secret:
          secretName: mysql-pass
```

## Exposer les services publics

Maintenant que nous avons configuré notre service go-auth, nous pouvons exposer le service avec les spécifications suivantes. Nous spécifions que nous utilisons le type de service NodePort qui expose le service sur un port donné de la gamme Kubernetes (30 000-32 767) sur chaque hôte Kubernetes. L'hôte utilise ensuite kubeproxy pour acheminer le trafic vers l'un des modules du déploiement go-auth. Nous pouvons maintenant utiliser un DNS circulaire ou un load balancer externe pour acheminer le trafic vers tous les nœuds Kubernetes afin de garantir la tolérance aux pannes et de répartir la charge.

```
apiVersion: v1
kind: Service
metadata:
  name: go-auth-web
  labels:
    app: go-auth
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30000
    protocol: TCP
    name: http
  selector:
    app: go-auth
    component: web
  
```  
Avec notre service exposé, nous pouvons utiliser l'API REST go-auth pour créer un utilisateur, générer un jeton pour l'utilisateur et vérifier le jeton à l'aide des commandes suivantes. Ces commandes fonctionneront même si vous tuez l'un des conteneurs go-auth-web. Ils fonctionneront également si vous supprimez le conteneur MySQL (après un moment où il est remplacé).

```  
curl -i -X PUT -d userid=USERNAME \
     -d password=PASSWORD KUBERNETES_HOST:30000/user
curl 'http://KUBERNETES_HOST:30000/token?userid=USERNAME&password=PASSWORD'
curl -i -X POST 'KUBERNETES_HOST:30000/token/USERNAME' \
     --data "IHuzUHUuqCk5b5FVesX5LWBsqm8K...."
```     
     
## Wrap up
Avec la configuration de nos services, nous avons à la fois un service et un déploiement MySQL persistant, ainsi qu'un déploiement Web sans état pour le service go-auth. Nous pouvons terminer le conteneur MySQL et il redémarrera sans perdre l'état (bien qu'il y aura un temps d'arrêt temporaire). Vous pouvez également monter le même volume NFS qu'un volume en lecture seule pour les esclaves MySQL afin de permettre les lectures même si le maître est en panne et en cours de remplacement. Dans les prochains articles, nous couvrirons l'utilisation de Pet Sets et de bases de données répliquées de la couche d'application de style Cassandra pour avoir des couches persistantes qui tolèrent l'échec sans interruption. Pour la couche Web sans état, nous prenons déjà en charge la reprise sur incident sans temps d'arrêt. En plus de nos services et de nos déploiements, nous avons examiné comment gérer les secrets de notre cluster de sorte qu'ils ne puissent être exposés à l'application qu'au moment de l'exécution. Enfin,

Kubernetes peut être décourageant avec sa pléthore de terminologie et de verbosité. Cependant, si vous avez besoin d'exécuter des charges de travail en production sous charge, Kubernetes fournit une grande partie de la plomberie que vous auriez à faire manuellement.


# License
[MIT](LICENSE)
