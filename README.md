# Cicoins

Guide de developpement, de deploiement et d'exploitation de l'infrastructure Cicoins.

La specification produit reste dans [`README_CICOINS.md`](./README_CICOINS.md).
La specification fonctionnelle de l'administration reste dans [`ADMIN.md`](./ADMIN.md).

## Etat d'implementation

Le depot contient la PWA React/Vite, l'API Fastify publique et privee,
PostgreSQL, Caddy, un sandbox marchand, un relais Docker vers Paie Server et un
sidecar Tailscale pour l'administration privee.

Les points suivants ne sont pas encore termines :

- le client `apps/api/src/providers/paie-server.ts` et ses identifiants sont
  raccordes, mais aucune route PAYIN utilisateur ne l'appelle encore ;
- l'ecran PWA Acheter reste volontairement indisponible ;
- le webhook Paie Server et le MINT idempotent restent a implementer ;
- l'authentification admin mot de passe + TOTP n'est pas implementee ;
  `ADMIN_BOOTSTRAP_TOKEN` est uniquement un acces transitoire de developpement.

Ne jamais crediter des Cicoins depuis un retour navigateur. Seul un webhook
provider verifie, rattache a un PAYIN local et traite de facon idempotente peut
declencher un MINT.

## Architecture

```text
Internet
   |
   | 80/443
   v
Caddy (VPS / Docker Cicoins)
   |-- PWA statique: apps/web/dist
   |-- /rt/* ----------------------> cicoins-api:3000
   |-- /provider-webhooks/* -------> cicoins-api:3000
   `-- sandbox.cicoins... ---------> cicoins-sandbox:8080

cicoins-api
   |-- PostgreSQL -----------------> postgres:5432
   |-- Orange SMS -----------------> Internet via reseau egress
   `-- Paie Server ----------------> paie-connector:8080
                                          |
                                          | paie-server-reseau
                                          v
                                 paie-server-application:3000

PC administrateur
   |-- Cicoins-Admin local: 127.0.0.1:4173
   |-- client Tailscale de la machine
   `-- HTTPS prive Tailnet --------> tailscale-vps:443
                                          |
                                          v
                                    cicoins-api:3101
```

Paie Server conserve son propre depot, son propre Compose, sa propre base et
son cycle de vie. Cicoins ne monte aucun fichier Paie Server et ne se connecte
jamais directement a sa base PostgreSQL.

## Repartition des composants

| Composant | Emplacement production | Docker | Exposition |
| --- | --- | --- | --- |
| PWA Cicoins | VPS, `apps/web/dist` | servie par Caddy | HTTPS public |
| Caddy | VPS Cicoins | oui | `80` et `443` |
| API publique | VPS Cicoins | oui | interne `3000`, via Caddy |
| API admin | VPS Cicoins | oui | interne `3101`, via Tailscale |
| PostgreSQL Cicoins | VPS Cicoins | oui | interne `5432`, aucun port hote |
| Sandbox Cicoins | VPS si necessaire | oui | via Caddy |
| Relais Paie Server | VPS Cicoins | oui | aucun port hote |
| Sidecar Tailscale | VPS Cicoins | profil `admin-access` | Tailnet `443` |
| Cicoins Admin | PC administrateur | non | `127.0.0.1:4173` |
| Client Tailscale admin | PC administrateur | non | aucune entree Internet |
| Paie Server | VPS, depot/Compose separe | oui | gere par Paie Server |

Le projet local d'administration est `../Cicoins-Admin`. Il ne doit pas etre
copie sur le VPS, ajoute au Compose Cicoins ou servi par Caddy.

## Arborescence

```text
apps/
  web/                    PWA React/Vite
  api/                    API publique et API admin Fastify
  sandbox/                sandbox marchand Cicoins
infra/
  Caddyfile               routage public
  paie-connector.conf     relais vers Paie Server
  tailscale/serve.json    HTTPS Tailnet vers l'API admin
scripts/
  backup-postgres.sh      sauvegarde PostgreSQL Docker
