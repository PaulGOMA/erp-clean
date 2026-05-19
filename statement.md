### Enoncé

Ce document est un **énoncé complet et réaliste** pour implémenter un mini‑projet basé sur **Clean Architecture** :  
- **Users Backend** : Django (auth, users, MySQL, API REST)  
- **Business Backend** : règles métiers pures (FastAPI ou service Python), consommation de l’API Users et worker asynchrone via Redis/RabbitMQ  
- **Frontend** : Vue 3 + TypeScript SPA  
- **Orchestration** : Docker Compose  

---

### 1 Objectifs et périmètre

**Objectif principal**  
Construire un mini‑ERP minimal mais réaliste où :  
- Django gère **utilisateurs, rôles, authentification** et expose une API REST (OpenAPI).  
- Le backend métier implémente **règles métiers** (pricing, validation, génération d’événements) en Clean Architecture, indépendant de Django.  
- La SPA Vue/TS consomme l’API métier pour afficher et déclencher calculs.  
- Communication synchrone via **API REST** et asynchrone via **message queue** (Redis streams ou RabbitMQ).

**Périmètre fonctionnel minimal (MVP)**  
- CRUD utilisateurs (email, rôle) + authentification JWT.  
- Endpoint métier `POST /compute-price` : calcule prix final d’une commande selon règles métiers.  
- Workflow asynchrone : génération de facture PDF (simulée) déclenchée par message.  
- Frontend : formulaire pour créer user et calculer prix.  
- Tests : unitaires, contract tests, intégration DB, E2E via Docker Compose.

**Contraintes non fonctionnelles**  
- Clean Architecture strict : **domaine pur** sans dépendances infra.  
- Tests automatisés et reproductibles.  
- Déploiement local via Docker Compose.  
- Sécurité minimale : authentification service‑to‑service (JWT) pour appels internes.

---

### 2 Architecture et composants

**Vue d’ensemble**  
- **users-backend (Django)** : couche *domain/application/infrastructure/interface* ; expose `/api/users/`, `/api/auth/`.  
- **business-backend (FastAPI)** : domaine métier pur + adaptateurs infra (UsersAPI, MQ client) + interface HTTP.  
- **worker** : consommateur MQ pour tâches longues (génération facture).  
- **frontend (Vue3 + TS)** : SPA qui appelle business-backend.  
- **mysql, redis/rabbitmq** : services d’infra.

**Réseau et orchestration**  
- Docker Compose crée un réseau interne ; services se joignent par nom (`users-backend:8000`, `business-backend:5000`, `mysql:3306`, `redis:6379`).

**Comparaison rapide des modes de communication**

| Mode | Usage | Avantages | Inconvénients |
|------|-------|----------|---------------|
| API REST | Lecture/écriture synchrone | Simple, traçable, OpenAPI | Bloquant pour tâches longues |
| Message Queue | Workflows asynchrones | Découplage, scalabilité | Complexité, monitoring |
| DB partagée | Prototype rapide | Simple | Couplage fort, anti-pattern |

**Composants techniques recommandés**  
- Django + DRF + SimpleJWT  
- FastAPI + Pydantic + requests  
- Redis (streams) ou RabbitMQ  
- MySQL 8  
- Vue 3 + Vite + TypeScript  
- pytest, pytest-django, factory_boy, httpx pour tests

---

### 3 Modèle de domaine et règles métiers détaillées

#### Entités principales
- **User** : `id:int`, `email:str`, `role:str` (`user`, `premium`, `admin`)  
- **Order** (domaine métier) : `id:int`, `user_id:int`, `items:list[Item]`, `base_price:float`, `currency:str`  
- **Invoice** : `id:int`, `order_id:int`, `amount:float`, `status:str` (`pending`, `generated`, `failed`)
- **Item** : `id:int`, `name:str`, `description:str`, `unit_price:float`, `quantity:int`, `satus:str` (`available`, `unavailable`)

