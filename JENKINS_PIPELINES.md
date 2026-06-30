# Guide Jenkins TP4 - Backend et Frontend

Ce document décrit les étapes à effectuer dans l'interface Jenkins pour créer et vérifier les pipelines CI/CD du backend et du frontend.

Instances utilisées :

- Jenkins : `https://jenkins.cicd.kits.ext.educentre.fr/`
- SonarQube : `https://sonarqube.cicd.kits.ext.educentre.fr`

Repositories :

- Backend : `https://github.com/liamor2/cicd-tasklist-backend`
- Frontend : `https://github.com/liamor2/cicd-tasklist-frontend`

Images DockerHub :

- Backend : `liamor2/efrei-pro-pipepline-tp4-backend`
- Frontend : `liamor2/efrei-pro-pipepline-tp4-frontend`

## Pré-requis Jenkins

Les agents Jenkins doivent disposer de :

- Git ;
- Node.js et npm ;
- Docker ;
- Docker Compose ;
- accès au socket Docker si les builds Docker sont exécutés depuis l'agent.

Les plugins Jenkins utiles sont :

- Pipeline ;
- Git ;
- GitHub ;
- Credentials Binding ;
- JUnit ;
- Workspace Cleanup.

## Credentials Jenkins

Vérifier que les credentials suivants existent dans `Manage Jenkins` > `Credentials`.

### DockerHub

Type : `Username with password`

Valeurs :

- ID : `liamor2-dockerhub-password`
- Username : `liamor2`
- Password : token DockerHub ou mot de passe DockerHub

Ce credential permet à Jenkins de pousser les images Docker vers DockerHub.

### SonarQube

Type : `Secret text`

Valeurs :

- ID : `liamor2-sonar-token`
- Secret : token SonarQube

Ce credential permet à Jenkins d'exécuter l'analyse SonarQube sans exposer le token dans le dépôt Git.

## Pipeline Backend

### 1. Vérifier le Jenkinsfile

Le fichier doit être présent à la racine du repo backend :

```text
cicd-tasklist-backend/Jenkinsfile
```

La pipeline backend exécute notamment :

- `npm ci` ;
- génération Prisma ;
- lint/format check avec Biome ;
- tests unitaires avec coverage ;
- tests e2e avec coverage ;
- analyse SonarQube et Quality Gate bloquante ;
- build Docker ;
- scan Trivy bloquant sur `HIGH` et `CRITICAL` ;
- génération du SBOM ;
- push DockerHub avec les tags `${BUILD_NUMBER}` et `latest`.

### 2. Créer le job Jenkins backend

Dans Jenkins :

1. Cliquer sur `New Item`.
2. Nommer le job :

```text
liamor2-backend
```

3. Choisir le type `Pipeline`.
4. Valider.

### 3. Configurer le déclenchement automatique

Dans la configuration du job :

1. Aller dans `Build Triggers`.
2. Cocher `GitHub hook trigger for GITScm polling` si le webhook GitHub est utilisé.
3. Cocher aussi `Poll SCM` comme fallback.
4. Renseigner :

```text
H/2 * * * *
```

### 4. Configurer la pipeline depuis Git

Dans la section `Pipeline` :

- Definition : `Pipeline script from SCM`
- SCM : `Git`
- Repository URL :

```text
https://github.com/liamor2/cicd-tasklist-backend
```

- Credentials Git : à renseigner seulement si le repo est privé
- Branch Specifier :

```text
*/main
```

- Script Path :

```text
Jenkinsfile
```

Cliquer sur `Save`.

### 5. Lancer et vérifier le premier build backend

1. Cliquer sur `Build Now`.
2. Vérifier que les stages passent :
   - `Install dependencies`
   - `Generate Prisma client`
   - `Lint and format check`
   - `Unit tests`
   - `E2E tests`
   - `SonarQube analysis and Quality Gate`
   - `Docker build`
   - `Trivy scan`
   - `Generate SBOM`
   - `Push Docker image`
