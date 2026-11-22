# CampusConnect — Infrastructure socle

Socle partagé du projet **CampusConnect** (INF4057) : configuration centralisée, découverte de services, bases de données et bus de messages. Il se lance en une commande, **sans rien modifier dans ce dépôt** : chaque membre branche simplement son service dessus.

| Composant | Rôle | Port hôte par défaut | URL |
|---|---|---|---|
| `config-server` | Spring Cloud Config Server (mode *native*, lit `config-repo/`) | 8888 | http://localhost:8888 |
| `eureka-server` | Registre de services Netflix Eureka | 8761 | http://localhost:8761 |
| `kafka` | Broker Kafka (mode KRaft, sans Zookeeper) | 9092 | `localhost:9092` |
| `postgres-auth` | Base `auth_db` du service d'authentification | 5433 | `localhost:5433` |
| `postgres-student` | Base `student_db` du service étudiants | 5434 | `localhost:5434` |

Tous les ports sont liés à `127.0.0.1` : rien n'est exposé sur le réseau du campus.

## 1. Démarrage rapide

**Prérequis** : Git, Docker et Docker Compose v2 (`docker compose version`).

```bash
git clone https://github.com/tamssonna2-png/campusconnect-infra.git
cd campusconnect-infra
cp .env.example .env
docker compose up -d --build
```

Le premier build est long (téléchargement de Maven et des dépendances). Ensuite, attendez environ 1 minute puis vérifiez :

```bash
docker compose ps                                   # tout doit être "healthy"
curl -s http://localhost:8888/auth-service/dev,docker   # JSON avec les propriétés d'auth-service
```

Eureka est visible dans le navigateur : http://localhost:8761

### Commandes utiles

```bash
docker compose logs -f config-server     # suivre les logs d'un service
docker compose restart config-server     # redémarrer un composant
docker compose down                      # arrêter (données conservées)
docker compose down -v                   # arrêter ET effacer les bases et Kafka
```

## 2. Variables (`.env`)

`.env` n'est jamais commité. Seul `.env.example` l'est. Vous n'avez normalement **rien à changer**. Si un port est déjà pris sur votre machine, modifiez-le dans `.env` :

| Variable | Défaut | Usage |
|---|---|---|
| `POSTGRES_AUTH_PORT` / `POSTGRES_STUDENT_PORT` | 5433 / 5434 | Port hôte des bases |
| `KAFKA_PORT` | 9092 | Port hôte de Kafka |
| `EUREKA_SERVER_PORT` | 8761 | Port hôte d'Eureka |
| `CONFIG_SERVER_PORT` | 8888 | Port hôte du Config Server |
| `POSTGRES_*_DB`, `_USER`, `_PASSWORD` | voir `.env.example` | Identifiants de dev |

> ⚠️ **Ne changez pas les identifiants de bases** sauf nécessité. Postgres ne les applique qu'à la **première** création du volume (pour les changer ensuite : `docker compose down -v`), et `config-repo/` utilise les mêmes valeurs par défaut.
>
> Si vous changez un **port**, mettez la même variable d'environnement (ex. `POSTGRES_AUTH_PORT=5435`) dans la configuration de lancement de votre service, sinon il continuera à viser le port par défaut.

## 3. Brancher son service

### Noms imposés

Le `spring.application.name` doit être **exactement** celui-ci, sinon le Config Server ne trouve pas votre fichier de configuration.

| Service | `spring.application.name` | Port | Base | Fichiers de config |
|---|---|---|---|---|
| Auth | `auth-service` | 8001 | `auth_db` (5433) | `auth-service-dev.yml`, `auth-service-docker.yml` |
| Student | `student-service` | 8002 | `student_db` (5434) | `student-service-dev.yml`, `student-service-docker.yml` |
| Gateway | `gateway` | 8080 | — | `gateway-dev.yml` |

Les autres services (course, enrollment, payment, notification…) suivent la même convention : ajoutez `<nom>-dev.yml` (et `<nom>-docker.yml` si besoin) dans `config-repo/`.

### Dépendances Maven

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

Versions à utiliser (alignées sur le socle) : Spring Boot **3.3.5**, Spring Cloud **2023.0.3**. Le gateway utilise en plus `spring-cloud-starter-gateway` et `spring-cloud-starter-loadbalancer`.

### `application.yml` de votre service (le seul fichier à écrire)

```yaml
spring:
  application:
    name: auth-service            # voir tableau ci-dessus
  config:
    import: configserver:${CONFIG_SERVER_URL:http://localhost:8888}
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
```

Le port, la datasource, Eureka et Kafka viennent du Config Server. Ne les redéfinissez pas dans votre service.

### Deux façons de lancer son service

