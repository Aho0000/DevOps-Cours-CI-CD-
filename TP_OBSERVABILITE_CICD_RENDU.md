# TP — Observabilité CI/CD avec GitHub Actions et Docker

**Étudiant** : [Ton nom]  
**Date** : [Date]  
**Durée estimée** : 2h-3h  
**Groupe** : [Ton groupe]

---

## 1. Analyse de la CI existante

### Vérifications déjà réalisées par GitHub Actions

**Étapes concernant le code :**
- [ ] Checkout du code
- [ ] Linting (si configuré)
- [ ] Installation des dépendances npm
- [ ] Lancement des tests unitaires/intégration

**Étapes concernant Docker :**
- [ ] Construction de l'image Docker de l'API
- [ ] Construction de l'image Docker du frontend (optionnel)

**Étapes concernant le démarrage :**
- [ ] Lancement du container Docker
- [ ] Attente du démarrage de l'application

**Étapes concernant la vérification :**
- [ ] Test du endpoint `/health` via `curl`
- [ ] Vérification que la réponse est correcte

### Ordre des étapes dans le workflow

```
1. Récupérer le code (actions/checkout@v4)
   ↓
2. Installer Node.js (actions/setup-node@v4)
   ↓
3. Installer les dépendances (npm install)
   ↓
4. Lancer les tests (npm test)
   ↓
5. Construire image Docker API (docker build)
   ↓
6. Construire image Docker frontend (docker build)
   ↓
7. Lancer le container (docker run)
   ↓
8. Attendre le démarrage (~5 secondes)
   ↓
9. Tester /health (curl)
   ↓
10. Afficher les logs en cas d'échec
```

---

## 2. Limites de la CI identifiées

### Ce que la CI NE garantit PAS

| Aspect | Garanti par CI ? | Pourquoi ? |
|--------|------------------|-----------|
| **Performances** | ❌ Non | La CI teste uniquement une fois, sans charge. |
| **Volumes réels** | ❌ Non | La CI ne simule pas 10 000 utilisateurs. |
| **Routes métier critiques** | ⚠️ Partiellement | Seul `/health` est testé, pas `/orders` ou `/checkout`. |
| **Stabilité long terme** | ❌ Non | La CI teste 30 secondes, pas 8 heures. |
| **Impact métier réel** | ❌ Non | La CI ne mesure pas les conversions ou les achats. |
| **Comportement en production** | ⚠️ Partiellement | L'environnement de test peut différer de la prod. |
| **Dépendances externes** | ⚠️ Partiellement | La BD est mockée dans les tests. |
| **Erreurs utilisateur** | ❌ Non | La CI ne teste pas les bugs fonctionnels métier. |

### Risques après déploiement

- ✅ CI verte mais application lente en production
- ✅ CI verte mais base de données inaccessible en production
- ✅ CI verte mais route critique en erreur 500
- ✅ CI verte mais taux de conversion en baisse

---

## 3. Conception des healthchecks

### 3.1 Endpoint `/health`

**Rôle** : Vérifier que l'application est vivante et répond.

**Réponse JSON proposée** :
```json
{
  "status": "ok",
  "service": "trainshop-api",
  "version": "1.3.0",
  "environment": "production",
  "timestamp": "2026-05-11T10:30:45Z",
  "uptime_seconds": 3600
}
```

**Caractéristiques** :
- ✅ Rapide (< 100ms)
- ✅ Sans donnée sensible
- ✅ Contient la version déployée
- ✅ Contient l'environnement (local/staging/prod)
- ✅ Contient un timestamp
- ✅ Contient l'uptime pour détecter les redémarrages

**Codes HTTP attendus** :
- `200 OK` : Application est vivante
- `503 Service Unavailable` : Processus applicatif ne répond plus

---

### 3.2 Endpoint `/ready`

**Rôle** : Vérifier que l'application est prête à recevoir du trafic métier.

**Réponse JSON proposée** :
```json
{
  "status": "ready",
  "service": "trainshop-api",
  "version": "1.3.0",
  "timestamp": "2026-05-11T10:30:45Z",
  "dependencies": {
    "database": {
      "status": "ok",
      "response_time_ms": 5
    },
    "cache": {
      "status": "ok",
      "response_time_ms": 2
    },
    "payment_service": {
      "status": "ok",
      "response_time_ms": 150
    }
  },
  "required_env_vars": {
    "DATABASE_URL": true,
    "PAYMENT_API_KEY": true,
    "CACHE_URL": true
  }
}
```