#### Règles métiers principales (avec exceptions)
Chaque règle est décrite avec **préconditions**, **effet**, **exceptions levées** et **codes d’erreur**.

1. **Règle Discount par rôle**
   - **Description** : si `user.role == "premium"` appliquer **10%** de réduction.
   - **Préconditions** : user existant et actif ; `base_price >= 0`.
   - **Effet** : `final_price = round(base_price * 0.9, 2)` sinon `base_price`.
   - **Exceptions** :
     - `UserNotFoundError` → HTTP 404 (code `USR_404`) si user introuvable.
     - `InvalidPriceError` → HTTP 400 (code `PRC_400`) si `base_price < 0`.
   - **Idempotence** : calcul pur, idempotent.

2. **Règle Minimum Order Fee**
   - **Description** : si `final_price < 5.00` appliquer frais fixe `+1.00`.
   - **Préconditions** : `final_price` calculé.
   - **Exceptions** : aucune additionnelle.
   - **Rounding** : arrondir à 2 décimales.

3. **Règle Taxation**
   - **Description** : appliquer taxe selon `country` (MVP : taux fixe 20% pour FR).
   - **Préconditions** : `order.currency` et `user` validés.
   - **Effet** : `final_with_tax = final_price * (1 + tax_rate)`.
   - **Exceptions** :
     - `TaxConfigMissingError` → HTTP 500 (code `TAX_500`) si config taxe manquante.

4. **Règle Validation des Items**
   - **Description** : chaque item doit avoir `price >= 0`, `quantity >= 1`.
   - **Exceptions** :
     - `InvalidItemError` → HTTP 422 (code `ITEM_422`) avec détails par item.

5. **Workflow Facture Asynchrone**
   - **Étapes** :
     1. Business backend publie message `generate_invoice` avec `order_id`.
     2. Worker consomme, récupère order + user via API, génère facture (simulée), stocke `Invoice` en DB ou envoie résultat via callback.
   - **Exceptions** :
     - `InvoiceGenerationError` → message de retry ; après N tentatives → statut `failed` et alerte.
     - `ExternalAPITimeoutError` → retry exponentiel.
   - **Idempotence** : message doit contenir `idempotency_key` pour éviter double génération.

6. **Règle d’accès**
   - **Description** : seules les requêtes authentifiées avec JWT peuvent appeler endpoints métier. Certaines actions (ex : création user) réservées aux `admin`.
   - **Exceptions** :
     - `UnauthorizedError` → HTTP 401 (code `AUTH_401`).
     - `ForbiddenError` → HTTP 403 (code `AUTH_403`).

#### Exceptions et schéma d’erreur standard
**Format d’erreur JSON** :
```json
{
  "error": {
    "code": "PRC_400",
    "message": "Invalid base_price: must be >= 0",
    "details": {"field": "base_price"}
  }
}
```
**Liste d’exceptions clés** :
- `UserNotFoundError` → `USR_404` → 404  
- `InvalidPriceError` → `PRC_400` → 400  
- `InvalidItemError` → `ITEM_422` → 422  
- `UnauthorizedError` → `AUTH_401` → 401  
- `ForbiddenError` → `AUTH_403` → 403  
- `InvoiceGenerationError` → `INV_500` → 500  
- `ExternalAPITimeoutError` → `EXT_504` → 504

---

### 4 API Contracts et messages

#### OpenAPI minimal pour Users (extrait)
```yaml
openapi: 3.0.3
info:
  title: Users API
  version: 1.0.0
paths:
  /api/users/:
    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email]
              properties:
                email: { type: string, format: email }
                role: { type: string, enum: [user, premium, admin] }
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      properties:
        id: { type: integer }
        email: { type: string }
        role: { type: string }
```

#### OpenAPI minimal pour Business
```yaml
openapi: 3.0.3
info:
  title: Business API
  version: 1.0.0
paths:
  /compute-price:
    post:
      summary: Compute final price
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [user_id, base_price]
              properties:
                user_id: { type: integer }
                base_price: { type: number }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  final_price: { type: number }
        '4xx':
          description: Client error
```

