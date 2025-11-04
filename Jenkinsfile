// -----------------------------------------------------------------------------
// Pipeline Jenkins pour "Sparacca-Jenkins_devops_exams"
// Objectif :
// - Construire et publier 2 images Docker (movie-service, cast-service) sur Docker Hub (public)
// - Déployer via Helm le chart mono-image 2 fois (une release par service)
// - 4 environnements (dev, qa, staging, prod) par Namespaces k8s
// - Prod déclenchée manuellement depuis la branche master
// -----------------------------------------------------------------------------

pipeline {
  agent any
  options { timestamps() }   // Journal horodaté ; couleur supprimée (plugin manquant)

  environment {
    // Identité Docker Hub (images publiques)
    DOCKER_ID    = "sparaccae"

    // Deux dépôts d’images — un par microservice
    MOVIE_IMAGE  = "movie-service"
    CAST_IMAGE   = "cast-service"

    // Tag de build traçable (immuable)
    BUILD_TAG    = "v.${BUILD_ID}.0"

    // Secret text Jenkins avec le token Docker Hub
    DOCKER_PASS  = credentials('DOCKER_HUB_PASS')
  }

  stages {

    stage('Checkout') {
      // Récupère le code du dépôt (chart Helm + sources des 2 services)
      steps { checkout scm }
    }

    stage('Build & Push: images (tag de build)') {
      // 1) Build des images avec le tag immuable BUILD_TAG
      // 2) Push ; les alias d’environnement seront créés dans les stages suivants
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin

          # movie-service
          docker build -t ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} movie-service
          docker push    ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}

          # cast-service
          docker build -t ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG} cast-service
          docker push    ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}

          docker logout || true
        '''
      }
    }

    stage('DEV: alias des tags & déploiement (toujours)') {
      // DEV réagit à chaque commit (feedback rapide)
      environment { KUBECONFIG = credentials('config') } // kubeconfig (Secret file)
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Alias :dev
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:dev
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:dev

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:dev
          docker push ${DOCKER_ID}/${CAST_IMAGE}:dev

          docker logout || true

          # Déploiements Helm (values DEV spécifiques à chaque service)
          helm upgrade --install exam-movie ./charts -n dev -f manifest/dev/values-dev-movie.yaml
          helm upgrade --install exam-cast  ./charts -n dev -f manifest/dev/values-dev-cast.yaml
        '''
      }
    }

    stage('QA: alias & déploiement (master uniquement)') {
      // QA depuis master uniquement (stabilisation)
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

          helm upgrade --install exam-movie ./charts -n qa -f manifest/qa/values-qa-movie.yaml
          helm upgrade --install exam-cast  ./charts -n qa -f manifest/qa/values-qa-cast.yaml
        '''
      }
    }

    stage('STAGING: alias & déploiement (master uniquement)') {
      // Pré-production ; même contrôle que QA
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

          helm upgrade --install exam-movie ./charts -n staging -f manifest/staging/values-staging-movie.yaml
          helm upgrade --install exam-cast  ./charts -n staging -f manifest/staging/values-staging-cast.yaml
        '''
      }
    }

    stage('PROD: validation manuelle + déploiement (master uniquement)') {
      // Production : déclenchement manuel + restriction master
      when { branch 'master' }

      // Boîte de validation avant déploiement (avec timeout)
      options { timeout(time: 20, unit: 'MINUTES') }
      input {
        message 'Déployer en production ?'
        ok 'Oui'
      }

      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Alias :prod (images stables pour prod)
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:prod
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:prod

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:prod
          docker push ${DOCKER_ID}/${CAST_IMAGE}:prod

          docker logout || true

          # Values PROD : réplicaCount=3 (déjà dans manifest/prod/*)
          helm upgrade --install exam-movie ./charts -n prod -f manifest/prod/values-prod-movie.yaml
          helm upgrade --install exam-cast  ./charts -n prod -f manifest/prod/values-prod-cast.yaml
        '''
      }
    }
  }

  post {
    always {
      // Hygiène côté agent : libérer l’espace disque Docker
      sh 'docker system prune -f || true'
    }
  }
}
