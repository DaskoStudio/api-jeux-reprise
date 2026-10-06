# API Catalogue de jeux

API REST d'un catalogue de jeux vidéo (jeux, éditeurs, notes, statistiques), avec
authentification par jeton. Elle sert de back-end à un front web ; sa
documentation interactive est générée automatiquement à `/docs`.

Ce dépôt est maintenu par un groupe d'étudiants de Bachelor 2 dans le cadre du
module *Travail collaboratif & documentation technique*.

## Prérequis

- **Python 3.12** ou plus récent
- **pip** (fourni avec Python)
- Aucune base externe n'est nécessaire pour le démarrage rapide : il utilise
  **SQLite**, inclus dans Python. PostgreSQL est possible en production (voir
  [Configuration](#configuration)).

## Démarrage rapide

Dans un dossier vide, cinq étapes suffisent à lancer l'API.

```bash
# 1. Récupérer le code
git clone https://github.com/DaskoStudio/api-jeux-reprise.git
cd api-jeux-reprise

# 2. Créer et activer un environnement virtuel
python -m venv .venv
source .venv/bin/activate        # Windows : .venv\Scripts\activate

# 3. Installer les dépendances
pip install -r requirements.txt

# 4. Créer le fichier de configuration
cp .env.example .env             # Windows : copy .env.example .env
```

Ouvrez `.env` et remplacez les deux lignes obligatoires par :

```ini
DATABASE_URL=sqlite:///./jeux.db
CLE_SECRETE=collez-ici-votre-cle
```

Générez la clé avec :

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Puis peuplez la base et lancez le serveur :

```bash
# 5. Catalogue de démonstration + compte administrateur
python scripts/peupler.py

# 6. Lancer l'API en mode développement
fastapi dev app/main.py
```

**Résultat attendu.** Le terminal affiche `Démarrage — environnement :
developpement` puis `Application startup complete`. Ouvrez
<http://127.0.0.1:8000/docs> : la liste des routes s'affiche. Ouvrez
<http://127.0.0.1:8000/> : la réponse est `{"message": "API opérationnelle", ...}`.

Le script `peupler.py` affiche `Administrateur créé : admin@example.com` et
`Jeux créés : 8`. Le compte de démonstration est **`admin@example.com` /
`motdepasse123`** — à utiliser avec le bouton **Authorize** de `/docs`.

## Configuration

Les variables sont lues dans `.env` au démarrage (`app/config.py`). Une variable
obligatoire absente empêche le lancement, avec un message explicite.

| Variable | Rôle | Obligatoire | Défaut |
|---|---|---|---|
| `DATABASE_URL` | Chaîne de connexion à la base. SQLite (`sqlite:///./jeux.db`) ou PostgreSQL (`postgresql+psycopg://utilisateur:motdepasse@hote:5432/base`) | **oui** | — |
| `CLE_SECRETE` | Clé de signature des jetons JWT. Qui la détient peut forger un jeton d'administrateur : gardez-la secrète, différente en production | **oui** | — |
| `ALGORITHME_JETON` | Algorithme de signature du jeton | non | `HS256` |
| `DUREE_JETON_MINUTES` | Durée de validité d'un jeton | non | `30` |
| `ORIGINES_AUTORISEES` | Origines CORS autorisées (liste séparée par des virgules) | non | `http://localhost:5173` |
| `ENVIRONNEMENT` | `developpement` ou `production` | non | `developpement` |
| `NIVEAU_JOURNAL` | Niveau de journalisation (`INFO`, `WARNING`…) | non | `INFO` |
| `ECHO_SQL` | Affiche chaque requête SQL dans le terminal (utile pour repérer un N+1) | non | `false` |
| `MAX_TENTATIVES_CONNEXION` | Nombre d'échecs de connexion avant blocage temporaire | non | `5` |
| `FENETRE_TENTATIVES_MINUTES` | Fenêtre de comptage des tentatives | non | `15` |

> `.env` n'est jamais versionné (il contient un secret). Seul `.env.example`,
> sans valeur réelle, l'est.

## Utilisation

La documentation complète et interactive des routes est à
<http://127.0.0.1:8000/docs>. Toutes les routes métier sont préfixées par
`/api/v1`. Quelques exemples :

```bash
# Lister le catalogue (public, paginé)
curl http://127.0.0.1:8000/api/v1/jeux

# Statistiques du catalogue (public)
curl http://127.0.0.1:8000/api/v1/jeux/statistiques

# Se connecter et récupérer un jeton (formulaire OAuth2 : username = l'email)
curl -X POST http://127.0.0.1:8000/api/v1/connexion \
  -d "username=admin@example.com&password=motdepasse123"

# Créer un jeu (authentifié : remplacez <JETON> par le token obtenu ci-dessus)
curl -X POST http://127.0.0.1:8000/api/v1/jeux \
  -H "Authorization: Bearer <JETON>" \
  -H "Content-Type: application/json" \
  -d '{"titre": "Hollow Knight", "genre": "Metroidvania", "note": 10, "annee": 2017}'
```

Les lectures (`GET`) sont publiques ; la création, la modification et la
suppression exigent un jeton.

## Tests

La suite de tests tourne sur une base SQLite en mémoire, sans configuration :

```bash
pytest
```

Le linter vérifie le style et les erreurs courantes :

```bash
ruff check .
```

Les deux sont exécutés automatiquement par l'intégration continue sur chaque
pull request.

## Architecture

L'application est découpée en couches. Chaque couche ne dépend que de la
suivante : une route ne lit jamais la base directement, elle passe par un
service, qui passe par un dépôt.

```mermaid
flowchart TD
    Client[Client HTTP / front] -->|/api/v1| Routeurs
    Routeurs -->|règles métier| Services
    Services -->|lecture / écriture| Depots[Dépôts]
    Depots --> Tables[Tables ORM]
    Tables --> BDD[(Base de données)]
    Routeurs -. validation / réponses .-> Modeles[Modèles Pydantic]
    Config[config.py] --> BaseDonnees[base_donnees.py]
    BaseDonnees --> BDD
```

Dossiers de `app/` :

| Dossier / fichier | Rôle |
|---|---|
| `routeurs/` | Les routes HTTP, un fichier par domaine. Elles reçoivent, délèguent, répondent — aucune règle métier |
| `services/` | Les règles métier et les décisions (403, 404, 409), ainsi que la limitation des tentatives de connexion |
| `depots/` | L'accès aux données : lit et écrit, aucune règle |
| `tables/` | Les modèles SQLAlchemy — le schéma de la base |
| `modeles/` | Les schémas Pydantic : validation des entrées et forme des réponses (ce qui sort, ce qui reste caché) |
| `config.py` | Configuration validée au démarrage (`pydantic-settings`) |
| `base_donnees.py` | Moteur, session par requête, création des tables |
| `securite.py` | Hachage des mots de passe (bcrypt) et jetons JWT |
| `dependances.py` | Dépendances FastAPI : session, utilisateur courant, pagination |
| `exceptions.py` | `ErreurMetier` et le format d'erreur unique de l'API |
| `journalisation.py` | Configuration des journaux |
| `main.py` | Assemblage : middlewares, gestionnaires d'erreurs, routeurs |

## Contribuer

L'équipe travaille par petites étapes relues :

1. **Une issue, une branche.** On nomme la branche `type/numero-description`
   (ex. `fix/1-statistiques-catalogue-vide`), créée depuis un `main` à jour.
2. **Des commits lisibles**, au format *Conventional Commits* (`fix(jeux): …`),
   avec `Refs #numero` dans le corps.
3. **Une pull request relue.** On ne pousse jamais directement sur `main` : tout
   passe par une PR, relue par un autre membre, avec `Closes #numero`.
4. **La fusion en squash** une fois la PR approuvée et l'intégration continue
   verte, puis on supprime la branche.
