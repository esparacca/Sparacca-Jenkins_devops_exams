// -----------------------------------------------------------------------------
// Pipeline Jenkins pour "Sparacca-Jenkins_devops_exams"
// Objectif :
// - Construire et publier 2 images Docker (movie-service, cast-service) sur Docker Hub (public)
// - Déployer via Helm le même chart mono-image, 2 fois (une release par service)
// - 4 environnements (dev, qa, staging, prod) basés sur des Namespaces k8s
// - Production déclenchée manuellement depuis la branche master
// - Alignement avec nos choix : images publiques (pas de secret), Service=ClusterIP (pas de NodePort),
//   probes définies dans les values par service (/api/v1/movies, /api/v1/casts), réplica=1 sauf prod=3
// -----------------------------------------------------------------------------

pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    // Identité Docker Hub (images publiques pour éviter toute gestion de secret k8s)
    DOCKER_ID    = "sparaccae"

    // Deux dépôts d’images — un par microservice issu du docker-compose
    MOVIE_IMAGE  = "movie-service"
    CAST_IMAGE   = "cast-service"

    // Tag de build traçable : ne pas confondre avec les tags d’environnement (dev/qa/staging/prod)
    BUILD_TAG    = "v.${BUILD_ID}.0"

    // Identifiants Docker Hub (Secret Text Jenkins nommé DOCKER_HUB_PASS)
    // Placé au niveau global car utilisé dans plusieurs stages (login/push)
    DOCKER_PASS  = credentials('DOCKER_HUB_PASS')
  }

  stages {

    stage('Checkout') {
      // Récupère le code du dépôt (chart Helm + sources des 2 services)
      steps { checkout scm }
    }

    stage('Build & Push: images (tag de build)') {
      // Processus :
      // 1) On construit d’abord un tag de build immuable (BUILD_TAG) pour garantir la traçabilité.
      // 2) Petit smoke test local (run + curl) sur movie pour valider l’image.
      // 3) On pousse ce tag. Les tags d’environnement (dev/qa/staging/prod) seront des alias créés ensuite.
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin

          # movie-service : build à partir du dossier dédié
          docker build -t ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} movie-service

          # cast-service : idem
          docker build -t ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG} cast-service

          # Smoke test local (movie) : on vérifie que l’API répond
          docker rm -f movie-test || true
          docker run -d -p 8001:8000 --name movie-test ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          sleep 5
          curl -f http://localhost:8001/api/v1/movies/ || (echo "Smoke test KO"; docker logs --tail=100 movie-test; docker rm -f movie-test || true; exit 1)
          docker rm -f movie-test || true

          # Push des deux images (tag de build immuable)
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker push ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}

          docker logout || true
        '''
      }
    }

    stage('DEV: alias des tags & déploiement (toujours)') {
      // Logique :
      // - DEV doit réagir à chaque commit (retour rapide)
      // - On crée les alias "dev" à partir du tag de build, puis on déploie
      // - Les values DEV (movie/cast) : ClusterIP, 1 réplica, probes adaptées
      environment { KUBECONFIG = credentials('config') } // Secret file kubeconfig
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Alias :dev pour movie et cast
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:dev
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:dev

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:dev
          docker push ${DOCKER_ID}/${CAST_IMAGE}:dev

          docker logout || true

          # Déploiements Helm (chart mono-image, deux releases : une par service)
          # Remarque : les values-dev-*.yaml contiennent ClusterIP, probes et tag "dev"
          helm upgrade --install exam-movie charts -n dev -f charts/values-dev-movie.yaml
          helm upgrade --install exam-cast  charts -n dev -f charts/values-dev-cast.yaml
        '''
      }
    }

    stage('QA: alias & déploiement (master uniquement)') {
      // Logique :
      // - QA ne doit se déployer que depuis la branche master (stabilisation)
      // - Même principe : alias ":qa" puis déploiement des deux releases
      when { branch 'master' }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:qa
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:qa

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:qa
          docker push ${DOCKER_ID}/${CAST_IMAGE}:qa

          docker logout || true

          # Values QA : identiques à DEV, seul le tag change ("qa")
          helm upgrade --install exam-movie charts -n qa -f charts/values-qa-movie.yaml
          helm upgrade --install exam-cast  charts -n qa -f charts/values-qa-cast.yaml
        '''
      }
    }

    stage('STAGING: alias & déploiement (master uniquement)') {
      // Logique :
      // - STAGING simule une pré-prod ; même contrôle que QA (seulement sur master)
      // - On conserve 1 réplica (VM unique), mais on pourrait augmenter ici si nécessaire
      when { branch 'master' }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:staging
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:staging

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:staging
          docker push ${DOCKER_ID}/${CAST_IMAGE}:staging

          docker logout || true

          # Values STAGING : tag "staging", mêmes probes/ClusterIP
          helm upgrade --install exam-movie charts -n staging -f charts/values-staging-movie.yaml
          helm upgrade --install exam-cast  charts -n staging -f charts/values-staging-cast.yaml
        '''
      }
    }

    stage('PROD: validation manuelle + déploiement (master uniquement)') {
      // Exigence de l’énoncé :
      // - Déploiement en production uniquement depuis la branche master
      // - Déclenchement explicitement manuel (bouton "Oui")
      when { branch 'master' }
      steps {
        script {
          timeout(time: 20, unit: 'MINUTES') {
            input message: 'Déployer en production ?', ok: 'Oui'
          }
        }
      }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Alias ":prod" (images stables correspondant au déploiement prod)
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:prod
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:prod

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:prod
          docker push ${DOCKER_ID}/${CAST_IMAGE}:prod

          docker logout || true

          # Values PROD : réplicas=3 (haute dispo), probes et ClusterIP
          helm upgrade --install exam-movie charts -n prod -f charts/values-prod-movie.yaml
          helm upgrade --install exam-cast  charts -n prod -f charts/values-prod-cast.yaml
        '''
      }
    }
  }

  post {
    always {
      // Hygiène côté agent : libérer l’espace disque
      sh 'docker system prune -f || true'
    }
  }
}
