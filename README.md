
# Déploiement Manuel d'un Cluster Docker Swarm

Ce guide décrit pas à pas la mise en place manuelle d'un cluster Docker Swarm sur des machines virtuelles (créées avec Vagrant par exemple) afin de déployer une application multi-services (web, worker, result, redis, postgres). Vous trouverez ici les étapes pour :

- Mettre en route vos machines virtuelles via Vagrant.
- Cloner le dépôt contenant le code de l'application.
- Préparer les machines (installation de Docker, Docker Compose et NFS).
- Configurer le cluster Swarm.
- **Construire, taguer et pousser les images Docker sur Docker Hub.**
- Déployer la stack via un fichier docker-compose.yml.

> **Remarque :** Cette procédure s’appuie sur la configuration suivante :
> 
> - **Manager :** manager1 (IP : 192.168.50.106)
> - **Workers :** worker1 (IP : 192.168.50.107) et worker2 (IP : 192.168.50.108)
> - Les volumes de persistance (pour Redis et PostgreSQL) sont montés en NFS depuis le manager.

## Table des Matières

1. Prérequis  
2. Mise en Route des Machines avec Vagrant  
3. Clonage du Dépôt  
4. Préparation des Machines  
   - Installation de Docker et Docker Compose  
   - Installation et Configuration de NFS  
5. Configuration Manuelle du Cluster Swarm  
   - Sur le Nœud Manager  
   - Sur les Nœuds Workers  
6. Construction, Taggage et Poussée des Images Docker sur Docker Hub  
7. Création et Déploiement de la Stack  
8. Vérifications et Accès aux Services  
9. Récapitulatif  

---

## 1. Prérequis

