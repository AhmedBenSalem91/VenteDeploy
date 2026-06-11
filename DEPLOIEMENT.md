# Guide de déploiement — Boutique e-commerce (NestJS + Nuxt)

Documentation complète : architecture, principes (SOLID / clean code), Docker,
CI/CD, et mise en production. Pensée pour un usage **agence** : 1 client = 1 boutique
= 1 déploiement isolé.

---

## 1. Architecture

```
                  ┌────────────────────────┐
   Navigateur ───▶│  Frontend  (Nuxt 4)    │  :3001
                  │  SSR + client          │
                  └───────────┬────────────┘
                              │  HTTP (NUXT_PUBLIC_API_BASE)
                              ▼
                  ┌────────────────────────┐
                  │  Backend  (NestJS 11)  │  :3002
                  │  API REST + JWT        │
                  └───────────┬────────────┘
                              │  TypeORM
                              ▼
                  ┌────────────────────────┐
                  │  MySQL 8               │  :3306
                  └────────────────────────┘
```

- **Frontend** et **Backend** sont 2 dépôts Git séparés (`VenteFront`, `VenteBack`).
- Le navigateur appelle directement le backend → `NUXT_PUBLIC_API_BASE` doit être
  l'**URL publique** du backend (pas l'adresse interne Docker).
- Images uploadées servies par le backend sous `/uploads/`.

### Stack
| Couche | Techno |
|--------|--------|
| Frontend | Nuxt 4, Vue 3, Tailwind, Nitro (SSR) |
| Backend | NestJS 11, TypeORM, JWT (`@nestjs/jwt`), bcrypt, Multer |
| Base | MySQL 8 |
| Conteneurs | Docker + Docker Compose |
| CI/CD | GitHub Actions |

---

## 2. Principes SOLID & Clean Code appliqués

Le projet vise (et applique en grande partie) :

- **S — Single Responsibility** : chaque module Nest (`product`, `auth`, `wholesale`,
  `promotion`) a une responsabilité. Les composables Nuxt (`useApi`, `useAuth`,
  `useCart`) isolent chaque préoccupation.
- **O — Open/Closed** : ajout d'un filtre/critère produit sans réécrire `findAll`
  (paramètres optionnels). Le thème est ouvert à l'extension via variables CSS.
- **L / I / D** : NestJS impose l'**injection de dépendances** (services injectés via
  le constructeur, jamais instanciés à la main) → inversion de dépendance native.
- **DRY** : mappers produit factorisés (`toPublicDto`, `toPromotionDto`), config Multer
  partagée (`multer.config.ts`), résolution d'image centralisée (`useImageUrl`), thème
  centralisé (`assets/css/main.css`).
- **Configuration externalisée** : URL d'API et secrets via variables d'environnement,
  jamais en dur.

> Voir `RAPPORT_OPTIMISATION.md` pour les points encore améliorables (Niveau 2 : DTOs +
> validation `class-validator`, factorisation `product.vue`/`gros.vue`, typage strict).

---

## 3. Prérequis

- **Docker** + **Docker Compose** (recommandé), OU
- **Node.js 22+** et **MySQL 8** pour un lancement manuel.

---

## 4. Lancement local — avec Docker (recommandé)

Les **3 dépôts** doivent être clonés côte à côte :

```
un-dossier/
├── backend/            (dépôt VenteBack)
├── frontEnd/           (dépôt VenteFront)
└── deploy/             (ce dépôt — docker-compose + docs)
```

```bash
git clone .../VenteBack.git  backend
git clone .../VenteFront.git frontEnd
git clone .../Deploy.git     deploy
```

1. Dans `deploy/`, copier la config et adapter les secrets :

```bash
cd deploy
cp .env.example .env
```

2. Lancer toute la stack :

```bash
docker compose up --build
```

3. Accès :
   - Site : **http://localhost:3001**
   - API  : **http://localhost:3002**
   - MySQL: localhost:3306

Arrêt : `docker compose down` (ajouter `-v` pour **effacer** la base et les uploads).

---

## 5. Lancement local — sans Docker (dev)

```bash
# 1. MySQL doit tourner + base "shop" créée

# 2. Backend
cd backend
cp .env.example .env        # puis adapter
npm install
npm run start:dev           # http://localhost:3002

# 3. Frontend (autre terminal)
cd frontEnd
cp .env.example .env        # NUXT_PUBLIC_API_BASE=http://localhost:3002
npm install
npm run dev -- --port 3001  # http://localhost:3001
```

---

## 6. Variables d'environnement

### Backend (`backend/.env`)
| Variable | Rôle |
|----------|------|
| `DB_HOST` `DB_PORT` `DB_USER` `DB_PASS` `DB_NAME` | Connexion MySQL |
| `DB_SYNC` | `true` en dev (crée les tables auto), `false` en prod |
| `JWT_SECRET` | Clé de signature des tokens (longue, aléatoire) |
| `ADMIN_EMAIL` `ADMIN_PASSWORD` | Compte admin créé au 1er démarrage |
| `CORS_ORIGIN` | URL(s) du frontend autorisée(s) — virgules pour plusieurs |
| `PORT` | Port d'écoute du backend (défaut 3002) |

