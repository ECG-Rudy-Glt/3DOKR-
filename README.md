# 🚀 Documentation de la Branche `docker_dev`

Cette branche permet de **déployer l'application en local** à l'aide de **Docker Compose** sans nécessiter la mise en place d'un cluster Swarm. L'objectif est de faciliter le développement et les tests sur une machine locale.



## 📌 Contenu de la Stack Docker Compose

L’architecture repose sur plusieurs **services conteneurisés**, qui communiquent entre eux via un réseau **bridge** (`my_network`).

### 🔹 **1. Services Déployés**

| Service    | Description                                              |
| ---------- | -------------------------------------------------------- |
| `web`      | Application web Python servant d’interface utilisateur.  |
| `redis`    | Stockage temporaire basé sur Redis.                      |
| `postgres` | Base de données PostgreSQL pour persistance des données. |
| `worker`   | Worker en .NET qui traite les événements et les données. |
| `result`   | Application Node.js qui affiche les résultats des votes. |

---

## ⚙️ **Configuration des Services**

### 🖥 **1. Service Web**

- **Construit à partir du dossier** `./vote/`
- **Accessible sur** `http://localhost:8080`
- **Dépend de** `redis` et `postgres`
- **Partage les fichiers locaux** avec le container grâce à un **volume** `.:/code`

```yaml
web:
  build: ./vote/
  ports:
    - "8080:8080"
  volumes:
    - .:/code
  depends_on:
    - redis
    - postgres
  networks:
    - my_network
```



---

### 🏠 **2. Service Redis**

- **Utilise l'image** `redis:alpine`
- **Expose le port** `6379`
- **Persiste les données dans un volume** `redis_data`
- **Inclut un healthcheck** pour vérifier que le service fonctionne

```yaml
redis:
  image: redis:alpine
  ports:
    - "6379:6379"
  volumes:
    - redis_data:/data
  networks:
    - my_network
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---

### 🗄 **3. Service PostgreSQL**

- **Utilise l'image** `postgres:11-alpine`
- **Stocke les données dans un volume** `postgres_data`
- **Charge les variables d’environnement** pour la configuration
- **Inclut un healthcheck** pour s'assurer que la base est prête

```yaml
postgres:
  image: postgres:11-alpine
  environment:
    POSTGRES_DB: ${POSTGRES_DB}
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  volumes:
    - postgres_data:/var/lib/postgresql/data
  networks:
    - my_network
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5
```

---

### 🔧 **4. Service Worker**

- **Construit à partir du dossier** `./worker/`
- **Dépend de** `redis` et `postgres` (vérifie leur état avant de démarrer)
- **Inclut un healthcheck** basé sur `.NET`

```yaml
worker:
  build: ./worker/
  depends_on:
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy
  networks:
    - my_network
  healthcheck:
    test: ["CMD", "dotnet", "--version"]
    interval: 10s
    timeout: 5s
    retries: 3
```

---

### 📊 **5. Service Result**

- **Construit à partir du dossier** `./result/`
- **Accessible sur** `http://localhost:8888`
- **Monte le dossier `./result` et `/app/node_modules` en volume**
- **Dépend du worker et de la base PostgreSQL**

```yaml
result:
  build: ./result/
  ports:
    - "8888:8888"
  volumes:
    - ./result:/app
    - /app/node_modules
  depends_on:
    worker:
      condition: service_healthy
    postgres:
      condition: service_healthy
  networks:
    - my_network
```

---

## 🔗 **Configuration des Volumes et du Réseau**

- **Volumes créés pour persister les données** :
  - `redis_data`
  - `postgres_data`
- **Réseau de communication interne** :
  - `my_network` (type `bridge`)
- **Secrets gérés via un fichier** :
  - `db_password` stocké dans `./secrets/db_password`

```yaml
volumes:
  redis_data:
  postgres_data:

networks:
  my_network:
    driver: bridge
```

---

## 🚀 **Comment Démarrer l'Application en Local ?**

### 🔹 **1. Cloner le Dépôt**

```bash
git clone https://github.com/votre-repo/docker_dev.git
cd docker_dev
```

### 🔹 **2. Créer un fichier `.env`**

Ajoutez les variables d'environnement PostgreSQL dans un fichier `.env` à la racine du projet :

```ini
POSTGRES_DB=Postgres
POSTGRES_USER=Postgres
POSTGRES_PASSWORD=Postgres
```

### 🔹 **3. Démarrer les Conteneurs**

Exécutez la commande suivante pour lancer l’application :

```bash
docker-compose up -d
```

👉 L'option `-d` permet de **démarrer les conteneurs en arrière-plan**.

### 🔹 **4. Vérifier l'état des Conteneurs**

```bash
docker ps
```

### 🔹 **5. Accéder aux Applications**

- **Interface Web** : `http://localhost:8080`
- **Résultats des Votes** : `http://localhost:8888`
- **Base de données** : Accessible en local sur le port `5432`
- **Redis** : Accessible en local sur le port `6379`

### 🔹 **6. Arrêter les Conteneurs**

Lorsque vous avez terminé vos tests, vous pouvez arrêter et supprimer les conteneurs avec :

```bash
docker-compose down
```
````