#### Schéma de message MQ (JSON)
**Topic / Stream** : `business.commands`  
**Message `generate_invoice`** :
```json
{
  "type": "generate_invoice",
  "order_id": 1001,
  "user_id": 42,
  "idempotency_key": "uuid-v4",
  "timestamp": "2026-05-12T10:00:00Z"
}
```

---

### 5 Implémentation pratique et tests

#### Arborescence recommandée (extrait)
```
users_backend/
  domain/
  application/
  infrastructure/
  interface/
business_backend/
  src/
    domain/
    application/
    infrastructure/
    interface/
frontend/
docker-compose.yml
```

#### Docker Compose minimal (extrait)
```yaml
version: "3.8"
services:
  mysql: { image: mysql:8, environment: { MYSQL_ROOT_PASSWORD: password, MYSQL_DATABASE: erp } }
  redis: { image: redis:7 }
  users-backend: { build: ./users_backend, ports: ["8000:8000"], depends_on: [mysql] }
  business-backend: { build: ./business_backend, ports: ["5000:5000"], depends_on: [users-backend, redis] }
  worker: { build: ./business_backend, command: ["python", "worker.py"], depends_on: [redis, users-backend] }
  frontend: { build: ./frontend, ports: ["5173:5173"], depends_on: [business-backend] }
```

#### Variables d’environnement essentielles
- `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` (Django)  
- `USERS_API_URL` (business)  
- `JWT_SECRET`, `JWT_ALGORITHM`, `JWT_EXP` (auth)  
- `REDIS_URL` ou `RABBITMQ_URL` (MQ)

#### Tests
- **Unit tests domaine** : tests purs sans infra (pytest).  
- **Contract tests** : Pydantic models partagés ou OpenAPI validation ; tests qui valident que Users API renvoie le schéma attendu.  
- **Integration tests Django** : `pytest-django` avec DB MySQL de test (docker).  
- **Integration tests business** : tests qui démarrent users-backend (docker) et appellent `/compute-price`.  
- **E2E** : `docker compose -f docker-compose.test.yml up --build` puis scripts qui simulent scénarios complets.  
- **Tests DB** : tests de migrations, contraintes uniques, FK, cascade.

#### CI pipeline recommandé
- Étapes : lint (flake8/ruff), unit tests, contract tests, build images, integration tests (docker-compose test), E2E smoke tests.  
- Exécuter tests DB et E2E sur runner capable de Docker.

#### Observabilité et monitoring
- **Logs structurés** JSON (niveau INFO/ERROR).  
- **Metrics** : exposer `/metrics` (Prometheus) dans business-backend et worker.  
- **Tracing** : optionnel, OpenTelemetry pour corréler requêtes API et worker.

---

### 6 Critères d’acceptation, livrables et plan de travail

**Critères d’acceptation MVP**
- Création et récupération d’un user via `/api/users/`.  
- Endpoint `/compute-price` retourne prix correct selon règles (tests unitaires et integration).  
- Worker consomme message `generate_invoice` et produit un résultat simulé (statut `generated`).  
- Frontend permet créer user et calculer prix.  
- Suite de tests automatisés (unit + contract + integration) qui passent en CI.

**Livrables attendus**
1. Repo Git avec 3 dossiers `users_backend`, `business_backend`, `frontend`.  
2. `docker-compose.yml` fonctionnel pour dev.  
3. Tests unitaires et d’intégration.  
4. OpenAPI pour Users et Business.  
5. README avec commandes pour démarrer, tester et déployer localement.

**Plan de travail suggéré (itératif)**
1. Initialiser repo et Docker Compose minimal (mysql, users-backend).  
2. Implémenter Users CRUD + JWT. Tests unitaires modèles.  
3. Implémenter business-backend domaine `compute_price` + adapter UsersAPI. Tests unitaires domaine.  
4. Exposer `/compute-price` et tester intégration via Docker.  
5. Ajouter MQ et worker pour `generate_invoice`.  
6. Développer frontend minimal.  
7. Ajouter tests E2E et CI.

