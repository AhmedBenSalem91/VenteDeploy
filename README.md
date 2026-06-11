# Déploiement — Boutique e-commerce

Dépôt d'**orchestration** : il lance ensemble le backend ([VenteBack](https://github.com/VenteBack/VenteBack)),
le frontend ([VenteFront](https://github.com/VenteFront/VenteFront)) et MySQL via Docker.

## Structure attendue

Les 3 dépôts doivent être clonés **côte à côte** :

```
un-dossier/
├── backend/     # git clone .../VenteBack.git backend
├── frontEnd/    # git clone .../VenteFront.git frontEnd
└── deploy/      # ce dépôt  (docker-compose + docs)
```

## Démarrage rapide

```bash
cd deploy
cp .env.example .env        # puis adapter les secrets
docker compose up --build
```

- Site  : http://localhost:3001
- API   : http://localhost:3002

Arrêt : `docker compose down` (ajouter `-v` pour effacer la base et les uploads).

## Documentation

- **[DEPLOIEMENT.md](DEPLOIEMENT.md)** — guide complet (architecture, SOLID/clean code, Docker, CI/CD, prod, checklist)
- **[DOCUMENTATION.md](DOCUMENTATION.md)** — vue d'ensemble du projet
- **[RAPPORT_OPTIMISATION.md](RAPPORT_OPTIMISATION.md)** — audit & feuille de route

## Connexion admin par défaut

`hammaKarmeni@gmail.com` / `Karmeni135135` (configurable via `ADMIN_EMAIL` / `ADMIN_PASSWORD`).
