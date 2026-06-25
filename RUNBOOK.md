# Runbook TP - Pipeline CI/CD bloqué par Trivy HIGH/CRITICAL

## 1. Sujet choisi

Ce runbook explique comment débloquer un pipeline CI/CD lorsque le scan Trivy d'une image Docker échoue à cause de vulnérabilités de sévérité `HIGH` ou `CRITICAL`.

Le cas principal du TP concerne les deux images Docker de l'application Tasklist :

- backend : `efrei-pro-pipepline-TP-backend:latest`
- frontend Nginx : `efrei-pro-pipepline-TP-frontend:latest`

L'exemple concret utilisé dans ce runbook est le frontend, dont l'image Nginx a été bloquée par Trivy à cause de paquets Alpine vulnérables.

## 2. Problème de production ou CI/CD traité

Le problème traité est un blocage de pipeline CI/CD au moment du scan de sécurité de l'image Docker.

Dans ce projet, Trivy est configuré pour échouer si une vulnérabilité `HIGH` ou `CRITICAL` est détectée :

```bash
bun run trivy:scan
```

Le scan produit un rapport JSON dans :

```text
reports/trivy-vulnerabilities.json
```

Si le scan échoue, l'image ne doit pas être poussée sur DockerHub tant que les vulnérabilités bloquantes n'ont pas été corrigées ou justifiées.

## 3. Symptômes

Les symptômes habituels sont :

- la commande `bun run trivy:scan` se termine avec un exit code `1` ;
- le pipeline CI/CD s'arrête sur l'étape de scan de vulnérabilités ;
- un rapport `reports/trivy-vulnerabilities.json` est généré ;
- le rapport contient au moins une vulnérabilité `HIGH` ou `CRITICAL` ;
- l'image Docker locale existe, mais ne doit pas encore être publiée.

Exemple de symptôme observé sur le frontend :

```text
error: script "trivy:scan" exited with code 1
```

Exemples de paquets détectés dans l'image Nginx avant correction :

- `libcrypto3`
- `libssl3`
- `libexpat`
- `libxml2`
- `nghttp2-libs`

## 4. Qui doit utiliser ce runbook ?

Ce runbook est destiné à :

- un développeur qui maintient le backend ou le frontend ;
- une personne responsable du pipeline CI/CD ;
- un reviewer qui doit vérifier qu'une image Docker peut être publiée ;
- un intervenant DevOps chargé de débloquer un rendu ou une livraison.

La personne qui applique ce runbook doit avoir :

- Docker fonctionnel localement ;
- Bun installé ;
- accès au repo concerné ;
- accès DockerHub si un push est nécessaire.

## 5. Quand faut-il appliquer ce runbook ?

Appliquer ce runbook immédiatement si :

- le pipeline CI/CD est bloqué par Trivy ;
- une image doit être livrée ou poussée rapidement ;
- le rapport Trivy contient une vulnérabilité `CRITICAL` ;
- le rapport Trivy contient une vulnérabilité `HIGH` avec correctif disponible ;
- l'image est utilisée pour une démonstration, un rendu ou un déploiement proche.

Application différée possible à J+3 si :

- le scan ne bloque pas encore une livraison ;
- les vulnérabilités détectées n'ont pas de version corrigée disponible ;
- l'image n'est pas exposée ni utilisée en production ;
- une correction demande une migration de version majeure qui doit être testée plus largement.

Dans le doute, traiter immédiatement les vulnérabilités `CRITICAL` et documenter les `HIGH` non corrigeables.

## 6. Quand ne faut-il surtout pas l'appliquer ?

Ne pas appliquer ce runbook tel quel si :

- le rapport Trivy concerne une image qui n'appartient pas au projet ;
- la correction proposée implique une montée de version majeure non testée ;
- le pipeline échoue à cause d'un problème de token, de registre Docker ou de réseau, et non à cause de vulnérabilités ;
- le rapport Trivy est absent ou vide, car il faut d'abord corriger la génération du rapport ;
- la correction consiste seulement à ignorer la vulnérabilité sans validation humaine ;
- l'image est déjà en production et nécessite une procédure de rollback ou de changement planifiée.

Ne jamais pousser une image corrigée sans avoir relancé le scan Trivy et vérifié que le rapport ne contient plus de `HIGH` ou `CRITICAL`.

## 7. Procédure à suivre

### 7.1 Identifier le repo concerné

Pour le backend :

```bash
cd cicd-tasklist-backend
```

Pour le frontend :

```bash
cd cicd-tasklist-frontend
```

### 7.2 Reconstruire l'image Docker

Backend :

```bash
bun run docker:build
```

Frontend :

```bash
bun run docker:build
```

Cette étape garantit que Trivy scanne la dernière version locale de l'image.

### 7.3 Lancer le scan Trivy

```bash
bun run trivy:scan
```

Résultat attendu si tout va bien :

- exit code `0` ;
- rapport généré dans `reports/trivy-vulnerabilities.json` ;
- aucune vulnérabilité `HIGH` ou `CRITICAL`.

Si la commande échoue avec exit code `1`, continuer avec l'étape suivante.

### 7.4 Lire le rapport Trivy

Ouvrir le fichier :

```text
reports/trivy-vulnerabilities.json
```

