## Personne 2 — Result (Frontend des résultats)

### Rôle dans l'architecture
Le service `result` est un serveur Node.js qui lit les résultats depuis PostgreSQL et les affiche en temps réel via Socket.IO. Il permet aux utilisateurs de voir l'évolution des votes instantanément, sans recharger la page.

### Docker

**Dockerfile** (`result/Dockerfile`) :
- Utilise l'image `node:18-slim` pour garantir un environnement Node.js léger et cohérent.
- Définit le dossier de travail `/usr/local/app`.
- Installe les dépendances via `npm ci` et `nodemon` pour le développement.
- Déplace `node_modules` pour optimiser le cache Docker (accélère les builds).
- Définit la variable d'environnement `PORT=4000` et expose ce port.
- Lance le serveur avec `node server.js`.

**docker-compose.yaml** :
- Le service `result` est connecté à deux réseaux :
  - `front-net` (pour l'accès utilisateur)
  - `back-net` (pour accéder à la base PostgreSQL)
- Expose le port `4000:4000` pour l'accès externe.
- Dépend du service `db` (PostgreSQL), avec un healthcheck pour s'assurer que la base est prête avant de démarrer.

**server.js** (`result/server.js`) :
- Se connecte à PostgreSQL via l'URL `postgres://postgres:postgres@db/postgres` (le nom `db` est résolu automatiquement par Docker/Kubernetes).
- Toutes les secondes, exécute la requête :
  `SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote`
- Utilise **Socket.IO** pour :
  - Envoyer les scores à tous les clients connectés en temps réel (`io.sockets.emit("scores", ...)`).
  - Gérer les connexions/déconnexions des clients.
- Sert la page web (HTML/JS) qui reçoit les scores et les affiche dynamiquement.

#### Notions techniques détaillées :
- **Socket.IO** : Permet une communication bidirectionnelle en temps réel entre le serveur et le navigateur. Ici, il "pousse" les scores dès qu'ils changent, sans que le client ait besoin de demander (pas de polling).
- **DNS interne** : Le nom `db` dans l'URL de connexion est automatiquement résolu vers l'adresse IP du service PostgreSQL, que ce soit en Docker ou en Kubernetes.
- **Healthcheck** : Vérifie que la base de données est prête avant de lancer le service result, pour éviter les erreurs de connexion.

### Kubernetes

**result-deployment.yaml** (`k8s/result-deployment.yaml`) :
- 1 replica (peut être augmenté si besoin).
- Image récupérée depuis Google Artifact Registry.
- Variables d'environnement injectées via `envFrom: configMapRef: app-config` (ex : PORT, POSTGRES_HOST).
- Limite les ressources CPU/mémoire pour éviter qu'un pod ne consomme tout le cluster.
- `livenessProbe` : vérifie que le serveur répond bien sur `/` port 4000 (redémarre le pod si besoin).

**result-service.yaml** (`k8s/result-service.yaml`) :
- Type `LoadBalancer` : rend le service accessible depuis l'extérieur du cluster (pour les utilisateurs).
- Expose le port 4000.

#### Notions techniques détaillées :
- **Pod** : Unité de déploiement Kubernetes, ici chaque pod exécute un conteneur result.
- **Service LoadBalancer** : Crée une IP publique pour accéder au service result depuis Internet.
- **ConfigMap** : Centralise la configuration (variables d'environnement) pour tous les services.

**Communication** :
- result → db (ClusterIP interne) : lit les résultats via le DNS interne `db`.
- result → navigateur : envoie les scores en temps réel via Socket.IO.

### Terraform Docker (`terraform/modules/result/main.tf`)
- `docker_image` : build l'image à partir du dossier `result/`, avec un trigger sur le hash du code source (rebuild automatique si le code change).
- `docker_container` : lance le conteneur sur les réseaux `front-net` et `back-net`, expose le port 4000, définit l'env `PORT=4000`.
- `docker_registry_image` : optionnel, push l'image sur le registry Google si besoin.

### Terraform GCP (`terraform-gcp/k8s-manifests.tf`)
- Le manifest `k8s/result-deployment.yaml` est appliqué automatiquement via la ressource `kubernetes_manifest`.
- Terraform lit le YAML, injecte le namespace et les variables nécessaires, puis applique la configuration sur le cluster GKE.

#### Notions techniques détaillées :
- **Terraform** : Outil d'infrastructure as code, il automatise la création et la gestion de l'infrastructure cloud (cluster, services, réseaux, etc.).
- **kubernetes_manifest** : Permet à Terraform d'appliquer directement des fichiers YAML Kubernetes sur le cluster.

### Points clés à mentionner
- Le service result permet l'affichage en temps réel des votes grâce à Socket.IO (aucun polling côté client).
- La connexion à la base PostgreSQL est sécurisée et automatisée via le DNS interne et les ConfigMaps.
- Le déploiement est entièrement automatisé de la construction de l'image Docker jusqu'à la mise en production sur GKE grâce à Terraform.
- LoadBalancer rend le service accessible aux utilisateurs, tandis que la base de données reste protégée en interne (ClusterIP).
- Toute la configuration (ports, variables, ressources) est centralisée et versionnée dans le code (YAML, Terraform).

---

N'hésite pas à demander des précisions sur une notion technique ou une partie du code !
