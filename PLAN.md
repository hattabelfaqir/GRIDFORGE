# Plan complet — Boutique FiveM (bases, scripts, bots Discord)

## Contexte

Le repo `GRIDFORGE` est actuellement vide (juste un `git init`). L'objectif n'est **pas** que je code le site — c'est explicitement demandé de ne pas le faire, sauf pour corriger un bug ponctuel plus tard. Ce document est donc **le livrable final** : un dossier de spécification technique complet (archi, stack, modèle de données, flux de paiement/livraison, sécurité, feuille de route) que tu pourras suivre pour coder toi-même la boutique.

Le site doit vendre des bases FiveM, des bots Discord liés à FiveM, et des configurations Discord. Les points durs identifiés :
- Compte classique email/mot de passe, **mais** obligation de lier un compte Discord avant tout achat, et ce Discord doit être unique (un seul compte site par Discord).
- Livraison automatique après paiement (fichier à télécharger + accès depuis le compte).
- Verrouillage des scripts vendus à la license FiveM du serveur acheteur (mesure anti-partage standard dans l'écosystème FiveM payant).

Décisions déjà validées avec toi :
- **Stack** : Node.js + Express + rendu serveur (EJS) + JS vanilla côté client — pas de framework front lourd.
- **Identité** : Discord OAuth obligatoire pour le login/l'unicité de compte, **+** capture de la license FiveM du serveur client pour verrouiller les scripts livrés.
- **Paiement** : Stripe (Checkout hébergé, pas de formulaire carte custom).
- **Hébergement** : Vercel (site) + base de données managée Postgres (Neon ou Supabase).

---

## ⚠️ Point d'architecture important : la "vérification FiveM" ne peut pas tourner sur Vercel

Vercel exécute du code **serverless** (sans état, sans processus qui tourne en continu). Il ne peut pas héberger un serveur de jeu FXServer. Donc :

- Le **site web** (front + API Express + webhooks Stripe) → Vercel. Fonctionne très bien en serverless du moment que rien ne dépend d'un état en mémoire (sessions stockées en base, pas en RAM — voir plus bas).
- Le **serveur de vérification FiveM** (qui capture la license du joueur) doit tourner sur un vrai serveur qui reste allumé → ton VPS existant (celui qui fait déjà tourner ton serveur FiveM RP) est l'endroit naturel pour ça. Ce n'est pas un "vrai" serveur RP, juste une petite ressource FXServer minimaliste dédiée à la capture d'identifiant, qui appelle ton API Vercel en HTTPS.

C'est le seul bout d'infra qui sort de Vercel — tout le reste (site, base, paiement, emails, fichiers) reste dans l'écosystème serverless que tu as choisi.

★ Insight ─────────────────────────────────────
Il n'existe **aucun bouton officiel "Se connecter avec FiveM"** utilisable par un site tiers — Cfx.re (l'éditeur de FiveM) ne propose pas d'OAuth public comme Discord ou Google. Toute boutique FiveM qui prétend "connecter ton compte FiveM" fait en réalité l'une de ces deux choses en coulisses : (1) utiliser Discord comme identité de fait, car toute la communauté FiveM vit sur Discord, ou (2) capturer la `license` identifier (un hash stable lié au compte Rockstar/Cfx du joueur, récupérable seulement depuis un serveur FXServer via la native `GetPlayerIdentifiers`). C'est pour ça qu'on combine les deux : Discord pour l'identité "compte", license FiveM pour le verrouillage anti-partage des scripts.
─────────────────────────────────────────────────

---

## 1. Stack technique détaillée

