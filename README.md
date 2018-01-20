# Exécution de WordPress hautement disponible avec MySQL sur Kubernetes

[![Apache]
![MySql]


# Scalable WordPress deployment on Kubernetes Cluster

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

À ce stade, nous avons créé la classe de stockage de revendications de volume qui va remettre des disques persistants à tous les pods qui les demandent, nous avons configuré la configuration qui définit quelques variables dans les fichiers de configuration MySQL, et nous avons configuré un réseau ... service de niveau qui va charger les demandes d'équilibrage aux serveurs MySQL. Tout ceci est juste un cadre pour les ensembles avec état, où les serveurs MySQL fonctionnent réellement, que nous explorerons ensuite.

##Configuration de MySQL avec des ensembles avec état

Dans cette section, nous allons écrire la configuration YAML pour une instance MySQL en utilisant un ensemble avec état.

Définissons notre ensemble dynamique:

1. [Créez trois pods et enregistrez-les dans le service MySQL.]
2. [Définissez le modèle suivant pour chaque pod:]
3. [Créez un conteneur d'initialisation pour le serveur MySQL maître nommé  init-mysql.]
  . [Utilisez l'  mysql:5.7 image pour ce conteneur.]
  . [Exécutez un script bash à configurer xtrabackup.]
  . [Montez deux nouveaux volumes pour la configuration et la configuration.]

4. [Créez un conteneur d'initialisation pour le serveur MySQL maître nommé  clone-mysql.]
  . [Utilisez l'image de Google Cloud Registry  xtrabackup:1.0 pour ce conteneur.]
  . [Exécutez un script bash pour cloner l'existant xtrabackupsdu pair précédent.]
  . [Montez deux nouveaux volumes pour les données et la configuration.]
  . [Ce conteneur héberge efficacement les données clonées afin que les nouveaux conteneurs esclaves puissent les récupérer.]
5. [Créez les conteneurs principaux pour les serveurs MySQL esclaves.]
  . [Créez un conteneur esclave MySQL et configurez-le pour vous connecter au maître MySQL.]
  . [Créez un xtrabackupconteneur esclave et configurez-le pour vous connecter au maître xtrabackup.]
6. [Créez un modèle de revendication de volume pour décrire chaque volume à créer en tant que disque persistant de 10 Go.]

La configuration suivante définit le comportement des maîtres et des esclaves de notre cluster MySQL, offrant une configuration bash qui exécute le client esclave et assure le bon fonctionnement d'un maître avant le clonage. Les esclaves et les maîtres ont chacun leur propre volume de 10 Go qu'ils demandent à partir de la classe de stockage de volume persistante que nous avons définie précédemment.

## Prerequisite

Create a Kubernetes cluster with either [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing, with [IBM IBM Cloud Container Service](https://github.com/IBM/container-journey-template), or [IBM Cloud Private](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/README.md) to deploy in cloud. The code here is regularly tested against [Kubernetes Cluster from IBM Cloud Container Service](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov) using Travis.

## Objectives

This scenario provides instructions for the following tasks:

- Create local persistent volumes to define persistent disks.
- Create a secret to protect sensitive data.
- Create and deploy the WordPress frontend with one or more pods.
- Create and deploy the MySQL database(either in a container or using IBM Cloud MySQL as backend).

## Deploy to IBM Cloud
If you want to deploy the WordPress directly to IBM Cloud, click on 'Deploy to IBM Cloud' button below to create an IBM Cloud DevOps service toolchain and pipeline for deploying the WordPress sample, else jump to [Steps](##steps)

[![Create Toolchain](https://metrics-tracker.mybluemix.net/stats/8201eec1bc017860952416f1cc5666ce/button.svg)](https://console.ng.bluemix.net/devops/setup/deploy/)

Please follow the [Toolchain instructions](https://github.com/IBM/container-journey-template/blob/master/Toolchain_Instructions_new.md) to complete your toolchain and pipeline.

## Steps
1. [Setup MySQL Secrets](#1-setup-mysql-secrets)
2. [Create local persistent volumes](#2-create-local-persistent-volumes)
3. [Create Services and Deployments for WordPress and MySQL](#3-create-services-and-deployments-for-wordpress-and-mysql)
  - 3.1 [Using MySQL in container](#31-using-mysql-in-container)
  - 3.2 [Using Bluemix MySQL](#32-using-bluemix-mysql-as-backend)
4. [Accessing the external WordPress link](#4-accessing-the-external-wordpress-link)
5. [Using WordPress](#5-using-wordpress)

# 1. Setup MySQL Secrets

> *Quickstart option:* In this repository, run `bash scripts/quickstart.sh`.

Create a new file called `password.txt` in the same directory and put your desired MySQL password inside `password.txt` (Could be any string with ASCII characters).


We need to make sure `password.txt` does not have any trailing newline. Use the following command to remove possible newlines.

```bash
tr -d '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
```

# 2. Create Local Persistent Volumes
To save your data beyond the lifecycle of a Kubernetes pod, you will want to create persistent volumes for your MySQL and Wordpress applications to attach to.

#### For "lite" IBM Bluemix Container Service
Create the local persistent volumes manually by running
```bash
kubectl create -f local-volumes.yaml
```
#### For paid IBM Bluemix Container Service OR Minikube
Persistent volumes are created dynamically for you when the MySQL and Wordpress applications are deployed. No action is needed.

# 3. Create Services and deployments for WordPress and MySQL

### 3.1 Using MySQL in container

> *Note:* If you want to use Bluemix Compose-MySql as your backend, please go to [Using Bluemix MySQL as backend](#32-using-bluemix-mysql-as-backend).

Install persistent volume on your cluster's local storage. Then, create the secret and services for MySQL and WordPress.

```bash
kubectl create secret generic mysql-pass --from-file=password.txt
kubectl create -f mysql-deployment.yaml
kubectl create -f wordpress-deployment.yaml
```


When all your pods are running, run the following commands to check your pod names.

```bash
kubectl get pods
```

This should return a list of pods from the kubernetes cluster.

```bash
NAME                               READY     STATUS    RESTARTS   AGE
wordpress-3772071710-58mmd         1/1       Running   0          17s
wordpress-mysql-2569670970-bd07b   1/1       Running   0          1m
```

Now please move on to [Accessing the External Link](#4-accessing-the-external-wordpress-link).

### 3.2 Using Bluemix MySQL as backend

Provision Compose for MySQL in Bluemix via https://console.ng.bluemix.net/catalog/services/compose-for-mysql

Go to Service credentials and view your credentials. Your MySQL hostname, port, user, and password are under your credential uri and it should look like this

![mysql](images/mysql.png)

Modify your `wordpress-deployment.yaml` file, change WORDPRESS_DB_HOST's value to your MySQL hostname and port(i.e. `value: <hostname>:<port>`), WORDPRESS_DB_USER's value to your MySQL user, and WORDPRESS_DB_PASSWORD's value to your MySQL password.

And the environment variables should look like this

```yaml
    spec:
      containers:
      - image: wordpress:4.7.3-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: sl-us-dal-9-portal.7.dblayer.com:22412
        - name: WORDPRESS_DB_USER
          value: admin
        - name: WORDPRESS_DB_PASSWORD
          value: XMRXTOXTDWOOPXEE
```

After you modified the `wordpress-deployment.yaml`, run the following commands to deploy WordPress.

```bash
kubectl create -f wordpress-deployment.yaml
```

When all your pods are running, run the following commands to check your pod names.

```bash
kubectl get pods
```

This should return a list of pods from the kubernetes cluster.

```bash
NAME                               READY     STATUS    RESTARTS   AGE
wordpress-3772071710-58mmd         1/1       Running   0          17s
```

# 4. Accessing the external WordPress link

> If you have a paid cluster, you can use LoadBalancer instead of NodePort by running
>
>`kubectl edit services wordpress`
>
> Under `spec`, change `type: NodePort` to `type: LoadBalancer`
>
> **Note:** Make sure you have `service "wordpress" edited` shown after editing the yaml file because that means the yaml file is successfully edited without any typo or connection errors.

You can obtain your cluster's IP address using

```bash
$ bx cs workers <your_cluster_name>
OK
ID                                                 Public IP        Private IP     Machine Type   State    Status   
kube-hou02-pa817264f1244245d38c4de72fffd527ca-w1   169.47.220.142   10.10.10.57    free           normal   Ready 
```

You will also need to run the following command to get your NodePort number.

```bash
$ kubectl get svc wordpress
NAME        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
wordpress   10.10.10.57   <nodes>       80:30180/TCP   2m
```

Congratulation. Now you can use the link **http://[IP]:[port number]** to access your WordPress site.


> **Note:** For the above example, the link would be http://169.47.220.142:30180

You can check the status of your deployment on Kubernetes UI. Run `kubectl proxy` and go to URL 'http://127.0.0.1:8001/ui' to check when the WordPress container becomes ready.

![Kubernetes Status Page](images/kube_ui.png)

> **Note:** It can take up to 5 minutes for the pods to be fully functioning.



**(Optional)** If you have more resources in your cluster, and you want to scale up your WordPress website, you can run the following commands to check your current deployments.
```bash
$ kubectl get deployments
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wordpress         1         1         1            1           23h
wordpress-mysql   1         1         1            1           23h
```

Now, you can run the following commands to scale up for WordPress frontend.
```bash
$ kubectl scale deployments/wordpress --replicas=2
deployment "wordpress" scaled
$ kubectl get deployments
NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
wordpress         2         2         2            2           23h
wordpress-mysql   1         1         1            1           23h
```
As you can see, we now have 2 pods that are running the WordPress frontend.

> **Note:** If you are a free tier user, we recommend you only scale up to 10 pods since free tier users have limited resources.

# 5. Using WordPress

Now that WordPress is running you can register as a new user and install WordPress.

![wordpress home Page](images/wordpress.png)

After installing WordPress, you can post new comments.

![wordpress comment Page](images/wordpress_comment.png)


# Troubleshooting

If you accidentally created a password with newlines and you can not authorize your MySQL service, you can delete your current secret using

```bash
kubectl delete secret mysql-pass
```

If you want to delete your services, deployments, and persistent volume claim, you can run
```bash
kubectl delete deployment,service,pvc -l app=wordpress
```

If you want to delete your persistent volume, you can run the following commands
```bash
kubectl delete -f local-volumes.yaml
```

If WordPress is taking a long time, you can debug it by inspecting the logs
```bash
kubectl get pods # Get the name of the wordpress pod
kubectl logs [wordpress pod name]
```


# References
- This WordPress example is based on Kubernetes's open source example [mysql-wordpress-pd](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd) at https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd.

# Privacy Notice

Sample Kubernetes Yaml file that includes this package may be configured to track deployments to [IBM Cloud](https://www.bluemix.net/) and other Kubernetes platforms. The following information is sent to a [Deployment Tracker](https://github.com/IBM/metrics-collector-service) service on each deployment:

* Kubernetes Cluster Provider(`IBM Cloud, Minikube, etc`)
* Kubernetes Machine ID
* Kubernetes Cluster ID (Only from IBM Cloud's cluster)
* Kubernetes Customer ID (Only from IBM Cloud's cluster)
* Environment variables in this Kubernetes Job.

This data is collected from the Kubernetes Job in the sample application's yaml file. This data is used by IBM to track metrics around deployments of sample applications to IBM Cloud to measure the usefulness of our examples so that we can continuously improve the content we offer to you. Only deployments of sample applications that include code to ping the Deployment Tracker service will be tracked.

## Disabling Deployment Tracking

Please comment out/remove the Metric Kubernetes Job portion at the end of the 'wordpress-deployment.yaml' file.


# License
[Apache 2.0](LICENSE)
