# TrainShop — Intégration Continue avec GitHub Actions et Docker

## À propos

TrainShop est un projet d'apprentissage DevOps complet. Il démontre comment mettre en place une chaîne d'intégration continue (CI) robuste avec GitHub Actions qui valide automatiquement :

- La qualité du code (linting)
- L'exécution des tests automatisés
- La construction d'une image Docker
- Le démarrage d'un container
- La santé de l'application via un endpoint `/health`
- Le nettoyage automatique des ressources

## Stack

- Frontend HTML/CSS/JS
- API Node.js / Express
- PostgreSQL
- Docker & Docker Compose
- GitHub Actions (CI/CD)
- ESLint (code quality)
- Jest (testing)

## Architecture

```
trainshop/
├── api/                    # API Node.js/Express
│   ├── src/
│   │   ├── app.js         # Application Express
│   │   ├── server.js      # Point d'entrée
│   │   └── db.js          # Configuration PostgreSQL
│   ├── tests/             # Tests automatisés (Jest)
│   │   ├── health.test.js
│   │   └── products.test.js
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   └── .eslintrc.json
├── frontend/              # Interface utilisateur
├── .github/workflows/
│   ├── ci.yml             # Workflow principal
│   └── docker-api.yml     # Workflow Docker Hub
├── docker-compose.yml
└── README.md
```

## Installation

### Prérequis

- Node.js 20+
- Docker & Docker Compose
- Git

### Étapes

```bash
# Cloner le repository
git clone <url-du-repo>
cd trainshop

# Installer les dépendances de l'API
cd api
npm install
```

## Utilisation locale

### Lancer le projet complet

```bash
# Depuis la racine du projet
docker compose up -d --build
```

### Développement API seule

```bash
cd api

# Mode développement (rechargement auto)
npm run dev

# Mode production
npm start
```

L'API démarre sur le port **3000** (configurable via `API_PORT`).

### Lancer les tests

```bash
cd api
npm test
```

Tests inclus :
- `/health` endpoint (status check)
- `/products` GET (list products)
- `/products` POST avec produit valide
- `/products` POST avec produit invalide (validation)

### Vérifier la qualité du code

```bash
cd api
npm run lint
```

ESLint vérifie :
- Indentation (2 espaces)
- Guillemets (simples)
- Points-virgules
- Variables inutilisées
- Code console en production

## API – Routes

### GET `/health`

État de l'API et de la base de données.

```bash
curl http://localhost:3000/health
```

**Réponse** (200 OK) :
```json
{
  "status": "ok",
  "service": "trainshop-api",
  "database": "connected"
}
```

### GET `/products`

Liste tous les produits.

```bash
curl http://localhost:3000/products
```

### GET `/products/:id`

Récupère un produit par ID.

### POST `/products`

Crée un nouveau produit.

```bash
curl -X POST http://localhost:3000/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Guide Docker",
    "description": "Support pédagogique",
    "price_cents": 1900,
    "stock": 20
  }'
```

**Champs obligatoires** : `name`, `description`, `price_cents`

## Docker

### Construire l'image

```bash
cd api
docker build -t trainshop-api .
```

### Lancer un container

```bash
docker run -d \
  --name trainshop-api \
  -p 3000:3000 \
  -e API_PORT=3000 \
  trainshop-api
```

### Vérifier la santé

```bash
curl http://localhost:3000/health
```

### Nettoyer

```bash
docker stop trainshop-api && docker rm trainshop-api
```

## GitHub Actions – CI/CD

### Workflow `ci.yml`

S'exécute automatiquement à chaque **push** ou **pull request** sur `main`.

**Étapes** :
1. Récupère le code
2. Installe Node.js 20
3. Installe les dépendances npm
4. Vérifie la qualité du code (ESLint)
5. Lance les tests (Jest)
6. Construit l'image Docker API
7. Démarre le container API
8. Attend que l'API démarre (30 sec timeout)
9. Teste l'endpoint `/health`
10. Affiche les logs du container
11. Nettoie le container (même en cas d'erreur)

### Workflow `docker-api.yml`

Pousse l'image vers Docker Hub après chaque push sur `main`.

**Secrets requis** (ajouter sur GitHub) :
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

## Interprétation des résultats

### ✅ **CI VERTE** (All checks passed)

Tous les tests sont au vert :
- Code respecte ESLint
- Tous les tests Jest passent
- Image Docker se construit
- Container démarre et répond `/health`

**Vous pouvez fusionner la PR sans risque.**

### ❌ **CI ROUGE** (Failed)

Cliquez sur **Details** pour voir quelle étape a échoué :

**Lint failed** → Corrigez le style du code
**Tests failed** → Debuggez la logique métier
**Docker failed** → Vérifiez le Dockerfile
**Health check failed** → Vérifiez les logs du container

## Vérifications avant push

```bash
cd api

# Test 1: Lint
npm run lint

# Test 2: Tests
npm test

# Test 3: Docker build & run
docker build -t trainshop-api .
docker run -d --name test -p 3000:3000 trainshop-api

# Test 4: Health check
curl http://localhost:3000/health

# Test 5: Cleanup
docker stop test && docker rm test
```

## FAQ

### "Pourquoi tester dans un container ?"
Un code peut passer les tests localement mais échouer en Docker (dépendance manquante, port, variables d'env). La CI garantit que le code **fonctionne réellement en production**.

### "Pourquoi vérifier `/health` ?"
C'est un contrôle de santé qui garantit :
- L'API a démarré
- La base de données est accessible
- Le service est prêt

### "Pourquoi nettoyer le container ?"
Chaque workflow GitHub consomme des ressources. Les containers doivent être supprimés même en cas d'erreur.

### "Comment déboguer une CI qui échoue ?"
1. Consultez les **logs GitHub Actions**
2. Reprochez localement : `docker build . && docker run ...`
3. Testez `npm test` et `npm run lint` en local

## Ressources

- [Express.js](https://expressjs.com)
- [Jest](https://jestjs.io)
- [Docker](https://docker.com)
- [GitHub Actions](https://github.com/features/actions)
- [ESLint](https://eslint.org)

## Livrables TP

- ✓ API avec routes `/health`, `/products`
- ✓ Tests automatisés (Jest)
- ✓ Dockerfile & `.dockerignore`
- ✓ Workflow GitHub Actions
- ✓ Configuration ESLint
- ✓ README complet
- ✓ CI passant au vert ✅

---

**Validation finale** : La CI de ce repository doit être **verte ✅**. Sinon, consultez les logs GitHub Actions.

Vérifier :

```bash
docker compose ps
```

Tester l'API :

```bash
curl http://localhost:3000/health
curl http://localhost:3000/products
```

Ouvrir le frontend :

```text
http://localhost:8081
```

## Arrêter

```bash
docker compose down
```

Supprimer aussi la base de données :

```bash
docker compose down -v
```

## Objectif du TP CI/CD

Les apprenants devront créer le dossier :

```text
.github/workflows/
```

Puis ajouter progressivement :

1. un workflow CI qui lance les tests API ;
2. un workflow qui vérifie les builds Docker ;
3. éventuellement un workflow de publication Docker, en bonus.