| | Profils | `CONFIG_SERVER_URL` | Eureka | Kafka | Postgres |
|---|---|---|---|---|---|
| **Depuis l'IDE** (Eclipse / VS Code) | `dev` (défaut) | `http://localhost:8888` (défaut) | `localhost:8761` | `localhost:9092` | `localhost:5433` / `5434` |
| **Dans un conteneur** | `dev,docker` | `http://config-server:8888` | `eureka-server:8761` | `kafka:29092` | `postgres-auth:5432` / `postgres-student:5432` |

L'ordre `dev,docker` compte : le dernier profil l'emporte, donc `docker` remplace les adresses `localhost` de `dev`.

> ⚠️ **Ne mélangez pas** les deux modes entre services qui s'appellent. Un service lancé depuis l'IDE s'enregistre dans Eureka avec l'adresse de votre machine, qu'un gateway tournant dans Docker ne peut pas joindre. Tout dans l'IDE, ou tout dans Docker.

### Lancer son service dans Docker

Dans **votre** dépôt de service, un `docker-compose.yml` qui rejoint le réseau du socle (créé par ce dépôt, il doit donc être démarré avant) :

```yaml
services:
  auth-service:
    build: .
    container_name: campus-auth-service
    ports:
      - "8001:8001"
    environment:
      SPRING_PROFILES_ACTIVE: dev,docker
      CONFIG_SERVER_URL: http://config-server:8888
    restart: on-failure
    networks:
      - campus-net

networks:
  campus-net:
    external: true
```

`restart: on-failure` est utile car un `depends_on` ne peut pas viser un service d'un autre fichier compose : si le Config Server n'est pas encore prêt, votre service redémarre jusqu'à ce qu'il le soit.

### Gateway

`gateway-dev.yml` définit déjà deux routes vers les services via Eureka :

- `/auth/**` → `lb://auth-service`
- `/students/**` → `lb://student-service`

Le chemin est transmis **tel quel** : les contrôleurs de `auth-service` doivent donc être sous `/auth/...` et ceux de `student-service` sous `/students/...`. Pour ajouter une route, éditez `config-repo/gateway-dev.yml` (voir section suivante).

## 4. Modifier la configuration partagée

Le Config Server lit directement les fichiers de `config-repo/` (monté en lecture seule). Pas besoin de commit pour qu'il les voie.

1. Éditez le fichier dans `config-repo/`.
2. Redémarrez le service concerné pour qu'il relise sa configuration.
3. Commitez et poussez : `config-repo/` fait partie de ce dépôt, tout le monde doit avoir la même version.

Annoncez toute modification de `config-repo/` à l'équipe.

Structure :

```
config-repo/
├── application.yml            # commun à tous les services (profil IDE)
├── application-docker.yml     # commun, remplace les adresses en mode conteneur
├── <service>-dev.yml          # config propre à un service (IDE)
└── <service>-docker.yml       # config propre à un service (conteneur)
```

## 5. Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `bind: address already in use` | Port déjà utilisé sur votre machine | Changez le port dans `.env`, puis `docker compose up -d` |
| `curl localhost:8888/...` ne répond pas | Config Server encore en démarrage | Attendez 30 à 60 s, `docker compose ps` doit afficher `healthy` |
| Un service démarre sans sa config (port 8080 par défaut) | `spring.application.name` incorrect ou Config Server injoignable | Vérifiez le nom exact du tableau §3 et testez `curl localhost:8888/<nom>/dev` |
| `password authentication failed` | Mot de passe modifié après la première initialisation du volume | `docker compose down -v` puis relancer |
| Service absent d'Eureka | Dépendance `eureka-client` manquante, ou profil `dev,docker` oublié dans un conteneur | Vérifiez la dépendance et le profil actif |
| Kafka : `Connection to node -1 could not be established` | Mauvaise adresse selon le mode | IDE : `localhost:9092`, conteneur : `kafka:29092` (venant du profil `docker`) |
| Le build Docker échoue | Pas de connexion pour télécharger Maven | Relancez avec `docker compose build --no-cache` une fois en ligne |

## 6. Décisions et limites connues

- **Java 21** est utilisé par les deux serveurs du socle (alors que le sujet mentionne Java 17). À confirmer en équipe. Le choix s'applique à tous les services.
- Les identifiants de bases sont des valeurs de **développement** uniquement. Ne jamais les réutiliser ailleurs.
- Les services applicatifs (auth, student, gateway…) ne sont **pas** dans ce dépôt : chacun vit dans son propre dépôt et rejoint le réseau `campus-net`.
- Les images Docker sont construites en local (build Maven dans l'image, aucun `.jar` n'est commité).

## 7. Contenu du dépôt

```
campusconnect-infra/
├── docker-compose.yml     # socle : config, eureka, kafka, 2 × postgres
├── .env.example           # variables à copier vers .env
├── config-repo/           # configuration centralisée (servie par le Config Server)
├── config-server/         # Spring Cloud Config Server (Dockerfile + sources)
└── eureka-server/         # Netflix Eureka Server (Dockerfile + sources)
```

Contact : membre 1 (infrastructure).