**Caractéristiques** :
- ✅ Teste la base de données
- ✅ Teste le cache (non-critique)
- ✅ Teste les services externes critiques
- ✅ Vérifie les variables d'environnement obligatoires
- ✅ Inclut les temps de réponse pour détecter la dégradation
- ✅ Aucun secret exposé

**Codes HTTP attendus** :
- `200 OK` : Application est prête
- `503 Service Unavailable` : Une dépendance critique est manquante

---

### 3.3 Comportements attendus en cas d'échec

| Scénario | `/health` | `/ready` | Action |
|----------|-----------|---------|--------|
| Processus ne répond plus | ❌ Timeout/KO | ❌ Non appelé | Redémarrer container |
| BD inaccessible | ✅ OK | ❌ KO | Ne pas servir du trafic |
| Cache indisponible | ✅ OK | ✅ OK* | Servir avec cache dégradé |
| Service paiement timeout | ✅ OK | ❌ KO | Ne pas accepter les commandes |
| Variable env manquante | ✅ OK | ❌ KO | Bloquer le déploiement |
| Version incorrecte | ✅ Mauvaise version | ✅ Mauvaise version | Vérifier le déploiement |
| Temps réponse élevé | ✅ OK | ✅ OK avec latence | Augmenter les ressources |

*Cache est non-critique, mais l'info est présente pour le monitoring.

---

### 3.4 Intégration dans GitHub Actions

**Ordre logique des contrôles** :

```yaml
steps:
  - name: 1. Construire image Docker
    run: docker build -t trainshop-api:${{ github.sha }} ./api
  
  - name: 2. Lancer le container
    run: docker run -d --name test-api -p 3000:3000 trainshop-api:${{ github.sha }}
  
  - name: 3. Attendre le démarrage (5 secondes)
    run: sleep 5
  
  - name: 4. Tester /health
    run: |
      curl -f http://localhost:3000/health || exit 1
  
  - name: 5. Tester /ready
    run: |
      curl -f http://localhost:3000/ready || exit 1
  
  - name: 6. Tester une route métier (/products)
    run: |
      curl -f http://localhost:3000/products || exit 1
  
  - name: 7. Afficher les logs en cas d'échec
    if: failure()
    run: |
      echo "=== Containers actifs ==="
      docker ps -a
      echo "=== Logs du container ==="
      docker logs test-api
      echo "=== État du container ==="
      docker inspect test-api
  
  - name: 8. Nettoyer
    if: always()
    run: docker rm -f test-api
```

---

### 3.5 Logs de diagnostic en cas d'échec

**À afficher automatiquement si `/health` ou `/ready` échoue** :

```bash
# Dans une étape "on: failure"
echo "=== 1. Containers actifs ==="
docker ps -a

echo "=== 2. État du container ==="
docker inspect test-api | grep -E "State|ExitCode|Error"

echo "=== 3. Derniers logs du container ==="
docker logs --tail 50 test-api

echo "=== 4. Tentative de /health ==="
curl -v http://localhost:3000/health 2>&1 || echo "ÉCHEC"

echo "=== 5. Tentative de /ready ==="
curl -v http://localhost:3000/ready 2>&1 || echo "ÉCHEC"

echo "=== 6. Infos de déploiement ==="
echo "Branche: ${{ github.ref }}"
echo "Commit: ${{ github.sha }}"
echo "Image: trainshop-api:${{ github.sha }}"

echo "=== 7. Variables non-sensibles ==="
echo "NODE_ENV=production"
echo "API_PORT=3000"
```

---

## 4. Logs à produire dans l'application

### Logs obligatoires

| Type | Exemple | Niveau | Quand ? |
|------|---------|--------|--------|
| **Démarrage** | `[INFO] TrainShop API v1.3.0 started on port 3000 (production)` | INFO | Au lancement |
| **Requête** | `[INFO] GET /products 200 45ms` | INFO | Chaque requête |
| **Erreur 4xx** | `[WARN] POST /orders 400 Bad Request: missing email` | WARN | Client error |
| **Erreur 5xx** | `[ERROR] POST /checkout 500 Internal Server Error: db timeout` | ERROR | Server error |
| **BD** | `[INFO] Connected to PostgreSQL` ou `[ERROR] DB connection failed: timeout` | INFO/ERROR | Connection |
| **Métier** | `[INFO] Order created: order_id=123, user_id=456, total=99.99€` | INFO | Action métier |
| **Dépendance** | `[WARN] Payment service slow: response time 2500ms (threshold: 1000ms)` | WARN | Service dégradé |

