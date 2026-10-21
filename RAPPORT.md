# Rapport — TP2 : Coolify et Forgejo

## 1. Environnement
- VM Ubuntu 24.04 `admin-ansible`, IP **192.168.190.155** (3,2 Go de RAM, 19 Go de disque).
- Coolify v4.4.7 : http://192.168.190.155:8000 — proxy Traefik sur le port 80.
- Forgejo 11 en conteneur : http://192.168.190.155:3001 (SSH 2222).
- Nom d'hôte local `portfolio.coolify.local` déclaré dans le fichier `hosts` de Windows.
- Application : portfolio Next.js du TP1, dépôt Forgejo `soulayma/portfolio`, build **Nixpacks**, port exposé 3000.

## 2. Déroulement
1. Installation de Coolify par le script officiel ; compte admin ; serveur `localhost` (« This machine ») connecté ; proxy démarré.
2. Forgejo lancé par `docker compose` (port 3001 pour ne pas entrer en conflit avec Next.js) avec
   `ALLOWED_HOST_LIST=external,private`, indispensable pour qu'il accepte d'envoyer un webhook vers une IP privée.
3. Dépôt public `portfolio` créé et alimenté par `git push`.
4. Ressource Coolify *Public Git Repository* pointant vers l'URL Forgejo **par IP** (Coolify ne connaît pas le fichier hosts du poste),
   Nixpacks, port 3000, domaine `http://portfolio.coolify.local`. Premier build : environ 10 minutes.
5. Webhook Forgejo → Coolify (section Gitea) : réponse 200 « Deployment queued. » ; chaque `git push` déclenche un
   déploiement (source *Webhook*) d'environ 1 minute grâce au cache.
6. Rollback : un commit cassant le build donne un déploiement *Failed*, mais l'ancien conteneur reste en service ;
   le dépôt est corrigé par un commit qui restaure la version stable.

## 3. Tableau comparatif : PaaS (Coolify, TP2) vs IaC (Ansible + Semaphore, TP1)

| Critère | TP1 — Ansible + Semaphore (IaC) | TP2 — Coolify + Forgejo (PaaS) |
|---|---|---|
| Philosophie | On décrit l'état du serveur (rôles, tâches) | On décrit l'application ; la plateforme gère le reste |
| Mise en place | Écrire 4 rôles, inventaire, variables, template Nginx | Installer Coolify puis remplir des formulaires |
| Exécution de l'app | Directement sur l'hôte avec PM2 | Dans un conteneur Docker construit par Nixpacks |
| Reverse proxy | Nginx configuré par template Jinja2 | Traefik intégré, routage par nom d'hôte automatique |
| Déclenchement | Clic dans Semaphore (ou CLI) | Automatique à chaque `git push` (webhook) |
| Idempotence | À garantir soi-même (`when`, `changed_when`, handlers) | Assurée par la plateforme (nouvelle image, remplacement du conteneur) |
| Échec / rollback | Redéployer un tag ou un commit via `app_version` | Build en échec sans coupure ; retour à un déploiement précédent |
| Contrôle / flexibilité | Total : n'importe quelle configuration système | Limité à ce que propose la plateforme |
| Traçabilité | Toute la configuration est du code versionné | Configuration stockée dans la base de Coolify (moins « as code ») |
| Courbe d'apprentissage | Plus forte (YAML, modules, Jinja2) | Faible (interface web) |
| Ressources | Légères (pas de Docker obligatoire) | Plus lourdes (Docker, Traefik, PostgreSQL, Redis, Coolify) |
| Cas d'usage idéal | Parc de serveurs, configuration système, conformité | Déployer rapidement des applications web |

**Conclusion :** Ansible offre une maîtrise fine et reproductible de l'infrastructure ; Coolify offre une expérience
« push to deploy » très rapide. Les deux sont complémentaires : Ansible peut provisionner la VM et installer Coolify,
Coolify déploie ensuite les applications.

## 4. Difficultés rencontrées
- **Pas de VM vierge disponible** : la VM cible du TP1 utilisait déjà le port 80 (Nginx, K3s). Coolify a donc été installé
  sur `admin-ansible`, où les ports 80, 8000 et 3001 étaient libres.
- **Espace disque** : l'installeur demande 20 Go libres (9 Go disponibles) ; l'installation a tout de même abouti.
- **Création du compte Coolify** : mot de passe refusé (symbole obligatoire), puis erreur 429 (trop d'essais) résolue par
  `php artisan cache:clear` dans le conteneur Coolify.
- **Domaine** : le formulaire « Add domain » sépare protocole, domaine, port et chemin. Saisir `http://` dans le champ domaine ou
  `3000` dans le port a produit des URL invalides (`https://…/3000`, ajout de `www`). Solution : protocole `http`, domaine seul,
  port et chemin vides, redirection HTTP→HTTPS désactivée. L'alerte « DNS mismatch » est normale pour un nom en `.local`.
- **Webhook refusé** (« No matching application or invalid signature ») : le secret était mal enregistré côté Forgejo.
  Diagnostic : vérification de la configuration dans la base de Coolify, puis envoi d'une requête signée avec `curl` qui
  a été acceptée. Le webhook a fonctionné une fois recréé en type **Gitea**, en `application/json`, avec le secret saisi à la main.
- **URL de webhook** : Coolify proposait une URL avec l'IP publique de la box ; elle a été remplacée par l'IP de la VM.
