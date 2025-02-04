# Documentation du Projet

Ce projet se décline en **trois branches principales**, chacune adaptée à un mode d'initialisation et de déploiement spécifique. Chaque branche possède sa propre documentation détaillée afin de vous guider dans l'utilisation et la configuration de l'environnement.

---

## Branches du Projet

### 1. `swarm_config`

**Description :**  
Cette branche est dédiée à l'initialisation d'un **Swarm Docker**. Elle contient tous les scripts et configurations nécessaires pour démarrer et gérer un cluster Swarm de manière automatisée avec **Vagrant**.

**Documentation :**  
Consultez la documentation spécifique dans cette branche pour connaître les prérequis, les étapes d'installation et les options de configuration du Swarm.

---

### 2. `mano`

**Description :**  
La branche `mano` est conçue pour l'initialisation **manuelle** du cluster. Elle détaille la procédure pour créer **3 machines virtuelles (VM)** et construire le cluster à la main. Cette approche vous permet de comprendre en profondeur chaque étape du déploiement.  
Chaque étape est expliquée.

**Documentation :**  
La documentation contenue dans cette branche explique en détail comment créer et configurer chaque VM, ainsi que la procédure complète pour assembler le cluster manuellement.

---

### 3. `docker_dev`

**Description :**  
Pour les environnements de développement locaux ou pour ceux qui ne souhaitent pas utiliser un cluster Swarm, la branche `docker_dev` propose une solution simple avec un fichier `docker-compose.yml`. Ce fichier permet d'initialiser et de gérer les conteneurs Docker en local, sans nécessiter la configuration d'un cluster.

**Documentation :**  
Vous trouverez dans cette branche une documentation détaillée sur l'utilisation du fichier `docker-compose.yml`, incluant les instructions d'installation et de démarrage des conteneurs Docker en local.

---

## Choisir la Branche Adaptée

- **Cluster Swarm :**  
  Basculez sur la branche `swarm_config` pour une mise en place automatisée et scalable du cluster via Docker Swarm.

- **Cluster Manuel :**  
  Utilisez la branche `mano` si vous souhaitez créer et configurer le cluster manuellement à l'aide de 3 machines virtuelles.

- **Développement Local :**  
  Optez pour la branche `docker_dev` pour travailler en local sans la complexité d'un cluster, grâce à l'utilisation de Docker Compose.