docker-compose.yml        infrastructure representant le VPS
.env.example              contrat de configuration sans secrets
```

## Reseaux Docker

| Reseau | Type | Membres | Role |
| --- | --- | --- | --- |
| `cicoins_public` | bridge | Caddy | entree publique |
| `cicoins_private` | bridge `internal` | Caddy, API, PostgreSQL, sandbox, relais, Tailscale | trafic interne |
| `cicoins_egress` | bridge | API, Tailscale | Orange SMS et control plane Tailscale |
| `paie-server-reseau` | bridge externe | Paie Server et `paie-connector` | contrat inter-Compose |

Seul `paie-connector` est attache a `cicoins_private` et a
`paie-server-reseau`. `cicoins-api` ne doit pas rejoindre directement le
reseau Paie Server. PostgreSQL reste uniquement sur `cicoins_private`.

`PAIE_SERVER_NETWORK` doit correspondre exactement au reseau cree par Paie Server.

## Ports

Ports Cicoins publies :

```text
80/tcp     HTTP local et redirection
443/tcp    HTTPS public
443/udp    HTTP/3 Caddy
```

Ports non publies :

```text
3000/tcp   API publique Fastify
3101/tcp   API admin Fastify
5432/tcp   PostgreSQL Cicoins
8080/tcp   sandbox et relais Paie Server
```

Sur le PC administrateur, Vite Admin ecoute exclusivement sur `127.0.0.1:4173`.

## Prerequis

- Docker Engine et Docker Compose v2 ;
- Node.js et npm ;
- `curl`, `openssl` et `gzip` pour l'exploitation ;
- le depot Paie Server separe pour tester ce provider ;
- Tailscale installe sur le PC admin.

## Configuration Cicoins

```bash
cp .env.example .env
chmod 600 .env
```

Ne jamais committer `.env`, une cle Orange, une cle Paie Server, une auth key
Tailscale, un jeton admin ou un dump PostgreSQL.

### Variables principales

| Variable | Local | Production |
| --- | --- | --- |
| `NODE_ENV` | `development` | `production` |
| `DATABASE_URL` | hostname Docker `postgres` | meme hostname, secrets forts |
| `POSTGRES_DB` | `cicoins` | nom choisi |
| `POSTGRES_USER` | `cicoins` | utilisateur dedie |
| `POSTGRES_PASSWORD` | valeur locale | secret aleatoire fort |
| `SESSION_SECRET` | secret local | au moins 32 octets aleatoires |
| `API_KEY_PEPPER` | secret local | au moins 32 octets aleatoires |
| `WEBHOOK_MASTER_KEY` | secret local | au moins 32 octets aleatoires |
| `CICOINS_PUBLIC_WEB_URL` | `http://localhost` | URL HTTPS PWA |
| `PUBLIC_WEB_ORIGIN` | origine locale | origine HTTPS exacte PWA |
| `ADMIN_ALLOWED_ORIGINS` | `http://127.0.0.1:4173` | origine locale admin |

Generer chaque secret separement :

```bash
openssl rand -hex 32
```

### Domaines Caddy

```env
CICOINS_PUBLIC_WEB_URL=https://cicoins.example.com
PUBLIC_WEB_URL=https://cicoins.example.com
PUBLIC_WEB_ORIGIN=https://cicoins.example.com
CICOINS_WEB_DOMAIN=cicoins.example.com
CICOINS_API_DOMAIN=api.cicoins.example.com
CICOINS_SANDBOX_DOMAIN=sandbox.cicoins.example.com
CADDY_EMAIL=ops@example.com
```

Les DNS `A`/`AAAA` pointent vers le VPS. Le firewall autorise 80/443. Le port
3101 ne doit jamais etre ouvert.

### Orange SMS

Developpement, sans consommation de SMS :

```env
NODE_ENV=development
OTP_MODE=development
```

Production :

```env
NODE_ENV=production
OTP_MODE=orange
ORANGE_SMS_CLIENT_ID=...
ORANGE_SMS_CLIENT_SECRET=...
ORANGE_SMS_SENDER_ADDRESS=+2250000
ORANGE_SMS_SENDER=
```

