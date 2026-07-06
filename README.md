# mspr_bloc4
Git for mspr block4 exam.

# FutureKawa — Documentation technique

Ce fichier README documente l'architecture Docker du projet "FutureKawa", la pipeline Jenkins, et la gestion des données secretes.
/!\ À lire avant de toucher au `docker-compose.yml` ou au `Jenkinsfile`. /!\

## Détails des services :

Le fichier `docker-compose.yml` fait tourner 6 conteneurs, répartis en trois blocs :
 - le backend "siège" et son frontend,
- la stack IoT séparée avec sa propre base et son broker MQTT.

**Backend/API + frontend**
- `db` (mysql:8.0) — base principale, contient les lots, les exploitations, les utilisateurs. Conteneur : `futurekawa-db`.
- `web` (build `./MSPR1/docker/Dockerfile`) — l'API REST du siège. Conteneur : `futurekawapp`, exposé sur le port 5000.
- `front` (build `./futureKawaFront/Dockerfile`) — l'interface web. conteneur `futurekawa-front`, exposé sur le port 9090 (mappé vers le 4173 interne).

**Stack IoT**
- `db_iot` (mysql:8) — base dédiée aux mesures capteurs. conteneur `iot_db`, port 3308. A un healthcheck mysqladmin.
- `mosquitto` (eclipse-mosquitto) — broker MQTT qui reçoit les données envoyées par les ESP32. Port 1883.
- `flask` (build `./futurekawa/Dockerfile`) — backend IoT, traite les mesures, gère les alertes et le watchdog. Conteneur : `futurekawa-iot_service`, port 5001.

L'ordre de démarrage compte : `flask` attend que `db_iot` soit "healthy" et que `mosquitto` ait démarré. `web` attend `db`, `front` attend `web`.

Trois volumes persistants : `db_data`, `iot_db_data`, `mosquitto_config`. Tout communique sur le même réseau bridge `futurekawa`.

## Le pipeline Jenkins :

Déclenché par polling SCM toutes les 12h (`pollSCM('H H/12 * * *')`) (Pour minimiser le nombre de requètes journalières, (écoconception)). Voici ce qui se passe étape par étape :

**1. Clonage des dépôts**
Les quatre repos sont récupérés dans des sous-dossiers séparés : `mspr_bloc4` (celui-ci, branche master, avec le credential `git-tok`), `futureKawaFront` (branche main), `futurekawa` (backend IoT), et `MSPR1` sur sa branche de test.

**2. Build initial**
On copie le `.env` depuis le credential Jenkins vers le workspace, puis on build seulement `db` et `web` — pas besoin du reste à ce stade, on va juste lancer les tests dessus.

**3. Tests unitaires**
```bash
docker compose run --rm web sh -c "pytest -v -s"
```
Un conteneur jetable lance pytest. Si ça échoue, la pipeline s'arrête net (`error` explicite avec le code retour).

**3.2 Lint du frontend**
Build d'une image dédiée juste pour le lint, puis :
```bash
docker run --rm futurekawa-front-lint npm run lint
```

**4. Nettoyage**
On supprime le dossier `MSPR1` (celui de test) et on fait tomber tous les conteneurs avec les volumes (`docker compose down -v --remove-orphans`). L'idée ic est de repartir sur une base propre avant de déployer le vrai code.

**5. Re-clone de MSPR1 en main**
Cette fois on récupère la vraie branche de prod du backend, et on recopie le `.env` (en 'credential').

**6. Build et déploiement final**
```bash
docker compose build
docker compose up -d
sleep 10
docker compose ps
```
Le `sleep 10` laisse le temps aux conteneurs de s'initialiser avant qu'on vérifie leur état. (les conteneurs de Bases de Données peuvent mettre plus de temps pour se lancer).

**Post-build**, dans tous les cas (succès ou échec), le `.env` est supprimé du workspace :
```bash
rm -f .env
```
C'est volontaire, pour ne pas laisser traîner de secrets en clair sur l'agent Jenkins.

## Les fichier "secrets" :

Que ce soient les variables d'environnement des repos, ou les tokens, rien ne doit apparaître en clair dans le repo. Tout passe par le magasin de credentials de Jenkins ('administrer Jenkins' --> Credentials --> boutton : `+ add Credentials`).

Il y a deux credentials à configurer :

- **`futurekawa.env`** : type "Secret file". C'est le `.env` complet du projet, uploadé une fois dans Jenkins. Il est référencé dans le Jenkinsfile via `credentials('futurekawa.env')` et copié dans le workspace au moment du build (`cp $SECRET_ENV .env`), puis effacé en fin de pipeline.
- **`git-tok`** : type "Secret text" ou "Username/password" selon la méthode choisie. C'est le token GitHub utilisé pour cloner le repo `mspr_bloc4`, qui est visiblement privé (c'est le seul des quatre clones qui référence explicitement un `credentialsId`).

Pour les ajouter : Manage Jenkins → Credentials → cliquer sur le domaine global → Add Credentials, puis choisir le bon type et renseigner l'ID exact (`futurekawa.env` ou `git-tok`) — le Jenkinsfile s'appuie sur ces IDs précis, donc une faute de frappe casse la pipeline silencieusement au moment du checkout ou du build.

À vérifier : les trois autres clones (front, futurekawa, MSPR1) n'ont pas de credential associé dans le Jenkinsfile actuel.
Avant de passer l'application en production, il faudra créer des tokens pour chaque repo, puis les référencer dans Jenkins ('administrer Jenkins' --> Credentials --> boutton : `+ add Credentials`), et passer les satuts des repos EN PRIVÉ !!

Une fois les repos passés en "privé" il faudra modifier le Jenkinsfile pour cahnger les triggers en pollCSM en githubPush() (Webhooks) --> /!\ bien regarder la documentation !

## Lancer le tout en local, sans Jenkins :

```bash
# le .env doit être présent à la racine avant de lancer quoi que ce soit
docker compose build
docker compose up -d
docker compose ps
```

Pour suivre les logs d'un service en particulier :
```bash
docker compose logs web
docker compose logs flask
```

Pour tout arrêter et repartir de zéro (supprime aussi les volumes, donc les données, NE PAS LANCER UN FOIS EN PROD) :
```bash
docker compose down -v
```

## Accès une fois que tout les conteneurs tournent :

- Frontend : http://localhost:9090
- API backend central : http://localhost:5000
- Backend IoT : http://localhost:5001
- Broker MQTT : port 1883
- MySQL IoT : port 3308 (le port de la base principale est mappé dynamiquement, voir `docker compose ps`)