3. Vérifier que les rapports JUnit sont visibles dans Jenkins.
4. Vérifier que les artefacts Trivy et SBOM sont archivés.
5. Vérifier SonarQube :

```text
https://sonarqube.cicd.kits.ext.educentre.fr/dashboard?id=liam-tasklist-backend
```

6. Vérifier DockerHub :
   - `liamor2/efrei-pro-pipepline-tp4-backend:<BUILD_NUMBER>`
   - `liamor2/efrei-pro-pipepline-tp4-backend:latest`

## Pipeline Frontend

### 1. Vérifier le Jenkinsfile

Le fichier doit être présent à la racine du repo frontend :

```text
cicd-tasklist-frontend/Jenkinsfile
```

La pipeline frontend exécute notamment :

- `npm ci` ;
- lint/format check avec Biome ;
- tests unitaires avec coverage ;
- build Vite ;
- analyse SonarQube et Quality Gate bloquante ;
- build Docker Nginx ;
- scan Trivy bloquant sur `HIGH` et `CRITICAL` ;
- génération du SBOM ;
- push DockerHub avec les tags `${BUILD_NUMBER}` et `latest`.

### 2. Créer le job Jenkins frontend

Dans Jenkins :

1. Cliquer sur `New Item`.
2. Nommer le job :

```text
liamor2-frontend
```

3. Choisir le type `Pipeline`.
4. Valider.

### 3. Configurer le déclenchement automatique

Dans la configuration du job :

1. Aller dans `Build Triggers`.
2. Cocher `GitHub hook trigger for GITScm polling` si le webhook GitHub est utilisé.
3. Cocher aussi `Poll SCM` comme fallback.
4. Renseigner :

```text
H/2 * * * *
```

### 4. Configurer la pipeline depuis Git

Dans la section `Pipeline` :

- Definition : `Pipeline script from SCM`
- SCM : `Git`
- Repository URL :

```text
https://github.com/liamor2/cicd-tasklist-frontend
```

- Credentials Git : à renseigner seulement si le repo est privé
- Branch Specifier :

```text
*/main
```

- Script Path :

```text
Jenkinsfile
```

Cliquer sur `Save`.

### 5. Lancer et vérifier le premier build frontend

1. Cliquer sur `Build Now`.
2. Vérifier que les stages passent :
   - `Install dependencies`
   - `Lint and format check`
   - `Unit tests`
   - `Build`
   - `SonarQube analysis and Quality Gate`
   - `Docker build`
   - `Trivy scan`
   - `Generate SBOM`
   - `Push Docker image`
3. Vérifier que les rapports JUnit sont visibles dans Jenkins.
4. Vérifier que les artefacts Trivy et SBOM sont archivés.
5. Vérifier SonarQube :

```text
https://sonarqube.cicd.kits.ext.educentre.fr/dashboard?id=liam-tasklist-frontend
```

6. Vérifier DockerHub :
   - `liamor2/efrei-pro-pipepline-tp4-frontend:<BUILD_NUMBER>`
   - `liamor2/efrei-pro-pipepline-tp4-frontend:latest`

## Webhooks GitHub

Pour déclencher automatiquement Jenkins lors d'un push sur `main`, ajouter un webhook dans chaque repo GitHub.

Dans GitHub :

1. Aller dans le repo concerné.
2. Ouvrir `Settings` > `Webhooks`.
3. Cliquer sur `Add webhook`.
4. Renseigner :

```text
Payload URL: https://jenkins.cicd.kits.ext.educentre.fr/github-webhook/
Content type: application/json
Events: Just the push event
Active: checked
```

5. Sauvegarder.
6. Vérifier dans l'onglet `Recent Deliveries` que le webhook retourne un code `200` ou `302` accepté par Jenkins.

Si GitHub retourne `403 No valid crumb was included in the request`, vérifier que l'URL est bien :

```text
/github-webhook/
```

