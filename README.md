# Démonstrateur MidPoint + Keycloak

Ce dépôt propose un environnement Docker Compose pour monter rapidement un POC combinant **Evolveum MidPoint**, **Keycloak** et une **application web de test** connectée à Keycloak.

## Prérequis

- Docker et Docker Compose (v2+) installés sur votre machine.
- Au moins 6 Go de RAM libres pour exécuter simultanément MidPoint et Keycloak.

### Installer Docker & Docker Compose

Si vous ne disposez pas encore de Docker Desktop (Windows/macOS) ou du moteur Docker sur Linux, suivez les indications
ci-dessous. Les instructions Docker officielles restent la référence : https://docs.docker.com/engine/install/

#### Ubuntu / Debian

```bash
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release && echo "$ID")/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/$(. /etc/os-release && echo "$ID") \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker "$USER"    # se déconnecter/reconnecter pour prendre en compte l'ajout au groupe
```

#### Fedora / CentOS / RHEL

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

#### Windows / macOS

- Téléchargez Docker Desktop depuis https://www.docker.com/products/docker-desktop/
- Installez-le en acceptant l’activation de la commande `docker compose` dans votre terminal.
- Après l’installation, démarrez Docker Desktop avant de lancer la pile `docker compose` du projet.

## Démarrage rapide

```bash
docker compose up -d
```

Les services exposent les ports suivants sur votre machine :

| Service                | URL                                |
| ---------------------- | ---------------------------------- |
| MidPoint               | http://localhost:8080/midpoint     |
| Console d'administration Keycloak | http://localhost:8081    |
| Application de test Keycloak      | http://localhost:8082    |

### Comptes par défaut

| Usage                        | Identifiant      | Mot de passe    |
| --------------------------- | ---------------- | --------------- |
| Administrateur Keycloak     | `admin`          | `admin`         |
| Utilisateur de démo Keycloak| `alice`          | `ChangeMe1!`    |

> ⚠️ Ces identifiants sont destinés à une démonstration locale. Pensez à les modifier avant toute exposition hors laboratoire.

## Composition des services

- **midpoint-db** : base Postgres utilisée par MidPoint (avec un _healthcheck_ `pg_isready`).
- **midpoint** : image officielle `evolveum/midpoint` configurée pour utiliser la base Postgres.
- **keycloak-db** : base Postgres pour Keycloak (avec un _healthcheck_ `pg_isready`).
- **keycloak** : serveur Keycloak démarré en mode développement avec import automatique du realm `midpoint-demo`.
- **keycloak-test-app** : page HTML statique qui démontre un login OIDC via Keycloak.

Les services MidPoint et Keycloak attendent désormais que leurs bases respectives soient déclarées « healthy » avant de démarrer, ce qui évite les erreurs de connexion lors du premier lancement.

Tous les services partagent le réseau Docker `iam` pour simplifier la découverte entre conteneurs.

## Intégration MidPoint ↔ Keycloak

Le realm Keycloak importé contient un client confidentiel `midpoint` (secret `midpoint-secret`) pour préparer l'intégration avec MidPoint. Du côté MidPoint :

1. Connectez-vous à http://localhost:8080/midpoint avec l'utilisateur `administrator` (mot de passe initial `5ecr3t`).
2. Activez l'assistant de configuration OIDC (Administration → Système → Paramètres → Authentification → OIDC) en utilisant :
   - **Issuer** : `http://keycloak:8080/realms/midpoint-demo`
   - **Client ID** : `midpoint`
   - **Client Secret** : `midpoint-secret`
   - **Redirect URL** : `http://localhost:8080/midpoint/oidc/callback`
3. Enregistrez puis testez la connexion. MidPoint est maintenant capable de déléguer l'authentification à Keycloak.

Pour la partie provisioning, vous pouvez importer le connecteur Keycloak (basé sur REST) fourni par MidPoint et définir une Ressource en pointant vers l'API Keycloak (`http://keycloak:8080`). Cette étape dépendra de votre scénario précis, mais les bases (serveurs, bases de données et clients) sont déjà disponibles.

## Application de test Keycloak

L'application accessible sur http://localhost:8082 utilise le script `keycloak.js` fourni par Keycloak pour :

- déclencher un login (`Se connecter`),
- rafraîchir automatiquement le token,
- afficher le contenu décodé du token JWT.

Elle utilise le client public `test-app` défini dans le realm `midpoint-demo`.

## Gestion des données

- Les bases de données Postgres sont persistées via les volumes Docker `midpoint-db-data` et `keycloak-db-data`.
- Le répertoire `midpoint/` monté dans le conteneur permet de conserver la configuration MidPoint (`config.xml`) et tout fichier généré dans `midpoint-home`.

Pour repartir d'un état vierge :

```bash
docker compose down -v
```

## Personnalisation

- Modifiez `keycloak/realm-export.json` pour ajouter des utilisateurs, rôles ou clients.
- Ajustez `midpoint/config.xml` si vous souhaitez utiliser un autre SGBD ou modifier la journalisation.
- Remplacez le contenu du dossier `test-app/` par votre propre application front si besoin.

## Ressources utiles

- [Documentation MidPoint](https://docs.evolveum.com/midpoint/)
- [Documentation Keycloak](https://www.keycloak.org/documentation)

Bon POC !

## Publication sur GitHub

Si vous souhaitez publier ce POC sur votre propre dépôt GitHub, ajoutez un dépôt distant puis poussez la branche locale :

```bash
git remote add origin git@github.com:<votre-compte>/<votre-repo>.git
git push -u origin work
```

Adaptez bien sûr l'URL au dépôt que vous avez créé au préalable sur GitHub. Une fois la commande `git push` exécutée, le
code et la configuration Docker seront disponibles en ligne pour les collaborateurs de votre choix.