`ORANGE_SMS_SENDER` reste vide pour le nom Orange par defaut ou contient un nom
deja approuve. Le backend derive lui-meme l'en-tete Basic.

## Developpement local

Paie Server demarre avant Cicoins car il cree `paie-server-reseau` :

```bash
cd ../paie-server
docker compose up -d

cd ../Cicoins
npm ci
npm run build
docker compose up -d --build
```

La PWA construite est disponible sur `http://localhost`.

Rechargement a chaud :

```bash
npm run dev:web
```

Ouvrir `http://localhost:5173`. Vite transmet `/rt` et
`/provider-webhooks` a Caddy sur `http://127.0.0.1`. La variable
`VITE_API_PROXY_TARGET` permet de remplacer la cible.

Ne pas lancer `npm run dev:api` contre `DATABASE_URL=...@postgres:5432` depuis
l'hote : `postgres` existe seulement dans Docker. Le flux normal est Vite vers
Caddy, puis Caddy vers l'API Docker.

## Integration Paie Server

### Separation obligatoire

Le projet Paie Server reste autonome :

```text
../paie-server/docker-compose.yml
../paie-server/.env
base et volumes Paie Server
```

Cicoins ne modifie aucun de ces elements. Les cles et la configuration metier
restent gerees depuis l'interface Paie Server.

### Contrat reseau

Paie Server fournit :

```text
reseau:  paie-server-reseau
service: paie-server-application
port:    3000
sante:   GET /api/sante
```

Le relais Cicoins `http://paie-connector:8080` transmet vers
`http://paie-server-application:3000`. Il ne publie aucun port sur l'hote.

### Identifiants

Dans la configuration marchand Paie Server, relever :

- `Cle pour creer des paiements` ;
- `Cle d'administration API`.

Les recopier uniquement dans `Cicoins/.env` :

```env
PAIE_SERVER_URL=http://paie-connector:8080
PAIE_SERVER_API_KEY=<cle-de-creation-paie-server>
PAIE_SERVER_ADMIN_API_KEY=<cle-admin-paie-server>
PAIE_SERVER_NETWORK=paie-server-reseau
```

Correspondance du client Cicoins :

```text
POST /api/paiements       header x-cle-api
GET  /api/paiements/:id   header x-cle-marchand actuellement envoye
```

Ces secrets restent dans l'API. Ils ne sont jamais exposes a la PWA, au
sandbox, a Caddy ou a Cicoins Admin.

### Verification sans paiement

```bash
docker network inspect paie-server-reseau
docker compose ps paie-connector
docker compose logs --tail=100 paie-connector
docker compose exec cicoins-api node -e \
  "fetch(process.env.PAIE_SERVER_URL + '/api/sante').then(r => { console.log(r.status); process.exit(r.ok ? 0 : 1) })"
```

Le resultat attendu est `200`. Un `502`/`504` signifie generalement que Paie
Server est arrete, en pause, hors reseau ou que son nom de service a change.

### Activation metier restante

La connectivite et les cles n'activent pas encore le bouton Acheter. Il faut :

1. ajouter une route de creation de PAYIN avec idempotence ;
2. enregistrer le PAYIN `PENDING` avant l'appel externe ;
3. fournir les URL publiques de retour et webhook ;
4. verifier `x-signature-paiement` sur le corps brut ;
5. dedupliquer avec `x-id-evenement-paiement` et `provider_events` ;
6. verifier reference, montant, devise et statut ;
7. effectuer MINT et `COMPLETED` dans une transaction PostgreSQL ;
8. tester acceptation, refus, timeout, retry et double webhook.

Paie Server ne fournit pas de retrait FCFA et ne doit jamais etre utilise comme
provider de withdrawal.

## Tailscale pour Cicoins Admin

Tailscale transporte uniquement l'API admin. La PWA et Paie Server n'utilisent
pas ce tunnel.

Documentation officielle :

