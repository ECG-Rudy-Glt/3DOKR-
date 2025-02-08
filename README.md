# Déploiement Manuel d'un Cluster Docker Swarm

Ce guide décrit pas à pas la mise en place manuelle d'un cluster Docker Swarm sur des machines virtuelles (créées avec Vagrant par exemple) afin de déployer une application multi-services (web, worker, result, redis, postgres). Vous trouverez ici les étapes pour :

- Mettre en route vos machines virtuelles via Vagrant.
- Cloner le dépôt contenant le code de l'application.
- Préparer les machines (installation de Docker, Docker Compose et NFS).
- Configurer le cluster Swarm.
- Construire les images Docker.
- Déployer la stack via un fichier `docker-compose.yml`.

> **Remarque :** Cette procédure s’appuie sur la configuration suivante :
>
> - **Manager** : `manager1` (IP : `192.168.50.106`)
> - **Workers** : `worker1` (IP : `192.168.50.107`) et `worker2` (IP : `192.168.50.108`)
>
> Les volumes de persistance (pour Redis et PostgreSQL) sont montés en NFS depuis le manager.

---

## Table des Matières

1. [Prérequis](#prérequis)
2. [Mise en Route des Machines avec Vagrant](#mise-en-route-des-machines-avec-vagrant)
3. [Clonage du Dépôt](#clonage-du-dépôt)
4. [Préparation des Machines](#préparation-des-machines)
   - [Installation de Docker et Docker Compose](#installation-de-docker-et-docker-compose)
   - [Installation et Configuration de NFS](#installation-et-configuration-de-nfs)
5. [Configuration Manuelle du Cluster Swarm](#configuration-manuelle-du-cluster-swarm)
   - [Sur le Nœud Manager](#sur-le-nœud-manager)
   - [Sur les Nœuds Workers](#sur-les-nœuds-workers)
6. [Construction des Images Docker](#construction-des-images-docker)
7. [Création et Déploiement de la Stack](#création-et-déploiement-de-la-stack)
8. [Vérifications et Accès aux Services](#vérifications-et-accès-aux-services)
9. [Récapitulatif](#récapitulatif)

---

## 1. Prérequis

- **Vagrant installé** sur votre machine hôte.  
  Pour installer Vagrant, consultez la [documentation officielle](https://www.vagrantup.com/docs/installation).
- **VMware Workstation** installé sur la machine afin de build les Vms.
- Le **Vagrantfile** définit trois machines virtuelles :
  - `manager1` (IP : `192.168.50.106`)
  - `worker1` (IP : `192.168.50.107`)
  - `worker2` (IP : `192.168.50.108`)
- Un accès SSH et les droits administrateur (sudo) sur chaque machine.

---

## 2. Mise en Route des Machines avec Vagrant

1. **Placez-vous dans le répertoire du projet** où se trouve le fichier `Vagrantfile`. (./3DOKR-/vagantfile)

2. **Démarrez toutes les machines** en exécutant la commande :

   ```bash
   vagrant up
   ```

   Cette commande lira le Vagrantfile et lancera la création et le provisionnement de toutes les VM définies (`manager1`, `worker1` et `worker2`).

3. **Vérifiez l'état des machines** avec :

   ```bash
   vagrant status
   ```

   Vous devriez voir que toutes les machines sont en état `running`.

4. **(Optionnel)** Pour accéder à une machine, par exemple le manager, utilisez :

   ```bash
   vagrant ssh manager1
   ```

---

## 3. Clonage du Dépôt

**Cloner manuellement**, procédez comme suit :

1. Connectez-vous au nœud manager :

   ```bash
   vagrant ssh manager1
   ```

2. Clonez le dépôt dans le répertoire `/home/vagrant/3DOKR` (si le dossier n'existe pas déjà) :

   ```bash
   if [ ! -d "/home/vagrant/3DOKR" ]; then
     git clone https://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@github.com/ECG-Rudy-Glt/3DOKR-.git /home/vagrant/3DOKR
   fi
   ```

3. Positionnez-vous dans le dépôt cloné et passez à la branche `mano` :

   ```bash
   cd /home/vagrant/3DOKR
   git checkout mano
   cd /home/vagrant/3DOKR/voting-app
   ```

Ce dépôt contient l’ensemble du code ainsi que le fichier `docker-compose.yml` nécessaire pour le déploiement.

---

## 4. Préparation des Machines

### Installation de Docker et Docker Compose

Sur **chaque VM** (manager et workers), mettez à jour le système et installez Docker et Docker Compose :

```bash
# Mettre à jour le système
sudo apt-get update && sudo apt-get upgrade -y

# Installer Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Installer Docker Compose (exemple avec la version 1.29.2)
sudo mkdir -p /usr/local/bin
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Ajouter l’utilisateur courant (ici 'vagrant') au groupe docker (nécessite une reconnexion)
sudo usermod -aG docker vagrant
```

Si jamais vous avez un probleme lors d'un telechargement c'est sans doute une erreur de DSN.
Pour remedier à cela:

```bash
sudo rm /etc/resolv.conf
sudo bash -c "echo 'nameserver 8.8.8.8' > /etc/resolv.conf"
sudo bash -c "echo 'nameserver 1.1.1.1' >> /etc/resolv.conf"

```

re essayé après cela.

> **Note :** Déconnectez-vous/reconnectez-vous pour que les changements de groupe prennent effet.

### Installation et Configuration de NFS

Afin de persister les données de Redis et PostgreSQL, nous allons utiliser le stockage NFS.

#### Sur le nœud Manager (`manager1`) – Serveur NFS

1. **Installer le serveur NFS :**

   ```bash
   sudo apt-get install -y nfs-kernel-server
   ```

2. **Créer les répertoires d’export et régler les permissions :**

   ```bash

   sudo mkdir -p /export/redis_data /export/postgres_data  # A commenter
   sudo chown -R nobody:nogroup /export # A commenter
   sudo chmod 777 /export/redis_data /export/postgres_data # A commenter
   ```

3. **Configurer les exports NFS :**

   ```bash
   echo "/export/redis_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   echo "/export/postgres_data *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
   sudo exportfs -a
   sudo systemctl restart nfs-kernel-server
   ```

#### Sur les nœuds Workers (`worker1`, `worker2`) – Clients NFS

    ```bash
    vagrant ssh worker1
    vagrant ssh worker2
    ```

1. **Installer le client NFS :**

   ```bash
   sudo apt-get install -y nfs-common
             # Création des répertoires d'export
    mkdir -p /export/redis_data /export/postgres_data
    chown -R nobody:nogroup /export
    chmod 777 /export/redis_data /export/postgres_data

          # Configuration des exports NFS
    echo "/export/redis_data *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
    echo "/export/postgres_data *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports

          # Redémarrage du serveur NFS
    exportfs -a
    systemctl restart nfs-kernel-server
   ```

2. **(Optionnel) Monter manuellement un volume NFS pour tester :**

   ```bash
   # Exemple pour Redis
   sudo mount -t nfs 192.168.50.106:/export/redis_data /chemin/local/redis_data
   ```

---

## 5. Configuration Manuelle du Cluster Swarm

### Sur le Nœud Manager (`manager1`)

1. **(Optionnel) Quitter un Swarm existant :**

   ```bash
   docker swarm leave -f
   ```

2. **Initialiser le Swarm** en utilisant l’IP privée du manager :

   ```bash
   docker swarm init --advertise-addr 192.168.50.106
   ```

3. **Créer le réseau overlay** nécessaire à la communication entre services :

   ```bash
   docker network create --driver overlay --attachable --subnet=10.0.0.0/16 my_network # Le paramètre --subnet=10.0.0.0/16 définit la plage d’adresses IP disponibles pour les conteneurs reliés à ce réseau.
   ```

   => repeter cett opération pour chaque réseaux du docker-compose.

4. **Récupérer le token de jointure pour les workers** :

   ```bash
   docker swarm join-token worker -q
   ```

   Notez ce token, il sera utilisé pour que les workers rejoignent le Swarm.

### Sur les Nœuds Workers (`worker1` et `worker2`)

1. **(Optionnel) Quitter un Swarm existant :**

   ```bash
   docker swarm leave -f
   ```

2. **Rejoindre le Swarm** en utilisant le token récupéré (remplacez `<IP_PRIVÉE_WORKER>` par l’IP du worker, et `<TOKEN_WORKER>` par le token obtenu) :

   ```bash
   docker swarm join --advertise-addr <IP_PRIVÉE_WORKER> --token <TOKEN_WORKER> 192.168.50.106:2377
   ```

   Par exemple, pour `worker1` :

   ```bash
   docker swarm join --advertise-addr 192.168.50.107 --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxx 192.168.50.106:2377
   ```

---

## 6. Construction des Images Docker

Pour chaque service, construisez manuellement l’image à partir du répertoire correspondant.

### Service Web (Python – dossier `vote/`)

1. Accédez au dossier :

   ```bash
   cd /chemin/vers/3DOKR/voting-app/vote
   ```

2. Construisez l’image :

   ```bash
   docker build -t voting_web:latest .
   ```

> **Dockerfile Utilisé (dossier vote/) :**
>
> ```dockerfile
> FROM python:3.11-alpine
> WORKDIR /c
> COPY requirements.txt requirements.txt
> RUN pip install -r requirements.txt
> COPY . .
> EXPOSE 8080
> CMD ["python", "app.py"]
> ```

---

### Service Worker (.NET – dossier `worker/`)

1. Accédez au dossier :

   ```bash
   cd /chemin/vers/3DOKR/voting-app/worker
   ```

2. Construisez l’image :

   ```bash
   docker build -t voting_worker:latest .
   ```

> **Dockerfile Utilisé (dossier worker/) :**
>
> ```dockerfile
> FROM mcr.microsoft.com/dotnet/sdk:7.0
> WORKDIR /app
> COPY *.csproj ./
> RUN dotnet restore
> COPY . ./
> RUN dotnet publish -c Release -o out
> RUN chmod +x out/Worker.dll
> ENTRYPOINT ["dotnet", "out/Worker.dll"]
> ```

---

### Service Result (Node.js – dossier `result/`)

1. Accédez au dossier :

   ```bash
   cd /chemin/vers/3DOKR/voting-app/result
   ```

2. Construisez l’image :

   ```bash
   docker build -t voting_result:latest .
   ```

> **Dockerfile Utilisé (dossier result/) :**
>
> ```dockerfile
> FROM node:17-alpine
> WORKDIR /app
> COPY package.json package-lock.json ./
> RUN npm install
> COPY . .
> EXPOSE 8888
> CMD ["npm", "start"]
> ```

---

## 7. Création et Déploiement de la Stack

1. **placez vous au niveau du fichier `docker-compose.yml`** `/chemin/vers/3DOKR/voting-app`

   Assurez-vous que le docker-compose.yml est existant.

2. **Déployer la stack Docker Swarm**

   Depuis le manager (dans le dossier contenant le `docker-compose.yml`), exécutez :

   ```bash
   docker stack deploy -c docker-compose.yml voting
   ```

   Docker va déployer les services sur le cluster en respectant le nombre de répliques et les stratégies de mise à jour définies.

---

## 8. Vérifications et Accès aux Services

- **Vérifier l’état de la stack :**

  ```bash
  docker stack services voting
  ```

  Vous devriez obtenir un résultat similaire à :

  ```
  ID             NAME              MODE         REPLICAS   IMAGE                  PORTS
  d5ydk4n8soj3   voting_postgres   replicated   1/1        postgres:alpine        *:5432->5432/tcp
  iwrd3ys5ajym   voting_redis      replicated   1/1        redis:alpine           *:6379->6379/tcp
  ubfo3cfcg5ee   voting_result     replicated   2/2        voting_result:latest   *:8888->8888/tcp
  7m7lort2szb0   voting_web        replicated   2/2        voting_web:latest      *:8080->8080/tcp
  xmkjmqwockxv   voting_worker     replicated   1/1        voting_worker:latest
  ```

- **Vérifier les containers déployés :**

  ```bash
  docker service ls
  docker service ps <NOM_DU_SERVICE>
  ```

- **Accéder aux applications :**
  - Pour le service **web**, rendez-vous sur [http://192.168.50.106:8080](http://192.168.50.106:8080)
  - Pour le service **result**, rendez-vous sur [http://192.168.50.106:8888](http://192.168.50.106:8888)

Il se peut qu'il y ait des conflits au niveau des cartes reseau de la vm. Si eth1 ne fonctionne pas, utilisez l'adresse eth0.