Identifier pour chaque vulnérabilité bloquante :

- le paquet concerné ;
- l'identifiant CVE ;
- la sévérité ;
- la version installée ;
- la version corrigée, si elle existe.

Commande utile pour résumer les vulnérabilités `HIGH` et `CRITICAL` :

```bash
node - <<'NODE_SUMMARY'
const fs = require('fs');
const report = JSON.parse(fs.readFileSync('reports/trivy-vulnerabilities.json', 'utf8'));
const rows = [];
for (const result of report.Results || []) {
  for (const v of result.Vulnerabilities || []) {
    if (['HIGH', 'CRITICAL'].includes(v.Severity)) {
      rows.push({
        target: result.Target,
        pkg: v.PkgName,
        id: v.VulnerabilityID,
        severity: v.Severity,
        installed: v.InstalledVersion,
        fixed: v.FixedVersion || 'aucun correctif indiqué',
      });
    }
  }
}
console.table(rows);
NODE_SUMMARY
```

### 7.5 Corriger l'image Docker

La correction dépend du paquet détecté.

Cas courant avec une image Alpine ou Nginx : si Trivy indique des versions corrigées disponibles pour des paquets système, ajouter ou conserver une mise à jour des paquets dans le stage runtime du `Dockerfile` :

```dockerfile
RUN apk upgrade --no-cache
```

Exemple appliqué au frontend Nginx :

```dockerfile
FROM nginx:1.29-alpine

RUN apk upgrade --no-cache

COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
```

Pour le backend, le même principe peut s'appliquer au stage runtime Node Alpine.

Si la vulnérabilité vient d'une dépendance applicative, corriger plutôt la dépendance concernée dans `package.json` puis régénérer le lockfile avec Bun.

### 7.6 Reconstruire l'image corrigée

```bash
bun run docker:build
```

Vérifier dans les logs que les paquets vulnérables sont bien mis à jour, par exemple :

```text
Upgrading libcrypto3
Upgrading libssl3
Upgrading libexpat
```

### 7.7 Relancer Trivy

```bash
bun run trivy:scan
```

Le scan doit maintenant se terminer avec exit code `0`.

Vérifier le nombre de vulnérabilités bloquantes :

```bash
node - <<'NODE_COUNTS'
const fs = require('fs');
const report = JSON.parse(fs.readFileSync('reports/trivy-vulnerabilities.json', 'utf8'));
const counts = { CRITICAL: 0, HIGH: 0 };
for (const result of report.Results || []) {
  for (const v of result.Vulnerabilities || []) {
    if (v.Severity in counts) counts[v.Severity]++;
  }
}
console.log(counts);
NODE_COUNTS
```

Résultat attendu :

```text
{ CRITICAL: 0, HIGH: 0 }
```

### 7.8 Produire le SBOM

Backend ou frontend :

```bash
bun run trivy:sbom
```

Le SBOM doit être généré ici :

```text
reports/sbom.cdx.json
```

Ce fichier sert à documenter les composants présents dans l'image Docker analysée.

### 7.9 Pousser l'image DockerHub

Ne faire cette étape que si le scan Trivy est passé.

Frontend :

```bash
bun run docker:push
```

Image publiée :

```text
liamor2/efrei-pro-pipepline-TP-frontend:latest
```

Pour le backend, utiliser le script ou la commande de tag/push équivalente si elle est présente dans le repo.

## 8. Validation finale

Avant de fermer l'incident ou de relancer la CI, vérifier :

- `bun run trivy:scan` retourne exit code `0` ;
- `reports/trivy-vulnerabilities.json` existe ;
- le rapport contient `0` vulnérabilité `HIGH` ;
- le rapport contient `0` vulnérabilité `CRITICAL` ;
- `reports/sbom.cdx.json` existe ;
- l'image Docker corrigée est reconstruite ;
- si nécessaire, l'image est poussée sur DockerHub ;
- le pipeline CI/CD peut être relancé.

Dans le cas frontend du TP, la validation attendue est :

```text
CRITICAL: 0
HIGH: 0
DockerHub: liamor2/efrei-pro-pipepline-TP-frontend:latest
SBOM: reports/sbom.cdx.json
```

## 9. Escalade

Escalader au responsable technique ou à l'enseignant si :

- Trivy détecte une vulnérabilité `CRITICAL` sans correctif disponible ;
- la correction nécessite une montée de version majeure ;
- la mise à jour casse le build ou le démarrage de l'application ;
- le push DockerHub échoue malgré une authentification valide ;
- le pipeline reste bloqué alors que le scan local passe ;
- une vulnérabilité doit être ignorée temporairement.

Toute exception doit être documentée avec :

- la CVE concernée ;
- le paquet concerné ;
- la raison de l'exception ;
- la date de revue prévue ;
- la personne qui a validé l'exception.

## 10. Résumé rapide

Pour le frontend :

```bash
cd cicd-tasklist-frontend
bun run docker:build
bun run trivy:scan
bun run trivy:sbom
bun run docker:push
```

Pour le backend :

```bash
cd cicd-tasklist-backend
bun run docker:build
bun run trivy:scan
bun run trivy:sbom
```

Ne jamais exécuter le push DockerHub si `bun run trivy:scan` échoue.