- [Tailscale dans Docker](https://tailscale.com/docs/features/containers/docker/docker-params)
- [MagicDNS](https://tailscale.com/docs/features/magicdns)
- [Certificats HTTPS](https://tailscale.com/docs/how-to/set-up-https-certificates)
- [Grants](https://tailscale.com/docs/features/access-control/grants)

### 1. Politique Tailnet

Exemple a adapter avec le compte Tailscale de l'administrateur :

```json
{
  "groups": {
    "group:cicoins-admins": ["admin@example.com"]
  },
  "tagOwners": {
    "tag:cicoins-vps": ["autogroup:admin"]
  },
  "grants": [
    {
      "src": ["group:cicoins-admins"],
      "dst": ["tag:cicoins-vps"],
      "ip": ["tcp:443"]
    }
  ]
}
```

Ne pas conserver une politique `allow all` en production.

### 2. MagicDNS et HTTPS

Dans la console Tailscale :

1. activer MagicDNS ;
2. activer HTTPS Certificates ;
3. choisir un hostname sans information sensible, car le nom du certificat est
   publie dans les journaux publics de certificats.

### 3. Enroler le sidecar VPS

Creer une auth key taguee `tag:cicoins-vps`, non ephemere et de preference a
usage unique :

```env
TAILSCALE_VPS_AUTHKEY=<auth-key-tailscale>
TAILSCALE_VPS_HOSTNAME=cicoins-vps
TAILSCALE_VPS_TAG=tag:cicoins-vps
```

```bash
docker compose --profile admin-access up -d tailscale-vps
docker compose exec tailscale-vps tailscale status
docker compose exec tailscale-vps tailscale serve status
```

`TS_AUTH_ONCE=true` et le volume `tailscale_vps_state` conservent l'identite.
Une fois l'enrolement confirme, retirer l'auth key du `.env`. Ne jamais activer
Tailscale Funnel. `infra/tailscale/serve.json` sert HTTPS uniquement dans le
Tailnet et transmet vers `cicoins-api:3101`.

### 4. Configurer le PC admin

Installer Tailscale directement sur le PC et se connecter avec le compte du
groupe autorise :

```bash
tailscale status
tailscale ping cicoins-vps
```

Dans `../Cicoins-Admin/.env` :

```env
ADMIN_API_TARGET=https://cicoins-vps.nom-du-tailnet.ts.net
ADMIN_API_TLS_VERIFY=true
ADMIN_API_TOKEN=meme_valeur_que_ADMIN_BOOTSTRAP_TOKEN
```

Dans `Cicoins/.env` :

```env
ADMIN_BOOTSTRAP_TOKEN=un_secret_aleatoire_long
ADMIN_ALLOWED_ORIGINS=http://127.0.0.1:4173,http://localhost:4173
```

Puis, uniquement sur le PC :

```bash
cd ../Cicoins-Admin
npm ci
npm run dev
```

Ouvrir `http://127.0.0.1:4173`.

Le jeton bootstrap ne remplace pas l'authentification finale. Avant production,
implementer mot de passe Argon2, TOTP, sessions courtes, revocation et audit.

## Deploiement VPS

### A deployer sur le serveur

- le depot Cicoins, sans `Cicoins-Admin` ;
- Node/npm pour construire la PWA, sauf si le build vient de la CI ;
- Docker Engine et Compose ;
- le Compose Cicoins et le sidecar Tailscale ;
- le depot/Compose Paie Server separe ;
- les volumes PostgreSQL, Caddy et Tailscale ;
- un firewall autorisant 80/443 et un SSH restreint.

### A laisser sur le PC administrateur

- `Cicoins-Admin` ;
- Node/npm pour Vite Admin ;
- le client Tailscale ;
- `Cicoins-Admin/.env` ;
- les outils d'exploitation autorises.

### Ordre de deploiement

```bash
cd /srv/paie-server
docker compose up -d

cd /srv/cicoins
npm ci
npm run build
docker compose --profile admin-access up -d --build
```

Verification :

```bash
docker compose ps
curl -fsS -o /dev/null https://cicoins.example.com/
curl -fsS https://api.cicoins.example.com/health
docker compose exec cicoins-api node -e \
  "fetch(process.env.PAIE_SERVER_URL + '/api/sante').then(r => { console.log(r.status); process.exit(r.ok ? 0 : 1) })"
docker compose exec tailscale-vps tailscale status
```

## Migrations PostgreSQL

Les migrations sont montees dans `/docker-entrypoint-initdb.d`. PostgreSQL ne
les execute automatiquement que lors de la creation d'un volume vide. Une base
existante n'est pas migree par un rebuild.

Sauvegarder, lire la migration, puis l'appliquer explicitement :

```bash
docker compose exec -T postgres psql \
  -U cicoins -d cicoins -v ON_ERROR_STOP=1 \
  < apps/api/migrations/003_otp_invalidation.sql
```

Adapter le fichier et les variables. Ne jamais appliquer une migration inconnue
directement en production.

## Sauvegarde et restauration

```bash
./scripts/backup-postgres.sh
```

Les sauvegardes vont dans `./backups/postgres` par defaut. Les copier vers un
stockage chiffre hors VPS avec retention.

Restauration sur une base explicitement preparee :

```bash
gunzip -c backups/postgres/cicoins-YYYYMMDDTHHMMSSZ.sql.gz | \
  docker compose exec -T postgres sh -c \
  'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" "$POSTGRES_DB"'
```

Tester regulierement une restauration complete.

## Exploitation

```bash
# Demarrer
docker compose --profile admin-access up -d --build

# Etat
docker compose --profile admin-access ps

# Logs
docker compose logs -f cicoins-api
docker compose logs -f caddy
docker compose logs -f postgres
docker compose logs -f paie-connector
docker compose logs -f tailscale-vps

# Pause sans suppression
docker compose --profile admin-access pause
docker compose --profile admin-access unpause

# Arret sans suppression des volumes
docker compose --profile admin-access down
```

Ne pas utiliser `down -v` en production : cette option supprime les volumes.

## Diagnostic

API et base :

```bash
curl -k -i https://api.localhost/health
docker compose logs --tail=100 cicoins-api
docker compose exec postgres pg_isready -U cicoins -d cicoins
```

Paie Server :

```bash
docker compose ps paie-connector
docker compose exec paie-connector getent hosts paie-server-application
docker compose exec paie-connector wget -qO- \
  http://paie-server-application:3000/api/sante
```

Tailscale :

```bash
docker compose --profile admin-access ps tailscale-vps
docker compose exec tailscale-vps tailscale status
docker compose exec tailscale-vps tailscale serve status
```

`NeedsLogin` exige une nouvelle auth key. Un `502` dans Cicoins Admin signifie
generalement que le sidecar ne joint pas `cicoins-api:3101`.

## Securite avant production

- remplacer toutes les valeurs `replace_me` et tous les secrets de test ;
- utiliser `NODE_ENV=production` et `OTP_MODE=orange` ;
- publier uniquement Caddy en 80/443 ;
- garder PostgreSQL, Fastify et le relais sans ports hote ;
- appliquer une politique Tailscale deny-by-default ;
- ne jamais activer Funnel pour l'admin ;
- implementer l'auth admin mot de passe + TOTP ;
- verifier cryptographiquement les webhooks provider ;
- conserver les cles Paie Server et Orange uniquement dans l'API ;
- mettre en place sauvegardes chiffrees, restauration et supervision ;
- ne jamais initialiser la production avec des donnees fictives.

## Routes

Le contrat public utilise le prefixe technique `/rt` :

```text
POST /rt/auth/otp
POST /rt/auth/users
POST /rt/auth/session
GET  /rt/me/overview
POST /rt/me/transfers
POST /rt/payments
GET  /rt/payments/:id
GET  /rt/checkout/:id
POST /rt/checkout/:id/confirm
GET  /rt/wallet
POST /rt/payouts
GET  /rt/payout-claims/:token
POST /rt/payout-claims/:token/confirm
```

Les routes `/internal/admin/*` ne sont jamais transmises par les domaines
publics Caddy. Elles passent uniquement par le listener `3101` et Tailscale.