### Format recommandé

```
[TIMESTAMP] [LEVEL] [SERVICE] [REQUEST_ID] [MESSAGE]
```

Exemple :
```
2026-05-11T10:30:45.123Z [INFO] trainshop-api [req-abc123] GET /products 200 45ms
2026-05-11T10:30:46.456Z [ERROR] trainshop-api [req-def456] POST /checkout 500 Internal Server Error: database timeout
```

### Commandes Docker pour lire les logs

```bash
docker ps                                    # Voir les containers
docker logs nom_du_container                 # Voir tous les logs
docker logs -f nom_du_container              # Suivre les logs en temps réel
docker logs --tail 100 nom_du_container      # Voir les 100 derniers logs
docker logs --since 10m nom_du_container     # Voir les logs des 10 dernières minutes
docker inspect nom_du_container              # Inspecter l'état
docker stats nom_du_container                # Voir CPU/RAM
```

---

## 5. Métriques à suivre

### Métriques techniques

| Métrique | Seuil d'alerte | Outil | Fréquence |
|----------|---|---|---|
| `/health` disponible | < 95% uptime | GitHub Actions + monitoring | Chaque 1min |
| Requêtes par minute | Baseline + 50% | Application logs | Chaque 1min |
| Erreurs 4xx | > 10% du total | Application logs | Chaque 5min |
| Erreurs 5xx | > 1% du total | Application logs | Chaque 5min |
| Temps réponse moyen | > 500ms | Application logs | Chaque 5min |
| Temps réponse p95 | > 1000ms | Application logs | Chaque 5min |
| CPU container | > 80% | Docker stats | Chaque 1min |
| RAM container | > 80% | Docker stats | Chaque 1min |
| Redémarrages container | > 0 en 1h | Docker inspect | Chaque 5min |

### Métriques métier

| Métrique | Seuil d'alerte | Pourquoi | Fréquence |
|----------|---|---|---|
| Produits consultés | Chute > 20% | Perte d'usage | Chaque 15min |
| Commandes créées | Chute > 30% | Impact critique | Chaque 15min |
| Checkouts réussis | Chute > 40% | Perte de revenu | Chaque 15min |
| Erreurs checkout | Hausse > 20% | Problème paiement | Chaque 15min |
| Taux conversion | Chute > 15% | Problème UX/métier | Chaque 15min |

---

## 6. Améliorations proposées du workflow CI/CD

### Amélioration 1 : Afficher le tag Docker dans les logs CI

**Implémentation** :
```yaml
- name: Afficher le tag Docker
  run: echo "Image déployée: trainshop-api:${{ github.sha }}"
```

**Intérêt** : Savoir exactement quelle image a été construite et testée.

---

### Amélioration 2 : Tester plusieurs endpoints métier

**Implémentation** :
```yaml
- name: Tester /products
  run: curl -f http://localhost:3000/products || exit 1

- name: Tester POST /orders
  run: |
    curl -f -X POST http://localhost:3000/orders \
      -H "Content-Type: application/json" \
      -d '{"user_id": 1, "items": []}' || exit 1
```

**Intérêt** : Vérifier que les routes critiques ne sont pas cassées avant le déploiement.

---

### Amélioration 3 : Ajouter un contrôle de régression

**Implémentation** :
```yaml
- name: Comparer les performances
  run: |
    # Mesurer le temps de réponse
    time curl http://localhost:3000/products
    # Vérifier qu'il ne dépasse pas 500ms
```

**Intérêt** : Éviter que une version lente soit déployée.

---

## 7. Analyse d'incident

### Scénario : CI verte, mais commandes en baisse après déploiement

**Signaux observés** :
- ✅ CI GitHub Actions : verte
- ✅ Docker : container tourne, pas de redémarrage
- ✅ `/health` : répond OK
- ❌ Logs : erreurs répétées sur `POST /orders`
- ❌ Métriques technique : taux erreurs 5xx en hausse
- ❌ Métriques métier : commandes créées en baisse

---

### 7.1 Pourquoi la CI n'a pas détecté le problème ?

**Raisons** :
1. ❌ La CI ne teste que `/health`, pas `/orders`
2. ❌ La CI ne teste pas avec une BD réelle (utilise mocks)
3. ❌ La CI ne teste pas les volumes réels
4. ❌ La CI ne teste que 30 secondes, pas 24h
5. ❌ La CI ne vérifie pas les métriques métier

---

### 7.2 Pourquoi `/health` répond OK malgré l'incident ?

