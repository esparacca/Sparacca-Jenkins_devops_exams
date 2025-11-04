// -----------------------------------------------------------------------------
// Pipeline Jenkins pour "Sparacca-Jenkins_devops_exams"
// Objectif :
// - Construire et publier 2 images Docker (movie-service, cast-service) sur Docker Hub (public)
// - Déployer via Helm un chart commun, 2 releases (movie/cast) par environnement
// - Environnements : dev (toujours), qa & staging (uniquement sur master), prod (uniquement sur master + validation manuelle)
// - Tags : on construit un tag de build immuable (BUILD_TAG), puis on crée des alias dev/qa/staging/prod
// -----------------------------------------------------------------------------

pipeline {
  agent any
  options { timestamps() }  // ansiColor retiré (plugin non garanti)

  environment {
    DOCKER_ID   = "sparaccae"
    MOVIE_IMAGE = "movie-service"
    CAST_IMAGE  = "cast-service"
    BUILD_TAG   = "v.${BUILD_ID}.0"

    // Secret text Jenkins: DOCKER_HUB_PASS
    DOCKER_PASS = credentials('DOCKER_HUB_PASS')
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push: images (tag de build)') {
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin

          # movie-service
          docker build -t ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} movie-service
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}

          # cast-service
          docker build -t ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG} cast-service
          docker push ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}

          docker logout || true
        '''
      }
    }

    stage('DEV: alias des tags & déploiement (toujours)') {
      environment { KUBECONFIG = credentials('config') } // kubeconfig (Secret file)
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Alias :dev (movie + cast)
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:dev
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:dev

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:dev
          docker push ${DOCKER_ID}/${CAST_IMAGE}:dev

          docker logout || true

          # Déploiements Helm (namespace dev)
          helm upgrade --install exam-movie ./charts -n dev -f manifest/dev/values-dev-movie.yaml
          helm upgrade --install exam-cast  ./charts -n dev -f manifest/dev/values-dev-cast.yaml
        '''
      }
    }

    stage('QA: alias & déploiement (master uniquement)') {
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
      when { branch 'master' }     // barrière 1 : jamais hors de master
      environment { KUBECONFIG = credentials('config') }
      steps {
        script {
          // barrière 2 : validation explicite
          timeout(time: 20, unit: 'MINUTES') {
            input message: 'Déployer en production ?', ok: 'Oui'
          }
        }
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # Aliases prod
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:prod
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:prod

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:prod
          docker push ${DOCKER_ID}/${CAST_IMAGE}:prod

          docker logout || true

          # Déploiements Helm (namespace prod)
          helm upgrade --install exam-movie ./charts -n prod -f manifest/prod/values-prod-movie.yaml
          helm upgrade --install exam-cast  ./charts -n prod -f manifest/prod/values-prod-cast.yaml
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