| Brique | Choix | Pourquoi |
|---|---|---|
| Runtime / serveur | Node.js + Express | Choisi par toi, proche de ce que tu connais déjà (JS). |
| Rendu des pages | EJS (templates HTML + JS server-side) | Pas besoin d'apprendre React ; tu gardes HTML/CSS/JS classiques. |
| Base de données | PostgreSQL (Neon ou Supabase, plan gratuit) | Managé = pas de sysadmin DB à gérer. |
| Accès BDD | Prisma ORM | Évite les erreurs SQL manuelles, migrations versionnées, autocomplétion — bon compromis pour un débutant. |
| Auth email/mdp | `bcrypt` (hash) + `express-session` | Standard, bien documenté. |
| Stockage session | `connect-pg-simple` (sessions en Postgres, pas en RAM) | **Obligatoire sur Vercel** : le serverless ne garde pas de mémoire entre les requêtes. |
| Login Discord | `passport-discord` (stratégie Passport.js) | OAuth Discord officiel, très simple à brancher. |
| Paiement | Stripe Checkout + Webhooks | Page de paiement hébergée par Stripe = zéro formulaire carte à sécuriser toi-même. |
| Email transactionnel | Resend (ou Postmark) | API simple, gratuit jusqu'à un bon volume, bonne délivrabilité (important pour ne pas finir en spam). |
| Stockage des fichiers vendus | Vercel Blob ou Supabase Storage | Pas de gros fichiers en pièce jointe email — juste des liens signés temporaires. |
| Validation des entrées | `zod` | Empêche les données mal formées d'atteindre la base. |
| Sécurité HTTP | `helmet`, `express-rate-limit`, `csurf` (ou double-submit CSRF token) | Voir section Sécurité. |
| Bots Discord | `discord.js` | Pour la livraison de rôle auto / notifications d'achat côté Discord. |

---

## 2. Modèle de données (schéma logique)

```
users
  id, email (unique), password_hash, created_at
  discord_id (unique, nullable jusqu'à liaison)
  discord_username, discord_avatar

fivem_licenses
  id, user_id (FK), license_identifier (unique), server_label, linked_at
  -- unique sur license_identifier => un serveur FiveM ne peut être lié qu'à un seul compte

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
  -- empêche de traiter deux fois le même webhook Stripe (idempotence)
```