- **Vagrant** installé sur votre machine hôte.  
  Pour installer Vagrant, consultez la [documentation officielle](https://www.vagrantup.com/docs/installation).
- **VMware Workstation** (ou un autre provider compatible) pour construire les VMs.
- Le fichier **Vagrantfile** définit trois machines virtuelles :  
  - **manager1** (IP : 192.168.50.106)  
  - **worker1** (IP : 192.168.50.107)  
  - **worker2** (IP : 192.168.50.108)
- Un accès SSH et les droits administrateur (sudo) sur chaque machine.
- Un compte Docker Hub (pensez à vous connecter sur chaque nœud avec `docker login`).

---

## 2. Mise en Route des Machines avec Vagrant

1. Placez-vous dans le répertoire du projet où se trouve le fichier **Vagrantfile** (par exemple, `./3DOKR-/Vagrantfile`).
2. Démarrez toutes les machines en exécutant la commande :

   ```bash
   vagrant up
   ```

   Cette commande lira le Vagrantfile et lancera la création et le provisionnement de toutes les VM définies (manager1, worker1 et worker2).

3. Vérifiez l'état des machines avec :

   ```bash
   vagrant status
   ```

   Vous devriez voir que toutes les machines sont en état `running`.

4. (Optionnel) Pour accéder à une machine, par exemple le manager :

   ```bash
   vagrant ssh manager1
   ```

---

## 3. Clonage du Dépôt

1. Connectez-vous au nœud **manager** :

   ```bash
   vagrant ssh manager1
   ```

2. Clonez le dépôt dans le répertoire `/home/vagrant/3DOKR` (si le dossier n'existe pas déjà) :

   ```bash
   if [ ! -d "/home/vagrant/3DOKR" ]; then
     git clone https://github.com/ECG-Rudy-Glt/3DOKR-.git /home/vagrant/3DOKR
   fi
   ```

3. Positionnez-vous dans le dépôt cloné et passez à la branche souhaitée (ici, par exemple, **mano**) :

   ```bash
   cd /home/vagrant/3DOKR
   git checkout mano
   cd /home/vagrant/3DOKR/voting-app
   ```

   Ce dépôt contient l’ensemble du code ainsi que le fichier **docker-compose.yml** nécessaire pour le déploiement.

---

## 4. Préparation des Machines

### Installation de Docker et Docker Compose

Sur chaque VM (manager et workers), procédez comme suit :

1. **Mettre à jour le système :**

   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

2. **Installer Docker :**

   ```bash
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh
   ```

3. **Installer Docker Compose** (exemple avec la version 1.29.2) :

   ```bash
   sudo mkdir -p /usr/local/bin
   sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   ```

4. **Ajouter l’utilisateur courant (ici `vagrant`) au groupe docker :**

   ```bash
   sudo usermod -aG docker vagrant
   ```

   *Note : Déconnectez-vous/reconnectez-vous pour que les changements de groupe prennent effet.*

5. **En cas de problème de résolution DNS :**

   ```bash
   sudo rm /etc/resolv.conf
   sudo bash -c "echo 'nameserver 8.8.8.8' > /etc/resolv.conf"
   sudo bash -c "echo 'nameserver 1.1.1.1' >> /etc/resolv.conf"
   ```

### Installation et Configuration de NFS

#### Sur le nœud Manager (manager1) – Serveur NFS

1. **Installer le serveur NFS :**

   ```bash
   sudo apt-get install -y nfs-kernel-server
   ```

2. **Créer les répertoires d’export et régler les permissions :**

   ```bash
   sudo mkdir -p /export/redis_data /export/postgres_data
   sudo chown -R nobody:nogroup /export
   sudo chmod 777 /export/redis_data /export/postgres_data
   ```

3. **Configurer les exports NFS :**

   ```bash
   echo "/export/redis_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   echo "/export/postgres_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -a
   sudo systemctl restart nfs-kernel-server
   ```

#### Sur les nœuds Workers (worker1, worker2) – Clients NFS

1. Connectez-vous à chacun des workers :

   ```bash
   vagrant ssh worker1
   vagrant ssh worker2
   ```

2. **Installer le client NFS :**

   ```bash
   sudo apt-get install -y nfs-common
   ```

3. *(Optionnel)* Vous pouvez tester le montage d’un volume NFS :

   ```bash
   sudo mount -t nfs 192.168.50.106:/export/redis_data /chemin/local/redis_data
   ```

---

## 5. Configuration Manuelle du Cluster Swarm

### Sur le Nœud Manager (manager1)

1. *(Optionnel)* Quitter un Swarm existant :

   ```bash
   docker swarm leave -f
   ```

2. **Initialiser le Swarm en utilisant l’IP privée du manager :**

   ```bash
   docker swarm init --advertise-addr 192.168.50.106
   ```

3. **Créer le réseau overlay nécessaire à la communication entre services :**

   ```bash
   docker network create --driver overlay --attachable --subnet=10.0.0.0/16 my_network
   ```

   > **Note :** Répétez cette opération pour chaque réseau défini dans le fichier docker-compose, si nécessaire.

4. **Récupérer le token de jointure pour les workers :**

   ```bash
   docker swarm join-token worker -q
   ```

   Notez ce token, il sera utilisé pour que les workers rejoignent le Swarm.

### Sur les Nœuds Workers (worker1 et worker2)

1. *(Optionnel)* Quitter un Swarm existant :

   ```bash
   docker swarm leave -f
   ```

2. **Rejoindre le Swarm en utilisant le token récupéré (remplacez `<TOKEN_WORKER>` par le token obtenu) :**

   Par exemple, pour **worker1** :

   ```bash
   docker swarm join --advertise-addr 192.168.50.107 --token <TOKEN_WORKER> 192.168.50.106:2377
   ```

   Répétez la commande pour **worker2** en remplaçant l'IP par 192.168.50.108.

---

## 6. Construction, Taggage et Poussée des Images Docker sur Docker Hub

Plutôt que de précharger manuellement les images sur chaque nœud, vous allez construire les images sur le manager, les taguer avec votre identifiant Docker Hub, puis les pousser sur Docker Hub afin que chaque nœud du cluster puisse les récupérer automatiquement.

> **Important :** Assurez-vous que chaque machine (manager et workers) est connectée à Internet et que vous êtes connecté à Docker Hub via la commande `docker login`.

### Pour chaque service, procédez comme suit :

#### Service Web (Python – dossier vote/)

1. **Accédez au dossier du service :**

   ```bash
   cd /home/vagrant/3DOKR/voting-app/vote
   ```

2. **Construisez l’image et taguez-la** (remplacez `<DOCKER_HUB_USER>` par votre nom d'utilisateur Docker Hub) :

   ```bash
   docker build -t <DOCKER_HUB_USER>/voting_web:latest .
   ```

3. **Poussez l’image sur Docker Hub :**

   ```bash
   docker push <DOCKER_HUB_USER>/voting_web:latest
   ```

#### Service Worker (.NET – dossier worker/)

1. **Accédez au dossier du service :**

   ```bash
   cd /home/vagrant/3DOKR/voting-app/worker
   ```

2. **Construisez l’image et taguez-la** :

   ```bash
   docker build -t <DOCKER_HUB_USER>/voting_worker:latest .
   ```

3. **Poussez l’image sur Docker Hub :**

   ```bash
   docker push <DOCKER_HUB_USER>/voting_worker:latest
   ```

#### Service Result (Node.js – dossier result/)

1. **Accédez au dossier du service :**

   ```bash
   cd /home/vagrant/3DOKR/voting-app/result
   ```

2. **Construisez l’image et taguez-la** :

   ```bash
   docker build -t <DOCKER_HUB_USER>/voting_result:latest .
   ```

3. **Poussez l’image sur Docker Hub :**

   ```bash
   docker push <DOCKER_HUB_USER>/voting_result:latest
   ```

> **Conseil :** Vous pouvez également automatiser ces étapes via un script de provisionnement sur le manager.

### Mise à jour du fichier docker-compose.yml

Modifiez votre fichier **docker-compose.yml** pour utiliser les images depuis Docker Hub. Par exemple, pour le service web :

```yaml
services:
  web:
    image: <DOCKER_HUB_USER>/voting_web:latest
    ports:
      - "8080:8080"
    depends_on:
      - redis
      - postgres
    networks:
      - front
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
  ...
```

Faites de même pour les services **worker** et **result** en utilisant les images `<DOCKER_HUB_USER>/voting_worker:latest` et `<DOCKER_HUB_USER>/voting_result:latest`.

---

## 7. Création et Déploiement de la Stack

1. **Placez-vous au niveau du fichier docker-compose.yml :**

   ```bash
   cd /home/vagrant/3DOKR/voting-app
   ```

2. **Déployez la stack Docker Swarm depuis le manager :**

   ```bash
   docker stack deploy -c docker-compose.yml voting
   ```

   Docker va déployer les services sur le cluster en respectant le nombre de répliques et les stratégies de mise à jour définies.

---

## 8. Vérifications et Accès aux Services

1. **Vérifiez l’état de la stack :**

   ```bash
   docker stack services voting
   ```

   Vous devriez obtenir un résultat similaire à :

   ```
   ID             NAME              MODE         REPLICAS   IMAGE                                PORTS
   d5ydk4n8soj3   voting_postgres   replicated   1/1        postgres:alpine                      *:5432->5432/tcp
   iwrd3ys5ajym   voting_redis      replicated   1/1        redis:7.4.2-alpine                   *:6379->6379/tcp
   ubfo3cfcg5ee   voting_result     replicated   2/2        <DOCKER_HUB_USER>/voting_result:latest *:8888->8888/tcp
   7m7lort2szb0   voting_web        replicated   2/2        <DOCKER_HUB_USER>/voting_web:latest     *:8080->8080/tcp
   xmkjmqwockxv   voting_worker     replicated   2/2        <DOCKER_HUB_USER>/voting_worker:latest
   ```

2. **Vérifiez les containers déployés :**

   ```bash
   docker service ls
   docker service ps <NOM_DU_SERVICE>
   ```

3. **Accédez aux applications :**

   - Pour le service web, rendez-vous sur [http://192.168.50.106:8080](http://192.168.50.106:8080)
   - Pour le service result, rendez-vous sur [http://192.168.50.106:8888](http://192.168.50.106:8888)

> **Astuce :** En cas de problème avec les interfaces réseau, si `eth1` ne fonctionne pas, utilisez l'adresse associée à `eth0`.