**Raisons** :
1. `/health` ne teste que le processus applicatif (vivant ✅)
2. `/health` ne teste pas la route `/orders` en particulier
3. Le problème est une erreur métier, pas un crash applicatif
4. `/ready` aurait pu détecter une dépendance manquante

---

### 7.3 Logs à consulter en priorité

```bash
docker logs --tail 200 trainshop_api | grep -i "error\|orders\|5xx"

# Ou chercher les patterns spécifiques :
# POST /orders
# 500 Internal Server Error
# database timeout
# payment service
```

---

### 7.4 Métriques à consulter en priorité

1. **Taux erreurs 5xx sur `/orders`** → Confirme le problème
2. **Nombre de commandes créées** → Mesure l'impact business
3. **Temps réponse `/orders`** → Identifie une lenteur
4. **Disponibilité de la BD** → Cherche la cause
5. **Appels au service paiement** → Cherche l'external

---

### 7.5 Hypothèse de cause

**Hypothèse 1 : Base de données inaccessible ou lente**
- Symptôme : POST /orders en erreur 500
- Preuve : logs `db timeout`
- Solution : vérifier connexion BD production

**Hypothèse 2 : Service de paiement indisponible**
- Symptôme : POST /checkout en erreur, mais /health OK
- Preuve : logs `payment service timeout`
- Solution : basculer sur mode offline, envoyer alerte

**Hypothèse 3 : Variable d'environnement manquante**
- Symptôme : application démarre mais route cassée
- Preuve : logs `missing env var`
- Solution : vérifier secrets en production

---

### 7.6 Décision : Rollback, Hotfix ou Surveillance ?

**Analyse** :
- ✅ Impact métier : commandes en baisse → **Critique**
- ✅ Cause identifiée : BD timeout → **Externe**
- ✅ `/health` OK : problème isolé à une route

**Décision** :
1. **Court terme** : Rollback à la version précédente (v1.2.5)
2. **Moyen terme** : Hotfix pour ajouter retry sur BD
3. **Long terme** : Ajouter test `/orders` en CI

**Raison** : La perte de revenu justifie un rollback rapide.

---

### 7.7 Améliorations pour éviter ce cas

**Pour la CI** :
- ✅ Ajouter un test sur `/orders` en CI
- ✅ Ajouter un test sur `/ready` en CI
- ✅ Tester avec une BD réelle (pas mockée)

**Pour le monitoring** :
- ✅ Alerte si taux erreurs 5xx > 1% pendant 5min
- ✅ Alerte si nombre commandes chute > 30% en 10min
- ✅ Dashboard avec courbe des commandes

---

## 8. Dashboard minimal proposé

### Bloc 1 : Santé globale
```
┌─────────────────────────────┐
│ SANTÉ GLOBALE               │
├─────────────────────────────┤
│ Status: 🟢 OK               │
│ Version: 1.3.0              │
│ Uptime: 48h 32min           │
│ Dernière alerte: Aucune     │
└─────────────────────────────┘
```

### Bloc 2 : Erreurs
```
┌─────────────────────────────┐
│ ERREURS (dernière heure)     │
├─────────────────────────────┤
│ Erreurs 4xx: 2.5%           │
│ Erreurs 5xx: 0.1%           │
│ Top endpoint en erreur:      │
│   POST /checkout: 0.3%       │
└─────────────────────────────┘
```

### Bloc 3 : Performance
```
┌─────────────────────────────┐
│ PERFORMANCE (dernière heure) │
├─────────────────────────────┤
│ Latence moyenne: 125ms      │
│ Latence p95: 450ms          │
│ Latence p99: 850ms          │
│ Requêtes/min: 1250          │
└─────────────────────────────┘
```

### Bloc 4 : Ressources
```
┌─────────────────────────────┐
│ RESSOURCES CONTAINER        │
├─────────────────────────────┤
│ CPU: 35%                    │
│ RAM: 280MB / 512MB (55%)    │
│ Redémarrages: 0 (24h)       │
└─────────────────────────────┘
```

### Bloc 5 : Métier
```
┌─────────────────────────────┐
│ MÉTRIQUES MÉTIER (24h)      │
├─────────────────────────────┤
│ Commandes: 1,250            │
│ Checkouts réussis: 950      │
│ Taux conversion: 76%        │
│ Revenu: 25 500€             │
└─────────────────────────────┘
```

---

## 9. Checklist post-déploiement

### ✅ Avant le déploiement