Contraintes clés à mettre en base (pas juste côté code) :
- `users.discord_id` → `UNIQUE` → empêche de relier deux comptes site au même Discord.
- `fivem_licenses.license_identifier` → `UNIQUE` → empêche de relier la même license FiveM à deux comptes.
- `orders.stripe_checkout_session_id` → `UNIQUE` + `webhook_events_log.stripe_event_id` → `UNIQUE` → empêche la double-livraison si Stripe renvoie le même webhook deux fois (ça arrive, c'est normal côté Stripe).

---

## 3. Flux détaillés

### 3.1 Inscription / connexion classique
1. Formulaire email + mot de passe → hash avec `bcrypt` (cost factor 12) → ligne `users`.
2. Email de vérification (recommandé, évite les faux comptes) via Resend.
3. Connexion → session créée, cookie `httpOnly; secure; sameSite=lax`, stockée en Postgres via `connect-pg-simple`.

### 3.2 Liaison Discord (obligatoire avant achat)
1. Bouton "Lier mon Discord" → redirection OAuth Discord (`passport-discord`).
2. Callback : Discord renvoie l'`id` Discord de l'utilisateur.
3. Avant d'écrire en base : vérifier qu'aucun autre `users` n'a déjà ce `discord_id`. Si déjà pris → message clair ("ce Discord est déjà lié à un autre compte") plutôt qu'une erreur SQL brute.
4. Middleware `requireDiscordLinked` sur toutes les routes d'achat — redirige vers la page de liaison si `discord_id` est null.

### 3.3 Liaison de la license FiveM (pour les produits "script")
1. Le client doit indiquer quel serveur FiveM va utiliser le script.
2. Deux options possibles à lui proposer (tu choisis laquelle exposer, ou les deux) :
   - **Option simple** : il colle lui-même sa license key de serveur (visible sur `keymaster.fivem.net`, ex: `cfxk_XXXX...`), tu la stockes telle quelle. Facile à coder, mais déclaratif — rien n'empêche l'utilisateur d'en mettre une fausse au moment de l'achat (le vrai contrôle se fait ensuite à l'exécution, voir 3.5).
   - **Option robuste** : il rejoint le petit serveur FXServer de vérification (celui qui tourne sur ton VPS). La ressource capture son identifiant `license` via `GetPlayerIdentifiers` et l'envoie à ton API (`POST /api/fivem/verify`, signé avec un secret partagé) qui l'associe à son compte connecté. Plus fiable car ça vient du serveur de jeu et pas d'un champ texte.
3. Contrainte `UNIQUE` en base empêche la réutilisation sur plusieurs comptes.

### 3.4 Achat (Stripe Checkout)
1. `POST /checkout/:productSlug` (utilisateur connecté + Discord lié) → crée une session Stripe Checkout côté serveur avec `metadata: { user_id, product_id }` → redirige le client vers la page Stripe.
2. Stripe redirige vers une page `/merci` après paiement.
3. **La livraison ne se fait jamais depuis cette redirection** (elle n'est pas fiable — l'utilisateur peut fermer l'onglet avant). Elle se fait uniquement via le webhook.
4. `POST /webhooks/stripe` reçoit `checkout.session.completed` :
   - Vérifier la signature Stripe (`stripe-signature` header + secret webhook).
   - Vérifier que `stripe_event_id` n'a pas déjà été traité (table `webhook_events_log`) → idempotence.
   - Créer `orders` + `order_items`.
   - Générer une `license_key` par item si le produit est un script.
   - Générer un `download_token` signé, à usage limité (ex: 5 téléchargements, expire sous 7 jours).
   - Envoyer l'email de livraison (lien de téléchargement + clé de licence) via Resend.
   - Débloquer l'accès dans l'espace "Mes achats" du compte (c'est le canal de livraison principal — l'email est une sauvegarde).

### 3.5 Téléchargement sécurisé
- `GET /download/:token` → vérifie que le token existe, n'est pas expiré, `use_count < max_uses`, et correspond bien à un achat de l'utilisateur connecté (double vérification : token + session).
- Incrémente `use_count`, stream le fichier depuis Vercel Blob / Supabase Storage.
- Jamais de fichier livré en pièce jointe email (limite de taille + pas de lien fiable derrière).

### 3.6 Vérification anti-partage côté script FiveM livré
- Le script vendu contient un petit appel réseau au démarrage (`fetch`/`PerformHttpRequest` en Lua) vers `POST /api/license/verify` avec `{ key_value, server_license }`.
- Ton API répond `valid: true/false`. Si `false`, le script s'arrête (`return`) au lieu de charger — pattern standard dans l'écosystème des scripts FiveM payants.
- Logger chaque vérification (IP, timestamp) pour repérer un usage anormal (même clé utilisée depuis trop de licenses différentes = signe de clé fuitée/partagée).

---

## 4. Sécurité — check-list à respecter dès le départ

- **Mots de passe** : `bcrypt`, jamais de SHA1/MD5, jamais en clair dans les logs.
- **Sessions** : cookies `httpOnly`, `secure` (HTTPS uniquement), `sameSite=lax`, stockage en Postgres (pas en mémoire — obligatoire en serverless).
- **CSRF** : token anti-CSRF sur tous les formulaires qui changent un état (achat exclu, car géré par Stripe côté redirection ; mais profil, liaison Discord/FiveM, etc. oui).
- **Rate limiting** : `express-rate-limit` sur `/login`, `/register`, `/api/license/verify`, `/api/fivem/verify` — évite le bruteforce et le spam de vérification.
- **Webhooks** : toujours vérifier la signature Stripe (`stripe.webhooks.constructEvent`) — sans ça, n'importe qui peut simuler un paiement réussi.
- **Secrets** : `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `DISCORD_CLIENT_SECRET`, `FIVEM_VERIFY_SHARED_SECRET`, `SESSION_SECRET`, `DATABASE_URL` → variables d'environnement Vercel, jamais commit dans le repo.
- **Validation d'entrée** : `zod` sur chaque body de requête (register, checkout, liaison license) pour rejeter les payloads malformés avant qu'ils touchent la base.
- **Headers** : `helmet()` en middleware global.
- **Téléchargements** : tokens signés + expirants + limités en usage (voir 3.5), jamais d'URL de fichier prévisible ou permanente.

---

## 5. Arborescence de projet proposée

```
/
├── prisma/
│   └── schema.prisma           # modèle de données (section 2)
├── src/
│   ├── app.js                  # setup Express, middlewares globaux
│   ├── routes/
│   │   ├── auth.js             # register/login/logout
│   │   ├── discord.js          # OAuth Discord (liaison)
│   │   ├── fivem.js            # liaison + vérification license
│   │   ├── shop.js             # catalogue, page produit
│   │   ├── checkout.js         # création session Stripe
│   │   ├── webhooks.js         # /webhooks/stripe
│   │   ├── account.js          # "mes achats", téléchargements
│   │   └── api/
│   │       └── license.js      # /api/license/verify (appelé par les scripts FiveM vendus)
│   ├── middlewares/
│   │   ├── requireAuth.js
│   │   ├── requireDiscordLinked.js
│   │   └── rateLimit.js
│   ├── services/
│   │   ├── stripe.js
│   │   ├── email.js            # wrapper Resend
│   │   ├── storage.js          # wrapper Blob/Supabase Storage
│   │   └── licenseKeys.js      # génération/validation des clés
│   └── views/                  # templates EJS
│       ├── shop/…
│       ├── account/…
│       └── auth/…
├── fivem-verify-server/         # ressource FXServer séparée, déployée sur le VPS (pas sur Vercel)
│   └── verify/                 # ressource Lua qui capture la license et POST vers l'API
├── public/                     # CSS/JS/images statiques
├── .env.example
└── package.json
```

---

## 6. Feuille de route (ordre de développement conseillé)

1. **Setup** — repo, `package.json`, Prisma + Postgres (Neon/Supabase), déploiement Vercel "hello world" pour valider la chaîne CI/CD dès le début.
2. **Auth email/mot de passe** — register, login, logout, sessions en base.
3. **Liaison Discord obligatoire** — OAuth, contrainte d'unicité, middleware `requireDiscordLinked`.
4. **Catalogue produits** — table `products`, pages listing/détail (admin peut être un simple script CLI ou une route protégée pour ajouter des produits au début, pas besoin d'un back-office complet tout de suite).
5. **Paiement Stripe** — Checkout + webhook + `orders`/`order_items`, idempotence.
6. **Livraison automatique** — email de confirmation, espace "mes achats", téléchargement sécurisé par token.
7. **Système de licence + serveur de vérification FiveM** — capture de license, génération/validation de clés, endpoint `/api/license/verify`.
8. **Bots Discord** — attribution de rôle automatique à l'achat, notifications, commandes de support liées au compte.
9. **Durcissement + légal** — rate limiting partout, CGV/mentions légales/RGPD (section 7), tests des flux critiques (paiement, double-webhook, unicité Discord/license).

---

## 7. Volet légal (France) — à ne pas zapper

- **Statut** : pour encaisser de l'argent légalement (Stripe le demandera au moment du KYC), il faut au minimum un statut auto-entrepreneur.
- **Mentions légales + CGV** obligatoires sur un site marchand français (identité du vendeur, conditions de vente, droit de rétractation — attention : pour du contenu numérique livré immédiatement, la loi permet de faire renoncer le client au droit de rétractation de 14 jours *s'il le confirme explicitement à l'achat*, sinon il pourrait légalement demander un remboursement après avoir déjà téléchargé le fichier).
- **RGPD** : tu stockes emails, mots de passe (hashés), IDs Discord, IPs (dans les logs de vérification) → il faut une politique de confidentialité, et un moyen de suppression de compte. Attention à la tension avec la compta : les factures/commandes doivent être gardées un certain nombre d'années même si le compte utilisateur est supprimé — anonymiser plutôt que supprimer les lignes `orders`.
- **TVA** : au-delà d'un certain seuil de chiffre d'affaires en auto-entrepreneur, la TVA doit être facturée — Stripe Tax peut automatiser ce calcul si besoin.
- **Écosystème FiveM/Cfx.re** : Cfx.re a ses propres règles sur la revente de ressources (notamment le système d'escrow officiel). Vérifie que les scripts que tu vends respectent les conditions d'utilisation de Cfx.re avant de les mettre en vente (au-delà du strict aspect technique de ce plan).

---

## 8. Documentation officielle à consulter par brique

- Express : https://expressjs.com/
- Prisma : https://www.prisma.io/docs
- Passport Discord : https://www.passportjs.org/packages/passport-discord/
- Stripe Checkout + Webhooks : https://docs.stripe.com/payments/checkout et https://docs.stripe.com/webhooks
- Resend : https://resend.com/docs
- FiveM natives (`GetPlayerIdentifiers`, `PerformHttpRequest`) : https://docs.fivem.net/natives/
- Vercel + Express (déploiement serverless) : https://vercel.com/docs

---

## Vérification / comment tester une fois codé

- Flux Discord : tenter de lier le même Discord sur deux comptes différents → doit être bloqué.
- Flux FiveM : tenter de lier la même license sur deux comptes différents → doit être bloqué.
- Paiement : déclencher un webhook Stripe en double (Stripe CLI `stripe trigger checkout.session.completed` deux fois avec le même event id) → une seule commande créée.
- Téléchargement : un token expiré ou déjà utilisé `max_uses` fois → doit être refusé.
- Licence script : appeler `/api/license/verify` avec une mauvaise `server_license` → doit renvoyer `valid: false`.

---

**Rappel** : conformément à ta demande, ce document est la livraison — je ne code rien à partir de ce plan sauf si tu me le redemandes explicitement (par exemple pour corriger un bug une fois que tu auras commencé à implémenter).
