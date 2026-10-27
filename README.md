# Laboratoire CI/CD Laravel

Application minimale basée sur Laravel 12 pour comparer un même pipeline sur
GitHub Actions, Jenkins et GitLab CI.

## Prérequis locaux

- PHP 8.3 avec les extensions `mbstring`, `xml`, `pdo_sqlite` et `zip`
- Composer 2

## Installation et lancement des contrôles

```sh
cp .env.example .env
composer install
php artisan key:generate
composer test
composer audit
```

`composer test` exécute les tests unitaires et fonctionnels Laravel.
`composer audit` recherche les vulnérabilités connues dans les dépendances
Composer et fait échouer le pipeline si une vulnérabilité est détectée.

## Pipelines

- GitHub Actions : `.github/workflows/ci.yml`
- Jenkins : `Jenkinsfile`
- GitLab CI : `.gitlab-ci.yml`

Les pipelines GitHub Actions, Jenkins et GitLab CI exécutent les contrôles
Composer, l'audit des dépendances et les tests. Sur `main`, ils préparent aussi
une release ; le déploiement en production demande une confirmation manuelle.
Si le déploiement en production échoue, chaque pipeline affiche un message
d'exemple indiquant qu'un rollback devrait être effectué. Aucune action de
rollback n'est exécutée.

Jenkins doit exécuter le job sur un agent avec PHP 8.3, Composer 2 et les
extensions PHP listées ci-dessus. Ajoutez les identifiants Jenkins
`production-ssh-key` (clé privée SSH) et `production-known-hosts` (fichier
`known_hosts`) dans **Manage Jenkins > Credentials**. Configurez aussi les
variables d'environnement `DEPLOY_USER`, `DEPLOY_HOST` et `DEPLOY_PATH` pour
le job. Le serveur doit contenir sa configuration Laravel de production dans
`DEPLOY_PATH/shared/.env`. Après les contrôles, ouvrez l'étape **Deploy to
production** dans Jenkins et confirmez le déploiement.

GitLab CI utilise l'image officielle `php:8.3-cli` et installe Composer après
vérification de la signature de son installateur.

L'audit Composer couvre les dépendances PHP ; il ne remplace pas une analyse
SAST, un scan des secrets ou un audit de sécurité applicatif.

Pour des builds reproductibles, lancez `composer update` une première fois,
puis versionnez le `composer.lock` généré. En son absence, Composer résout les
dernières versions compatibles à chaque exécution.
