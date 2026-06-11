# Documentation — Projet Vente (E-commerce)

> Site e-commerce : produits, promotions, vente en gros, et interface admin.
> Stack : **Backend NestJS + MySQL** · **Frontend Nuxt 4 (Vue 3) + Tailwind**.

---

## 1. Architecture générale

```
projet_vente/
├── backend/     → API REST (NestJS 11 + TypeORM + MySQL)   → port 3002
└── frontEnd/    → Site web (Nuxt 4 + Vue 3 + Tailwind)      → port 3001
```

Le frontend appelle le backend en dur sur `http://localhost:3002`
(voir `frontEnd/composables/useApi.ts`).

Le backend n'autorise le **CORS que pour `http://localhost:3001`**
(voir `backend/src/main.ts`) → **le front DOIT tourner sur le port 3001**.

---

## 2. Backend (`backend/`)

### Technologies
- **NestJS 11** (framework Node.js)
- **TypeORM 0.3** (ORM)
- **MySQL** (driver `mysql2`)
- **Multer** (upload d'images)

### Configuration base de données
Fichier `backend/.env` :
```
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=root
DB_PASS=
DB_NAME=shop
```
> `synchronize: true` est activé (`app.module.ts`) → **les tables sont créées
> automatiquement** au démarrage. Pas besoin de migrations en dev.
> ⚠️ À mettre à `false` en production.

### Structure des modules (`backend/src/`)
| Module | Rôle | Route de base |
|--------|------|---------------|
| `product/`   | CRUD produits, recherche, filtres, pagination, upload image | `/product` |
| `promotion/` | Promotions liées à un produit | `/promotions` |
| `wholesale/` | Produits en vente en gros (prix de gros, quantité min) | `/wholesale` |
| `auth/`      | Connexion admin | `/auth` |

### Modèle de données (entités)
- **Product** : `id, name, description, price, quantity, imageUrl, panoramicImageUrl, isPromotion, originalPrice, discountPercentage, createdAt`
- **Promotion** : `id, productId (→Product), discountPercentage, discountedPrice, startDate, endDate, isActive`
- **WholesaleProduct** : `id, productId (→Product), wholesalePrice, minQuantity`
- **Admin** (table `admins`) : `id, email, password, role, createdAt`

### Principaux endpoints
| Méthode | URL | Description |
|---------|-----|-------------|
| GET  | `/product?page=&limit=&search=&minPrice=&maxPrice=&isPromotion=&sortBy=` | Liste paginée + filtres |
| GET  | `/product/promotions` | Produits en promo (paginé) |
| GET  | `/product/top-promotions` | Top 3 promos |
| GET  | `/product/:id` | Un produit |
| POST | `/product` | Créer un produit (multipart, champ image `image`) |
| POST | `/product/bulk` | Créer plusieurs produits |
| PUT  | `/product/:id` | Modifier (multipart) |
| DELETE | `/product/:id` | Supprimer |
| GET  | `/wholesale?page=&limit=&search=` | Liste produits en gros |
| GET  | `/wholesale/:id` | Un produit en gros |
| POST | `/wholesale` | Créer |
| GET  | `/promotions` | Promotions actives |
| POST | `/promotions` | Créer une promotion |
| POST | `/auth/login` | Connexion admin (`{ email, password }`) |
| GET  | `/auth/me` | Profil via header `Authorization: Bearer <token>` |

### ⚠️ Authentification (à savoir)
L'auth admin est **hardcodée** dans `backend/src/auth/auth.service.ts` :
- Email : `bsabensalemahmed@gmail.com`
- Mot de passe : `123456789!`
- Le token retourné est un simple `admin-token-<timestamp>` (pas de vrai JWT, pas de hashage). À refaire pour la prod.

### Upload de fichiers
- Images stockées dans `backend/uploads/`
- Servies en statique sous le préfixe `/uploads/` (ex : `http://localhost:3002/uploads/xxx.jpg`)
- Limite 5 Mo, formats : jpg, jpeg, png, gif

---

## 3. Frontend (`frontEnd/`)

### Technologies
- **Nuxt 4** + **Vue 3** + **Vue Router**
- **Tailwind CSS** (`@nuxtjs/tailwindcss`)

### Pages (`frontEnd/pages/`)
| Page | Rôle |
|------|------|
| `index.vue` | Accueil / catalogue produits |
| `product.vue` | Détail produit |
| `promotions.vue` | Page promotions |
| `gros.vue` | Page vente en gros |
| `contact.vue` | Contact |

### Composables (`frontEnd/composables/`)
| Fichier | Rôle |
|---------|------|
| `useApi.ts` | **Toutes les requêtes vers l'API** (URL backend en dur : `http://localhost:3002`) |
| `useAuth.ts` | Gestion connexion admin |
| `useCart.ts` | Panier |
| `useQuote.ts` | Devis |
| `useFlashTimer.ts` | Compte à rebours promos |

### Composants notables (`frontEnd/components/`)
ProductCard, ProductAddModal, ProductEditModal, Cart, Navbar, FilterSidebar,
LoginModal, Pagination, PromotionCard, WholesaleCard, SearchBar, Quote…

> ⚠️ L'URL de l'API est codée en dur dans `useApi.ts`. Si tu changes le port
> du backend, modifie-la ici aussi.

---

## 4. Lancer le projet en local (étapes)

### Prérequis à installer
1. **Node.js** ≥ 18 (testé avec v22) → déjà installé.
2. **MySQL Server** (8.x recommandé) → **à installer**. Options :
   - **MySQL Community Server** : https://dev.mysql.com/downloads/mysql/
   - ou **XAMPP** (inclut MySQL/MariaDB + interface phpMyAdmin) : https://www.apachefriends.org/
   - ou **Docker** : `docker run --name shop-mysql -e MYSQL_ROOT_PASSWORD= -e MYSQL_ALLOW_EMPTY_PASSWORD=yes -e MYSQL_DATABASE=shop -p 3306:3306 -d mysql:8`

### A. Préparer la base de données
1. Démarrer MySQL (service Windows, XAMPP, ou Docker).
2. Créer la base **`shop`** (les tables seront créées automatiquement par TypeORM) :
   ```sql
   CREATE DATABASE shop;
   ```
3. Vérifier que `backend/.env` correspond à ta config MySQL
   (user `root`, mot de passe vide par défaut — adapte si besoin).

### B. Lancer le backend
```powershell
cd backend
npm install        # si pas déjà fait
npm run start:dev  # mode watch
```
→ API disponible sur **http://localhost:3002**

### C. Lancer le frontend (sur le port 3001 !)
```powershell
cd frontEnd
npm install                 # si pas déjà fait
npm run dev -- --port 3001
```
→ Site disponible sur **http://localhost:3001**

> Important : démarrer sur le **port 3001** (et pas 3000 par défaut), sinon le
> backend bloquera les requêtes (CORS configuré uniquement pour 3001).

### D. Se connecter en admin
Sur le site, ouvrir la modale de connexion et utiliser :
- Email : `bsabensalemahmed@gmail.com`
- Mot de passe : `123456789!`

---

## 5. Récapitulatif des ports
| Service | Port |
|---------|------|
| MySQL | 3306 |
| Backend (API NestJS) | 3002 |
| Frontend (Nuxt) | 3001 |

---

## 6. Système multi-utilisateurs + dashboard (ajouté)

Chaque vendeur a un compte et gère **uniquement ses propres produits** depuis un dashboard.

### Authentification
- Mots de passe **hashés (bcrypt)**, tokens **JWT** signés (expiration 7 jours).
- Secret JWT dans `backend/.env` → `JWT_SECRET` (à changer en prod).
- Table `users` : `id, email, password (hash), name, role ('admin'|'seller'), createdAt`.
- **Pas d'inscription publique** : seul un admin crée les comptes.
- Un **compte admin initial** est créé automatiquement au démarrage du backend
  (`auth.service.ts` → `onModuleInit`) :
  - Email : `bsabensalemahmed@gmail.com` · Mot de passe : `123456789!`

### Endpoints d'authentification
| Méthode | URL | Accès | Description |
|---------|-----|-------|-------------|
| POST | `/auth/login` | public | Connexion → renvoie `{ user, token }` |
| GET  | `/auth/me` | connecté | Profil de l'utilisateur du token |
| POST | `/auth/users` | admin | Créer un compte (`email, password, name?, role?`) |
| GET  | `/auth/users` | admin | Lister les comptes |
| DELETE | `/auth/users/:id` | admin | Supprimer un compte |

### Propriété des produits
- La table `product` a une colonne **`ownerId`** (id du vendeur).
- `POST /product` (token requis) assigne automatiquement `ownerId` = utilisateur connecté.
- `PUT`/`DELETE /product/:id` : autorisés **uniquement au propriétaire** (ou à un admin).
- `GET /product/mine` (token requis) : produits du vendeur connecté (dashboard).
- Les `GET` publics (`/product`, `/product/:id`, etc.) restent ouverts (vitrine).
- Guards : `JwtAuthGuard` (token valide) et `AdminGuard` (rôle admin) dans
  `backend/src/auth/jwt-auth.guard.ts`.

### Frontend
- Pages : **`/dashboard`** (CRUD de ses produits), **`/admin/users`** (admin : gestion des comptes).
- Middlewares : `middleware/auth.ts` (connecté requis), `middleware/admin.ts` (admin requis).
- Le token JWT est stocké dans le cookie `auth-token` et envoyé en header
  `Authorization: Bearer ...` sur les routes protégées (`composables/useApi.ts`).
- Navbar : bouton **Connexion** / lien **Dashboard** selon l'état de connexion.

### Comment l'utiliser
1. Se connecter avec le compte admin (identifiants ci-dessus).
2. Aller dans **Dashboard → Gérer les comptes** (`/admin/users`) et créer des vendeurs.
3. Chaque vendeur se connecte et gère ses produits depuis **`/dashboard`**.

> Note : un compte de test `vendeur1@test.com` / `vendeur123` a été créé lors
> des vérifications — supprime-le depuis `/admin/users` si tu n'en as pas besoin.

---

## 7. Points d'attention / dette technique
- 🟢 Auth sécurisée (bcrypt + JWT) — pensez juste à définir un vrai `JWT_SECRET` en prod.
- 🟠 Identifiants de l'admin initial en dur dans `auth.service.ts` → à changer / paramétrer.
- 🟠 URL API en dur dans le frontend (`useApi.ts`, middlewares) → externaliser dans une variable d'env.
- 🟠 `synchronize: true` activé → ne pas utiliser tel quel en production (risque de perte de données).
- 🟢 Produits existants créés avant le multi-utilisateurs ont `ownerId = NULL`
  (visibles sur la vitrine, modifiables uniquement par un admin).