### Frontend (`frontEnd/.env`)
| Variable | Rôle |
|----------|------|
| `NUXT_PUBLIC_API_BASE` | URL **publique** du backend |

> Générer un secret : `openssl rand -hex 32`
> Les `.env` sont **gitignorés** — ne jamais les committer. Utiliser les `.env.example` comme modèle.

---

## 7. CI/CD (GitHub Actions)

Chaque dépôt contient `.github/workflows/ci.yml` :
- **À chaque push / PR sur `main`** : install → lint → build (échoue si le code casse).
- Un job **Docker** (commenté) est prêt : décommenter + ajouter les secrets
  `DOCKER_USERNAME` / `DOCKER_PASSWORD` pour publier l'image automatiquement.

Pipeline type pour la prod :
```
push main ─▶ CI (lint+build) ─▶ build image Docker ─▶ push registry ─▶ déploiement serveur
```

---

## 8. Configuration prod (déjà câblée dans le code)

Ces points sont **désormais configurables par variable d'environnement** (plus rien en dur) :

1. **CORS** (`backend/src/main.ts`) → `CORS_ORIGIN`
   ```ts
   const corsOrigin = process.env.CORS_ORIGIN || 'http://localhost:3001';
   app.enableCors({ origin: corsOrigin.split(',').map(o => o.trim()), credentials: true });
   ```
   En prod : `CORS_ORIGIN=https://maboutique.com`.

2. **Schéma de base** (`backend/src/app.module.ts`) → `DB_SYNC`
   ```ts
   synchronize: process.env.DB_SYNC !== 'false',
   ```
   - **Dev** : `DB_SYNC=true` (les tables se créent toutes seules).
   - **Prod** : `DB_SYNC=false` + **migrations TypeORM** pour faire évoluer le schéma
     sans risque (`npm run typeorm -- migration:generate` puis `migration:run`).

3. **Port** (`backend/src/main.ts`) → `PORT` (défaut 3002).

### Comptes de connexion
- Le **premier admin** est créé au démarrage à partir de `ADMIN_EMAIL` / `ADMIN_PASSWORD`.
- Les **autres comptes** (vendeurs ou admins) se créent depuis le **Dashboard → Gérer les comptes**
  (ou `POST /auth/users`, réservé admin).
- La connexion accepte un **email OU un identifiant simple** (ex : `hammaKarmeni`).

---

## 9. Mise en production (VPS)

Schéma recommandé : un **reverse proxy** (Nginx ou Traefik) devant les conteneurs,
avec HTTPS (Let's Encrypt).

```
Internet ─▶ Nginx/Traefik (443, HTTPS)
              ├─ maboutique.com       ─▶ frontend:3000
              └─ api.maboutique.com   ─▶ backend:3002
```

Étapes :
1. Pointer 2 sous-domaines (`maboutique.com`, `api.maboutique.com`) vers le serveur.
2. `NUXT_PUBLIC_API_BASE=https://api.maboutique.com` et `CORS_ORIGIN=https://maboutique.com`.
3. `docker compose up --build -d`.
4. Configurer le reverse proxy + certificats TLS.
5. Sauvegardes : volume `mysql_data` (base) + volume `uploads` (images) → backup régulier.

Alternative sans serveur à gérer : **frontend sur Vercel/Netlify**, **backend + MySQL
sur Railway/Render/Fly.io** (chacun lit ses variables d'env).

---

## 10. Workflow « agence » (1 client = 1 boutique)

Pour un nouveau client :
1. **Dupliquer** les 2 dépôts (ou créer une branche/fork par client).
2. **Rebrander** : changer le logo (`frontEnd/public/`), le nom et les textes
   (`pages/index.vue`, `Navbar.vue`), et **les 2 lignes de couleur** dans
   `frontEnd/assets/css/main.css` (`--brand`, `--brand-dark`).
3. **Configurer** les `.env` (base, secrets, URLs du client).
4. **Déployer** via `docker compose up --build -d` sur l'infra du client.

Chaque client est ainsi **isolé** (sa base, ses images, son domaine, son branding).

---

## 11. Checklist de déploiement

- [ ] `.env` backend et frontend remplis (secrets forts, pas de valeurs par défaut)
- [ ] `JWT_SECRET` généré aléatoirement
- [ ] `ADMIN_PASSWORD` changé
- [ ] `CORS_ORIGIN` = URL réelle du frontend
- [ ] `DB_SYNC=false` en prod (+ migrations si le schéma évolue)
- [ ] HTTPS actif (reverse proxy + certificats)
- [ ] Volumes `mysql_data` et `uploads` sauvegardés
- [ ] CI verte sur les 2 dépôts
- [ ] Branding client appliqué (logo, couleurs, textes)
```
