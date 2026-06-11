# Rapport d'audit — Architecture, SOLID & Clean Code

> Évaluation des deux projets (backend NestJS + frontend Nuxt).
> **Aucune modification de code n'a été faite** — ceci est un rapport d'aide à la décision.

## Verdict global

Les deux projets sont **fonctionnels, lisibles et bien découpés en modules**. Mais ils
ne respectent pas pleinement les principes SOLID / clean code. Ce sont des défauts
classiques d'un projet qui a grandi vite. **Rien n'est cassé**, mais il y a de la dette
technique réelle à traiter avant une mise en production sérieuse ou une reprise par
quelqu'un d'autre.

Note indicative : **~6,5/10** en l'état (fonctionnel ✅, propreté/robustesse à améliorer).

---

## Backend (NestJS)

### 🔴 Priorité haute (sécurité)

1. **Endpoints de seed non protégés**
   - Fichiers : `src/product/seed-panoramic.controller.ts`, `src/product/add-more-products.controller.ts`
   - `POST /seed/panoramic` et `POST /seed/more-products` créent des produits **sans authentification**.
   - Principe violé : sécurité de base. N'importe qui peut injecter des produits.
   - Fix : supprimer ces controllers (outils de dev), ou les protéger par `JwtAuthGuard + AdminGuard`.

2. **Secrets codés en dur**
   - `src/auth/auth.service.ts` : email + mot de passe admin en clair dans le code.
   - `src/auth/auth.module.ts` : `JWT_SECRET` a un fallback `'dev-secret-change-me'`.
   - Fix : tout passer en variables d'environnement (`.env`), sans fallback de secret en prod.

3. **`synchronize: true`** (`src/app.module.ts`)
   - Pratique en dev, **dangereux en prod** (peut altérer/perdre des données au démarrage).
   - Fix : `false` en prod + migrations TypeORM.

### 🟠 Priorité moyenne (SOLID / clean)

4. **Pas de DTOs ni de validation d'entrée**
   - On utilise `@Body() data: Partial<Product>` et des objets inline.
   - Principe violé : SRP + robustesse. Les données ne sont pas validées (types, champs requis, bornes).
   - Fix : créer des DTOs (`CreateProductDto`, `UpdateProductDto`, `LoginDto`…) avec `class-validator`
     et activer un `ValidationPipe` global. La conversion de types (`isPromotion` string→bool, etc.)
     se ferait via `@Transform` au lieu du `normalize()` manuel.

5. **Logique métier dans le controller**
   - `src/product/product.controller.ts` : `normalize()` et `extractWholesale()` contiennent de la
     logique qui devrait être dans le service ou les DTOs.
   - Principe violé : séparation des responsabilités (le controller doit juste router).

6. **Mapping entité → réponse dupliqué (~10 fois)**
   - `src/product/product.service.ts` : le même mapping `{ id, name, price: Number(...), image: p.imageUrl, ... }`
     est recopié dans `findAll`, `findOne`, `findByOwner`, `findPromotions`, `findTopPromotions`.
   - Principe violé : DRY.
   - Fix : un seul mapper privé `toDto(product)` réutilisé partout.

7. **Configuration Multer dupliquée (3×)**
   - Le bloc `diskStorage({ destination, filename })` + `fileFilter` est recopié dans create, update et upload.
   - Fix : extraire une factory `multerImageOptions()` partagée.

### 🟡 Priorité basse

8. **Modèle de promotion ambigu** : un produit a un champ `isPromotion` + il existe une table
   `promotions` séparée. Redondance / source de confusion.
9. Pas d'enveloppe de réponse ni de gestion d'erreurs standardisée (filtres d'exception).

---

## Frontend (Nuxt)

### 🟠 Priorité moyenne

1. **URL backend en dur (5 fichiers)**
   - `http://localhost:3002` répété dans : `composables/useApi.ts`, `composables/useAuth.ts`,
     `composables/useImageUrl.ts`, `middleware/admin.ts`, `pages/dashboard.vue`.
   - Principe violé : configuration / déployabilité. Impossible de changer d'environnement sans éditer le code.
   - Fix : `runtimeConfig.public.apiBase` dans `nuxt.config.ts` + variable d'env `NUXT_PUBLIC_API_BASE`.

2. **Couleurs / thème en dur partout**
   - Le rose `#e08aaa` / `#b85f82` est répété dans ~10 fichiers (changé par `sed`).
   - Fix : une **variable CSS** (`--brand`, `--brand-dark`) définie une fois (dans `app.vue` ou un CSS global).
     Changer la couleur d'un client = 1 ligne.

3. **`pages/product.vue` et `pages/gros.vue` quasi identiques**
   - ~600 lignes dupliquées (header, sidebar, toolbar, états loading/empty, pagination).
   - Principe violé : DRY.
   - Fix : un composant partagé `ProductListing` paramétré par la source de données.

### 🟡 Priorité basse

4. **15 `console.log`** laissés dans le code applicatif (Navbar, useCart…). À retirer.
5. **`any` généralisé** (`products: any[]`, `shop: any`…). Manque de types partagés.
6. **Hacks de réactivité** dans `composables/useAuth.ts` (`triggerRef`, `nextTick`) — symptôme d'un
   state mal modélisé.
7. **Logique d'image dupliquée** : `useImageUrl()` existe, mais `dashboard.vue` a son propre `imageSrc()`.

---

## Feuille de route proposée (par valeur / risque)

### Niveau 1 — Sûr & rapide (recommandé en premier) — ~30-45 min
- [ ] Supprimer / protéger les endpoints seed
- [ ] Secrets (admin + JWT) en `.env`
- [ ] URL backend en `runtimeConfig` Nuxt
- [ ] Mapper produit unique (back) + factory Multer
- [ ] Variable CSS pour le thème
- [ ] Retirer les `console.log`
- → Risque de régression : très faible.

### Niveau 2 — Structurel (SOLID) — ~1-2h
- [ ] DTOs + `class-validator` + `ValidationPipe` (back)
- [ ] Sortir `normalize`/`extractWholesale` du controller vers service/DTO
- [ ] Factoriser `product.vue` / `gros.vue` en composant partagé
- [ ] Typer le frontend (interfaces partagées, supprimer les `any`)
- → Risque : moyen (à tester après).

### Niveau 3 — Refonte profonde
- [ ] Clarifier le modèle promotions (champ vs table)
- [ ] Enveloppe de réponse + filtres d'exception standardisés
- [ ] Migrations TypeORM, `synchronize: false`
- → Risque : plus élevé, à faire posément.

---

## Conclusion

Pour un projet d'agence (1 boutique = 1 déploiement), le **Niveau 1** apporte déjà
l'essentiel (sécurité + déployabilité + thème configurable en 1 ligne) sans risque.
Le **Niveau 2** vaut le coup dès que le code doit durer / être repris.
Le **Niveau 3** est optionnel selon l'ambition.