- [ ] Branche mergée sur main
- [ ] CI GitHub Actions au vert
- [ ] Code review approuvé
- [ ] Tests unitaires passent
- [ ] Tests d'intégration passent

### ✅ Pendant le déploiement

- [ ] Image Docker correctement taguée
- [ ] Secrets/env vars vérifiés
- [ ] Plan de rollback préparé
- [ ] Équipe support en alerte

### ✅ Juste après le déploiement (5 min)

- [ ] Container est en `running` (pas de crashed)
- [ ] `/health` répond et retourne le bon tag de version
- [ ] `/ready` répond OK
- [ ] Logs du container visibles et sans erreur massive
- [ ] CPU < 50%, RAM < 60%

### ✅ À 15 minutes

- [ ] Taux erreurs 5xx < 1%
- [ ] Latence moyenne < 500ms
- [ ] Au moins 1 commande créée sans erreur
- [ ] Dashboard affiche les bonnes métriques

### ✅ À 1 heure

- [ ] Pas d'augmentation anormale des erreurs
- [ ] Taux conversion stable par rapport à avant
- [ ] Pas de redémarrage du container
- [ ] CPU et RAM stables
- [ ] Les logs ne montrent pas de warning répété

### ✅ À 24 heures

- [ ] Métriques métier comparables à la version précédente
- [ ] Pas de chute de trafic
- [ ] Uptime = 100%
- [ ] Performance stable

### 🚨 Actions correctives en cas de problème

| Problème | Action immédiate | Qui | Délai |
|----------|---|---|---|
| Container crash | Rollback v1.2.5 | DevOps | < 5min |
| Erreurs 5xx > 5% | Investiguer logs | Dev + Ops | < 15min |
| Commandes en baisse | Rollback ou hotfix | Lead dev | < 30min |
| Latence > 1s p95 | Augmenter ressources | Ops | < 1h |

---

## 10. Critères de validation

### Cohérence générale
- ✅ Tous les points du TP sont traités
- ✅ Les propositions sont réalistes et utilisables
- ✅ Les liens entre CI, Docker et observabilité sont explicités

### CI/CD
- ✅ Compréhension du workflow GitHub Actions
- ✅ Identification claire des limites
- ✅ Propositions d'améliorations pertinentes

### Observabilité
- ✅ `/health` et `/ready` bien distincts
- ✅ Logs exploitables et contextualisés
- ✅ Métriques techniques ET métier
- ✅ Alertes avec seuil et action
- ✅ Dashboard cohérent

### Incident
- ✅ Analyse croise CI, Docker, logs et métriques
- ✅ Hypothèse de cause justifiée
- ✅ Décision (rollback/hotfix) argumentée
- ✅ Propositions d'amélioration futures

---

## 11. Questions de validation orale (auto-vérification)

### À pouvoir répondre sans hesiter

**CI/CD** :
- [ ] Que vérifie le workflow GitHub Actions ?
- [ ] Pourquoi construire l'image Docker dans la CI ?
- [ ] Pourquoi tester `/health` dans la CI ?
- [ ] Qu'est-ce que la CI ne peut pas garantir ?

**Docker** :
- [ ] Quelle commande voir les containers actifs ?
- [ ] Quelle commande lire les logs ?
- [ ] Quelle commande voir CPU/RAM ?
- [ ] Que regarder si container redémarre en boucle ?

**Observabilité** :
- [ ] Différence entre `/health` et `/ready` ?
- [ ] Pourquoi des métriques métier ?
- [ ] Quand décider un rollback ?
- [ ] Qu'expose-t-on (ou non) dans `/health` ?

---

## Ressources utiles

### Commandes Docker essentielles

```bash
docker ps                           # Containers actifs
docker logs <container>             # Afficher les logs
docker logs -f <container>          # Suivre les logs
docker logs --tail 50 <container>   # 50 derniers logs
docker inspect <container>          # État complet
docker stats <container>            # CPU/RAM
docker exec <container> sh          # Accès shell
```

### Commandes cURL pour tester

```bash
curl http://localhost:3000/health
curl http://localhost:3000/ready
curl http://localhost:3000/products
curl -X POST http://localhost:3000/orders -H "Content-Type: application/json" -d '{}'
```

### Métriques à tracker

- Uptime de `/health`
- Taux erreurs par endpoint
- Temps réponse p50/p95/p99
- CPU/RAM/Disk
- Commandes/Checkouts/Conversions

---

**Fin du rendu**

*Ce document est une proposition de structure. À adapter selon tes observations et ta compréhension du TP.*