---

**Annexe utile rapide**

**Exemple d’exception Python (domaine)** :
```python
class DomainError(Exception):
    code: str

class UserNotFoundError(DomainError):
    code = "USR_404"
```

**Politique de retry worker**  
- Retry exponentiel : 1s, 2s, 4s, 8s ; max 5 tentatives.  
- Après échec final : marquer `Invoice` `failed`, publier événement `invoice.failed`.

---

### 7 Table des cas d’usage essentiels

| **Use case** | **Déclencheur** | **Sync/Async** | **Entrées clés** | **Sorties clés** |
|---|---:|---:|---|---|
| **Créer un utilisateur** | Frontend / Admin | Synchrone | `email`, `role` | `User` créé (id, email, role) |
| **Récupérer un utilisateur** | Frontend / service métier | Synchrone | `user_id` | `User` (id, email, role) |
| **Authentifier / JWT** | Frontend | Synchrone | `email`, `password` | `access_token`, `refresh_token` |
| **Mettre à jour un utilisateur** | Frontend / Admin | Synchrone | `user_id`, champs modifiés | `User` mis à jour |
| **Calculer prix final** | Frontend / API interne | Synchrone | `user_id`, `order` ou `base_price` | `final_price`, breakdown |
| **Valider commande** | Frontend / worker | Synchrone ou Async | `order` (items, qty, prices) | `validation_result` ou message d’erreur |
| **Générer facture** | Business backend (commande) | Asynchrone | `order_id`, `idempotency_key` | `Invoice` (status, url ou blob) |
| **Lister commandes / factures** | Frontend / Admin | Synchrone | filtres | liste paginée |
| **Opérations admin** | Admin UI | Synchrone | actions (ban, role change) | statut OK |

---

#### Détails par use case et règles métier associées

##### **Créer un utilisateur**
- **Endpoint** : `POST /api/users/` (Django)  
- **Règles** : email unique ; rôle parmi `user|premium|admin`.  
- **Préconditions** : requête authentifiée si création par admin ; validation email.  
- **Exceptions** :
  - `USR_409` (409) **EmailAlreadyExistsError** si email déjà présent.  
  - `USR_400` (400) **InvalidUserDataError** si format invalide.  
- **Tests** : unit tests pour repository, contract test pour la réponse JSON.

##### **Récupérer un utilisateur**
- **Endpoint** : `GET /api/users/{id}/`  
- **Règles** : seuls `admin` ou le propriétaire peuvent voir certains champs sensibles.  
- **Exceptions** :
  - `USR_404` (404) **UserNotFoundError**.  
  - `AUTH_403` (403) **ForbiddenError** si accès non autorisé.

##### **Authentifier / JWT**
- **Endpoint** : `POST /api/auth/token/`  
- **Règles** : tokens signés, durée configurable, refresh token.  
- **Exceptions** :
  - `AUTH_401` (401) **UnauthorizedError** pour identifiants invalides.  
  - `AUTH_429` (429) **TooManyAttemptsError** pour brute force protection.

##### **Calculer prix final (use case métier central)**
- **Endpoint** : `POST /compute-price` (business-backend)  
- **Pipeline** :
  1. **Récupérer user** via Users API (adapter `UserRepository`).  
  2. **Valider items** (prix >= 0, qty >= 1).  
  3. **Appliquer discount** selon rôle (`premium` → -10%).  
  4. **Appliquer minimum fee** si < 5.00 → +1.00.  
  5. **Appliquer taxe** selon configuration (ex : FR 20%).  
  6. **Retourner breakdown** : base, discount, fees, tax, final.  
- **Exceptions** :
  - `USR_404` (404) si user introuvable.  
  - `PRC_400` (400) **InvalidPriceError** si base_price < 0.  
  - `ITEM_422` (422) **InvalidItemError** si items invalides (détails par item).  
  - `TAX_500` (500) **TaxConfigMissingError** si config taxe absente.