et non :

```text
/github-webhooks
```

## Vérification par push

Pour chaque repo :

1. Faire une modification ou un commit de test.
2. Pousser sur `main`.
3. Vérifier que le job Jenkins correspondant se déclenche automatiquement.
4. Vérifier que le build est en succès.
5. Vérifier que SonarQube et DockerHub ont bien été mis à jour.

## Test local avec Jenkins Docker

Un Jenkins local est disponible via :

```bash
docker compose -f docker-compose.jenkins.yml up --build -d
```

Interface :

```text
http://localhost:8080
```

Mot de passe initial :

```bash
docker compose -f docker-compose.jenkins.yml exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Pour tester un repo local dans Jenkins, utiliser une URL Git locale :

Backend :

```text
file:///workspace/tp4/cicd-tasklist-backend
```

Frontend :

```text
file:///workspace/tp4/cicd-tasklist-frontend
```

## Points de blocage courants

### npm ci échoue

Vérifier que `package-lock.json` est versionné dans le repo concerné.

### cleanWs échoue

Installer le plugin Jenkins `Workspace Cleanup`, ou remplacer `cleanWs()` par `deleteDir()` dans le Jenkinsfile.

### Docker build échoue

Vérifier que l'agent Jenkins a accès au socket Docker et que Docker Compose est disponible.

### Trivy échoue

Si Trivy retourne un exit code `1`, lire le rapport :

```text
reports/trivy-vulnerabilities.json
```

Corriger les vulnérabilités `HIGH` ou `CRITICAL`, reconstruire l'image, puis relancer le build.

### Quality Gate SonarQube échoue

Ouvrir le dashboard SonarQube du projet, corriger les problèmes bloquants, puis relancer la pipeline.

## Optimisation des builds avec cache par stage et cache npm

Les Jenkinsfiles backend et frontend utilisent un cache par stage basé sur un hash des fichiers surveillés par chaque étape.

Objectif : ne pas relancer une étape si les fichiers qu'elle vérifie n'ont pas changé depuis la dernière exécution réussie de cette même étape.

Fonctionnement :

1. Avant chaque stage, Jenkins calcule un hash des fichiers utiles pour ce stage.
2. Jenkins compare ce hash avec un marqueur stocké dans :

```text
$HOME/.jenkins-stage-cache/<job-name>/<stage>.sha256
```

3. Si le hash existe déjà, le stage est ignoré.
4. Si le hash est nouveau ou absent, le stage est exécuté.
5. Le marqueur est mis à jour uniquement si le stage se termine avec succès.

Exemples :

- modification de `src/**` : tests, Sonar, build Docker, Trivy, SBOM et push peuvent être relancés ;
- modification de `Dockerfile` : build Docker, Trivy, SBOM et push sont relancés ;
- modification de `sonar-project.properties` : tests de coverage et Sonar peuvent être relancés ;
- modification de fichiers Markdown uniquement : les stages CI/CD principaux sont ignorés si leurs entrées n'ont pas changé depuis leur dernier succès.

Cette approche est plus précise qu'un simple `changeset`, car elle ne regarde pas seulement le dernier commit : elle vérifie si les entrées du stage sont identiques à celles d'une exécution réussie précédente.

### Cache npm

Les pipelines utilisent :

```bash
npm ci --cache "$HOME/.npm-cache" --prefer-offline
```

`npm ci` réinstalle toujours `node_modules`, ce qui est normal et garantit une installation reproductible à partir de `package-lock.json`.

En revanche, le cache npm permet d'éviter de retélécharger les paquets à chaque build. Le cache est placé hors du workspace Jenkins, donc il n'est pas supprimé par `cleanWs()`.

À retenir :

- on ne garde pas `node_modules` entre deux builds ;
- on garde le cache des paquets npm ;
- si les fichiers surveillés par `Install dependencies` n'ont pas changé depuis son dernier succès, `npm ci` est ignoré.
