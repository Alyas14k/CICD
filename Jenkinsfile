pipeline {
    agent any

    // Ajoute les horodatages aux lignes de log pour faciliter le diagnostic.
    options {
        timestamps()
    }

    // Autorise Composer à fonctionner dans un agent Jenkins exécuté en root.
    environment {
        COMPOSER_ALLOW_SUPERUSER = '1'
    }

    stages {
        // Vérifie la présence des fichiers Laravel et la configuration Composer.
        stage('Code check') {
            steps {
                sh 'test -f artisan'
                sh 'test -f composer.json'
                sh 'composer validate --no-check-publish'
            }
        }

        // Installe les dépendances utilisées par les étapes suivantes.
        stage('Build dependencies') {
            steps {
                sh 'composer install --prefer-dist --no-interaction --no-progress'
                sh 'test -f vendor/autoload.php'
            }
        }

        // Recherche les vulnérabilités connues dans les dépendances Composer.
        stage('Dependency audit') {
            steps {
                sh 'composer audit --no-interaction'
            }
        }

        // Prépare une base SQLite temporaire et exécute les tests Laravel.
        stage('Laravel tests') {
            steps {
                sh '''
                    cp .env.testing .env
                    mkdir -p database
                    touch database/database.sqlite
                    php artisan key:generate --force
                    php artisan config:clear
                    php artisan migrate:fresh --env=testing --force
                    php artisan test
                '''
            }
        }

        // Produit une archive de production uniquement pour la branche main.
        stage('Build production release') {
            when {
                expression {
                    env.BRANCH_NAME == 'main' || env.GIT_BRANCH in ['main', 'origin/main']
                }
            }
            steps {
                sh 'composer install --no-dev --prefer-dist --no-interaction --no-progress --optimize-autoloader'
                sh '''
                    set -eu
                    archive="$(mktemp)"
                    trap 'rm -f "$archive"' EXIT
                    tar -czf "$archive" \
                        --exclude=.git \
                        --exclude=.env \
                        --exclude=.env.testing \
                        --exclude=database/database.sqlite \
                        --exclude=release.tar.gz \
                        .
                    cp "$archive" release.tar.gz
                '''
                stash name: 'production-release', includes: 'release.tar.gz'
            }
        }

        // Demande une approbation avant le déploiement de la branche main.
        // Les identifiants SSH et les variables DEPLOY_* doivent être configurés dans Jenkins.
        stage('Deploy to production') {
            when {
                expression {
                    env.BRANCH_NAME == 'main' || env.GIT_BRANCH in ['main', 'origin/main']
                }
            }
            steps {
                input message: 'Deploy this release to production?', ok: 'Deploy'
                unstash 'production-release'
                withCredentials([
                    // Clé privée SSH Jenkins et fichier known_hosts vérifié.
                    sshUserPrivateKey(
                        credentialsId: 'production-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    ),
                    file(
                        credentialsId: 'production-known-hosts',
                        variable: 'SSH_KNOWN_HOSTS_FILE'
                    )
                ]) {
                    sh '''
                        set -eu

                        : "${DEPLOY_USER:?Configure DEPLOY_USER in Jenkins}"
                        : "${DEPLOY_HOST:?Configure DEPLOY_HOST in Jenkins}"
                        : "${DEPLOY_PATH:?Configure DEPLOY_PATH in Jenkins}"

                        case "$DEPLOY_USER" in
                            ''|*[!A-Za-z0-9._-]*) echo "Invalid DEPLOY_USER" >&2; exit 1 ;;
                        esac
                        case "$DEPLOY_HOST" in
                            ''|*[!A-Za-z0-9._:-]*) echo "Invalid DEPLOY_HOST" >&2; exit 1 ;;
                        esac
                        case "$DEPLOY_PATH" in
                            /*) ;;
                            *) echo "DEPLOY_PATH must be an absolute path" >&2; exit 1 ;;
                        esac
                        case "$DEPLOY_PATH" in
                            *[!A-Za-z0-9_./-]*) echo "DEPLOY_PATH may contain only letters, numbers, '.', '_', '/', and '-'" >&2; exit 1 ;;
                        esac

                        remote="$DEPLOY_USER@$DEPLOY_HOST"
                        remote_release="$DEPLOY_PATH/releases/$GIT_COMMIT"
                        archive="release-$GIT_COMMIT.tar.gz"
                        ssh_options="-i $SSH_KEY -o UserKnownHostsFile=$SSH_KNOWN_HOSTS_FILE -o StrictHostKeyChecking=yes"

                        scp $ssh_options release.tar.gz "$remote:/tmp/$archive"
                        ssh $ssh_options "$remote" \
                            "mkdir -p '$remote_release' '$DEPLOY_PATH/shared/storage'"
                        ssh $ssh_options "$remote" \
                            "tar -xzf '/tmp/$archive' -C '$remote_release' && rm -f '/tmp/$archive'"
                        ssh $ssh_options "$remote" \
                            "test -f '$DEPLOY_PATH/shared/.env'"
                        ssh $ssh_options "$remote" \
                            "cd '$remote_release' &&
                             ln -sfn '$DEPLOY_PATH/shared/.env' .env &&
                             rm -rf storage &&
                             ln -sfn '$DEPLOY_PATH/shared/storage' storage &&
                             mkdir -p bootstrap/cache &&
                             php artisan migrate --force &&
                             php artisan config:cache &&
                             ln -sfn '$remote_release' '$DEPLOY_PATH/current'"
                    '''
                }
            }
            post {
                failure {
                    echo 'Exemple : le déploiement en production a échoué ; un rollback devrait être effectué.'
                }
            }
        }
    }
}