- **Idempotence** : calcul pur → idempotent ; inclure `request_id` pour traçabilité.
- **Tests** : unit tests domaine (compute_price), use case tests avec fake repo, contract tests pour format de réponse.

##### **Valider commande**
- **But** : vérifier intégrité des items, disponibilité stock (simulé), limites business.  
- **Règles** :
  - chaque item `price >= 0`, `quantity >= 1`.  
  - vérification de règles métier (ex : max quantity par user role).  
- **Exceptions** :
  - `ITEM_422` (422) avec `details` listant les items invalides.  
  - `STK_409` (409) **OutOfStockError** si stock insuffisant.
- **Sync vs Async** : validation rapide synchrone ; vérifications externes (stock tiers) asynchrones.

##### **Générer facture (workflow asynchrone)**
- **Flux** :
  1. Business-backend publie message `generate_invoice` sur `business.commands`.  
  2. Worker consomme, récupère order + user via API, génère PDF (simulé), stocke Invoice.  
  3. Worker publie événement `invoice.generated` ou `invoice.failed`.  
- **Messages** : inclure `idempotency_key`, `attempt`, `timestamp`.  
- **Exceptions et gestion** :
  - `EXT_504` (504) **ExternalAPITimeoutError** → retry exponentiel.  
  - `INV_500` (500) **InvoiceGenerationError** → retry jusqu’à N, puis marquer `failed`.  
  - Sur échec final, publier `invoice.failed` et alerter (log/metric).  
- **Idempotence** : obligatoire ; worker doit ignorer messages déjà traités (clé idempotence).

##### **Opérations admin et listing**
- **Endpoints** : `GET /api/orders`, `GET /api/invoices`, `PATCH /api/users/{id}/role`  
- **Règles** : pagination, filtres, tri ; actions réservées aux `admin`.  
- **Exceptions** :
  - `AUTH_403` (403) si non admin.  
  - `PAG_400` (400) pour paramètres de pagination invalides.

---

#### Communication entre use cases et gestion des erreurs

- **Contract tests** : définir Pydantic/OpenAPI pour `User`, `Order`, `Invoice`. Valider que Users API respecte le contrat attendu par business-backend.  
- **Mapping d’erreurs** : transformer exceptions domaine en réponses HTTP standardisées (format JSON d’erreur avec `code`, `message`, `details`).  
- **Retries et timeouts** : tous les appels HTTP internes doivent avoir timeout court (ex : 2s) et stratégie de retry limitée (ex : 3 tentatives, backoff).  
- **Observabilité** : chaque use case doit émettre logs structurés et métriques (compteurs d’erreurs, latence).

---

#### Tests recommandés par use case et critères d’acceptation

- **Unit tests domaine** : couverture pour `compute_price`, `validate_items`, règles de taxe et fees.  
- **Use case tests** : mock `UserRepository` pour tester `ComputePriceUseCase` sans HTTP.  
- **Contract tests** : valider réponse `GET /api/users/{id}` correspond au schéma `User`.  
- **Integration tests** : lancer users-backend + business-backend via Docker Compose et exécuter scénarios : création user → compute-price → assert final.  
- **E2E** : frontend crée user puis déclenche calcul et vérifie affichage du breakdown.  
- **Worker tests** : tests d’intégration MQ simulée (Redis) pour `generate_invoice` avec cas de retry et idempotence.

---

#### Priorisation et roadmap technique

- **Phase 1** : Users CRUD + JWT ; domain tests.  
- **Phase 2** : Business domain `compute_price` + adapter UsersAPI ; unit + contract tests.  
- **Phase 3** : Exposer `/compute-price` ; integration tests via Docker Compose.  
- **Phase 4** : Worker MQ `generate_invoice` avec idempotence et retries.  
- **Phase 5** : Frontend Vue3+TS ; E2E tests.  
- **Phase 6** : Hardening (auth service-to-service, monitoring, CI).

---
