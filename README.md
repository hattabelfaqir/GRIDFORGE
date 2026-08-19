<title>GRIDFORGE — Documentation du projet</title>

# GRIDFORGE — Boutique FiveM (bases, scripts, bots Discord)

Documentation de référence du projet. Le squelette de fichiers/dossiers est déjà créé (vide, sans logique) et l'infra Docker (MySQL + phpMyAdmin) est prête à démarrer — à toi de coder le contenu.

---

## Table des matières

1. [Démarrage rapide](#1-démarrage-rapide)
2. [Contexte du projet](#2-contexte-du-projet)
3. [Stack technique](#3-stack-technique)
4. [Modèle de données](#4-modèle-de-données)
5. [Flux détaillés](#5-flux-détaillés)
6. [Sécurité — check-list](#6-sécurité--check-list)
7. [Arborescence du projet](#7-arborescence-du-projet)
8. [Où coder quoi — feuille de route](#8-où-coder-quoi--feuille-de-route)
9. [Volet légal (France)](#9-volet-légal-france)
10. [Documentation officielle & vidéos](#10-documentation-officielle--vidéos)
11. [Vérification](#11-vérification)

---

## 1. Démarrage rapide

Tout tourne dans Docker : backend (Express), frontend (compilateur Tailwind), MySQL et phpMyAdmin.

```bash
docker compose up -d --build
```

⚠️ Sur cette machine (WSL2), Docker Desktop est installé côté Windows mais **l'intégration WSL n'est pas activée** pour cette distro — la commande `docker` n'est pas trouvée ici. Pour corriger : ouvre Docker Desktop → Settings → Resources → WSL Integration → active l'intégration pour cette distribution, puis relance un terminal.

Une fois les conteneurs lancés :
- **Backend** : http://localhost:3000 (ne répondra qu'une fois que tu auras écrit du code dans `backend/src/app.js` — pour l'instant le fichier est vide exprès, donc le conteneur tourne mais ne sert rien).
- **phpMyAdmin** : http://localhost:8080 — connexion avec les identifiants du `backend/.env` (`DB_USER` / `DB_PASSWORD`), serveur = `mysql`.
- **Frontend** : pas de port exposé, il compile en continu `frontend/src/input.css` → `backend/public/css/style.css` (Tailwind CSS v4).

Arrêter la stack : `docker compose down` (ajoute `-v` pour aussi supprimer les données MySQL).

Commandes utiles :
```bash
docker compose logs -f backend      # suivre les logs du serveur Node
docker compose exec backend sh      # ouvrir un shell dans le conteneur backend
npx prisma studio                   # interface visuelle Prisma (depuis backend/, en dehors de Docker)
```

---

## 2. Contexte du projet

Site marchand qui vend des bases FiveM (scripts), des bots Discord liés à FiveM, et des configurations Discord. Points durs de l'architecture :

- Compte classique email/mot de passe, **mais** obligation de lier un compte Discord avant tout achat (1 Discord = 1 compte site, unique).
- Livraison automatique après paiement (fichier à télécharger + accès depuis le compte).
- Verrouillage des scripts vendus à la license FiveM du serveur acheteur (anti-partage).

★ Insight ─────────────────────────────────────
Il n'existe **aucun bouton officiel "Se connecter avec FiveM"** pour un site tiers — Cfx.re (l'éditeur de FiveM) n'a pas d'OAuth public comme Discord. D'où l'architecture à deux niveaux : **Discord OAuth** pour l'identité de compte (unique, vérifiable), et la **license FiveM** du serveur du client (capturée via une petite ressource FXServer) pour verrouiller les scripts vendus contre le partage.
─────────────────────────────────────────────────

**Hébergement — choix auto-hébergé (Docker)** : pour que phpMyAdmin fonctionne, la base doit être MySQL/MariaDB, ce que les hébergeurs Postgres serverless (Neon/Supabase) ne proposent pas. On est donc passés d'un modèle "Vercel + base managée" à une stack **Docker auto-hébergée** — la même stack tourne en local aujourd'hui pour coder, et pourra tourner telle quelle plus tard sur ton VPS FiveM en production.

---

## 3. Stack technique

| Brique | Choix | Pourquoi |
|---|---|---|
| Runtime / serveur | Node.js + Express | Proche de ce que tu connais déjà (JS). |
| Rendu des pages | EJS (templates HTML + JS server-side) | Pas besoin d'apprendre React. |
| Base de données | **MySQL 8** (conteneur Docker) | Nécessaire pour phpMyAdmin. |
| Admin base de données | **phpMyAdmin** (conteneur Docker, port 8080) | Interface visuelle pour consulter/éditer la base. |
| Accès BDD | Prisma ORM (`provider = "mysql"`) | Migrations versionnées, pas de SQL à la main. |
| Auth email/mdp | `bcrypt` + `express-session` | Standard, bien documenté. |
| Stockage session | `express-mysql-session` (+ `mysql2`) | Sessions stockées en base MySQL, pas en mémoire. |
| Login Discord | `passport-discord-auth` | `passport-discord` (l'ancien choix) est abandonné — celui-ci est son remplaçant maintenu. |
| Paiement | Stripe Checkout + Webhooks | Page de paiement hébergée par Stripe. |
| Email transactionnel | Resend | API simple, bonne délivrabilité. |
| Stockage des fichiers vendus | Volume Docker (`backend/storage/`) | Pas de gros fichiers en pièce jointe email. |
| Validation des entrées | `zod` | Rejette les payloads malformés. |
| Sécurité HTTP | `helmet`, `express-rate-limit`, `csrf-csrf` | Voir section 6. |
| Bots Discord | `discord.js` | Livraison de rôle auto / notifications d'achat. |
| Styles | Tailwind CSS v4 (`@tailwindcss/cli`, conteneur `frontend`) | Compile `frontend/src/input.css` → `backend/public/css/style.css`. |
| Infra | Docker + Docker Compose | Un seul environnement à comprendre, réutilisable en prod sur ton VPS. |

---

## 4. Modèle de données

```
users
  id, email (unique), password_hash, created_at
  discord_id (unique, nullable jusqu'à liaison)
  discord_username, discord_avatar

fivem_licenses
  id, user_id (FK), license_identifier (unique), server_label, linked_at

products
  id, slug, name, description, price_cents, currency, type (script | discord_bot | discord_template)
  active, created_at

product_files
  id, product_id (FK), storage_key, version, checksum

orders
  id, user_id (FK), stripe_checkout_session_id (unique), status (pending|paid|failed|refunded), amount_cents, created_at

order_items
  id, order_id (FK), product_id (FK), unit_price_cents

license_keys
  id, order_item_id (FK), user_id (FK), fivem_license_id (FK nullable), key_value (unique), status (active|revoked), created_at

download_tokens
  id, order_item_id (FK), token (unique), expires_at, max_uses, use_count

webhook_events_log
  id, stripe_event_id (unique), type, received_at, processed_at
```

Contraintes clés à mettre en base :
- `users.discord_id` → `UNIQUE`
- `fivem_licenses.license_identifier` → `UNIQUE`
- `orders.stripe_checkout_session_id` → `UNIQUE` + `webhook_events_log.stripe_event_id` → `UNIQUE` (idempotence des webhooks)

À traduire dans `backend/prisma/schema.prisma` (actuellement seuls le `datasource` et le `generator` y sont définis — les modèles ci-dessus sont à écrire).

---

## 5. Flux détaillés

### 5.1 Inscription / connexion classique
1. Formulaire email + mot de passe → hash `bcrypt` (cost 12) → ligne `users`.
2. Email de vérification (recommandé) via Resend.
3. Connexion → session créée, cookie `httpOnly; secure; sameSite=lax`, stockée en MySQL via `express-mysql-session`.

### 5.2 Liaison Discord (obligatoire avant achat)
1. Bouton "Lier mon Discord" → OAuth Discord (`passport-discord-auth`).
2. Callback : Discord renvoie l'`id` Discord.
3. Vérifier qu'aucun autre `users` n'a déjà ce `discord_id` avant d'écrire → message clair si déjà pris.
4. Middleware `requireDiscordLinked` sur les routes d'achat.

### 5.3 Liaison de la license FiveM (produits "script")
1. Le client colle la license key de son serveur (`keymaster.fivem.net`) **ou** rejoint le petit serveur FXServer de vérification (`fivem-verify-server/verify/`) qui capture l'identifiant `license` via `GetPlayerIdentifiers` et l'envoie à `POST /api/fivem/verify` (secret partagé).
2. Contrainte `UNIQUE` en base empêche la réutilisation sur plusieurs comptes.

### 5.4 Achat (Stripe Checkout)
1. `POST /checkout/:productSlug` → session Stripe Checkout avec `metadata: { user_id, product_id }`.
2. Redirection `/merci` après paiement — **jamais** de livraison à cette étape (pas fiable).
3. `POST /webhooks/stripe` reçoit `checkout.session.completed` :
   - Vérifier la signature Stripe.
   - Vérifier l'idempotence (`webhook_events_log`).
   - Créer `orders` + `order_items`.
   - Générer `license_key` (si script) + `download_token` (expirant, usage limité).
   - Email de livraison via Resend + déblocage dans "Mes achats".

### 5.5 Téléchargement sécurisé
- `GET /download/:token` → vérifie expiration, `use_count < max_uses`, et que le token appartient bien à l'utilisateur connecté.
- Stream depuis `backend/storage/`.

### 5.6 Vérification anti-partage côté script FiveM livré
- Le script vendu appelle `POST /api/license/verify` (`{ key_value, server_license }`) au démarrage.
- Réponse `valid: false` → le script s'arrête.
- Logger chaque vérification pour repérer une clé partagée (trop de licenses différentes pour une même clé).

---

## 6. Sécurité — check-list

- **Mots de passe** : `bcrypt`, jamais en clair dans les logs.
- **Sessions** : cookies `httpOnly`, `secure`, `sameSite=lax`, stockage MySQL.
- **CSRF** : `csrf-csrf` sur les formulaires qui changent un état (profil, liaison Discord/FiveM…).
- **Rate limiting** : `express-rate-limit` sur `/login`, `/register`, `/api/license/verify`, `/api/fivem/verify`.
- **Webhooks** : toujours vérifier la signature Stripe.
- **Secrets** : jamais commit — tout passe par `backend/.env` (déjà dans `.gitignore`).
- **Validation** : `zod` sur chaque body reçu.
- **Headers** : `helmet()` en middleware global.
- **Téléchargements** : tokens signés, expirants, usage limité.

---

## 7. Arborescence du projet

```
/
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── .env                    # rempli (DB + secret de session) — à compléter (Discord/Stripe/Resend)
│   ├── package.json
│   ├── prisma.config.cjs       # config Prisma 7 (URL de connexion)
│   ├── prisma/schema.prisma    # datasource + generator prêts, modèles à écrire
│   ├── storage/                # fichiers vendus (volume)
│   ├── public/
│   │   ├── css/                # généré par le conteneur frontend (Tailwind)
│   │   ├── js/
│   │   └── images/
│   └── src/
│       ├── app.js              # vide — point d'entrée Express à écrire
│       ├── routes/
│       │   ├── auth.js
│       │   ├── discord.js
│       │   ├── fivem.js
│       │   ├── shop.js
│       │   ├── checkout.js
│       │   ├── webhooks.js
│       │   ├── account.js
│       │   └── api/license.js
│       ├── middlewares/
│       │   ├── requireAuth.js
│       │   ├── requireDiscordLinked.js
│       │   └── rateLimit.js
│       ├── services/
│       │   ├── stripe.js
│       │   ├── email.js
│       │   ├── storage.js
│       │   └── licenseKeys.js
│       └── views/
│           ├── shop/
│           ├── account/
│           └── auth/
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json            # tailwindcss + @tailwindcss/cli
│   └── src/input.css           # `@import "tailwindcss";`
└── fivem-verify-server/
    └── verify/                 # ressource FXServer à déployer sur ton VPS FiveM (pas dans Docker)
        ├── fxmanifest.lua
        └── server.lua
```

Tous les fichiers `.js`/`.lua`/`.ejs` listés ci-dessus sont **vides** — c'est le point de départ pour coder.

---

## 8. Où coder quoi — feuille de route

| Étape | Quoi | Fichiers concernés |
|---|---|---|
| 1 | Modèles Prisma + première migration | `backend/prisma/schema.prisma` |
| 2 | Auth email/mot de passe (register, login, logout, session) | `backend/src/routes/auth.js`, `backend/src/app.js` |
| 3 | Liaison Discord obligatoire (OAuth + unicité) | `backend/src/routes/discord.js`, `backend/src/middlewares/requireDiscordLinked.js` |
| 4 | Catalogue produits (listing/détail) | `backend/src/routes/shop.js`, `backend/src/views/shop/` |
| 5 | Paiement Stripe (Checkout + webhook) | `backend/src/routes/checkout.js`, `backend/src/routes/webhooks.js`, `backend/src/services/stripe.js` |
| 6 | Livraison auto (email + téléchargement + "mes achats") | `backend/src/routes/account.js`, `backend/src/services/email.js`, `backend/src/services/storage.js` |
| 7 | Licence + serveur de vérification FiveM | `backend/src/routes/fivem.js`, `backend/src/routes/api/license.js`, `backend/src/services/licenseKeys.js`, `fivem-verify-server/verify/` |
| 8 | Bots Discord (rôle auto, notifications) | (nouveau dossier à créer, ex: `backend/src/bot/`) |
| 9 | Durcissement + légal | `backend/src/middlewares/rateLimit.js`, section 9 ci-dessous |

Front d'abord (comme tu le prévois) : commence par les `.ejs` dans `backend/src/views/` + le CSS dans `frontend/src/input.css`, tu peux visualiser sans aucune route ni base fonctionnelle.

---

## 9. Volet légal (France)

- **Statut** : au minimum auto-entrepreneur pour encaisser légalement (Stripe le demande au KYC).
- **Mentions légales + CGV** obligatoires (identité vendeur, conditions de vente, renoncement explicite au délai de rétractation de 14 jours pour le contenu numérique livré immédiatement).
- **RGPD** : politique de confidentialité, suppression de compte possible — mais anonymiser les `orders` plutôt que les supprimer (obligations comptables).
- **TVA** : au-delà d'un seuil de CA en auto-entrepreneur, la TVA doit être facturée.
- **Cfx.re** : vérifie les règles de revente de ressources (système d'escrow officiel) avant de mettre des scripts en vente.

---

## 10. Documentation officielle & vidéos

### Express + EJS
- [Express — guide du routing (officiel)](https://expressjs.com/en/guide/routing.html)
- [EJS — documentation officielle](https://ejs.co/#docs)
- [DigitalOcean — How To Use EJS to Template Your Node Application](https://www.digitalocean.com/community/tutorials/how-to-use-ejs-to-template-your-node-application)

### Base de données (Prisma + MySQL)
- [Prisma — Quickstart MySQL (officiel)](https://www.prisma.io/docs/prisma-orm/quickstart/mysql)
- [Prisma — Ajouter Prisma à un projet existant avec MySQL (officiel)](https://www.prisma.io/docs/getting-started/setup-prisma/add-to-existing-project/relational-databases/querying-the-database-node-mysql)
- [Prisma — Upgrade guide v7 (config `prisma.config.*`)](https://www.prisma.io/docs/guides/upgrade-prisma-orm/v7)

### Paiement (Stripe)
- [Stripe — Checkout (officiel)](https://docs.stripe.com/payments/checkout)
- [Stripe — Webhooks (officiel)](https://docs.stripe.com/webhooks)
- [Vidéo — How to integrate Stripe Checkout with Node.js](https://www.youtube.com/watch?v=cheDHoEazPs)

### Authentification Discord
- [Discord — Documentation officielle OAuth2](https://docs.discord.com/developers/topics/oauth2)
- [passport-discord-auth — package npm (le remplaçant maintenu de passport-discord)](https://www.npmjs.com/package/passport-discord-auth)
- [Vidéo — Adding Discord OAuth2 to Your Application](https://www.youtube.com/watch?v=hnk2-Fs8JVI)

### Docker / Docker Compose
- [Docker Compose — Quickstart (officiel)](https://docs.docker.com/compose/gettingstarted/)
- [Docker Compose — Documentation (officiel)](https://docs.docker.com/compose/)

### FiveM (ressource de vérification)
- [FiveM — Natives (`GetPlayerIdentifiers`, `PerformHttpRequest`)](https://docs.fivem.net/natives/)

---

## 11. Vérification

- `docker compose up -d --build` → 4 conteneurs `running` (`docker compose ps`).
- `backend` reste "up" sans rien servir tant que `src/app.js` est vide — normal.
- `http://localhost:8080` (phpMyAdmin) → connexion avec `DB_USER`/`DB_PASSWORD` du `.env`, base `gridforge` visible (vide).
- Une fois les modèles Prisma écrits : `npx prisma generate` puis `npx prisma migrate dev --name init` (depuis `backend/`, en dehors de Docker ou via `docker compose exec backend sh`).
- Flux Discord : lier le même Discord sur deux comptes → doit être bloqué (contrainte `UNIQUE`).
- Flux FiveM : lier la même license sur deux comptes → doit être bloqué.
- Webhook Stripe en double (`stripe trigger checkout.session.completed` deux fois, même event id) → une seule commande créée.
- Token de téléchargement expiré ou épuisé → refusé.
- `/api/license/verify` avec une mauvaise `server_license` → `valid: false`.

---

**Rappel** : ce README + le squelette de fichiers sont la livraison actuelle — aucun fichier de logique (routes, services, vues) ne contient de code, seuls les fichiers de config/infra (`.env`, `Dockerfile`, `docker-compose.yml`, `schema.prisma`, `prisma.config.cjs`) sont remplis. Je n'y retoucherai plus sauf si tu me le redemandes.
