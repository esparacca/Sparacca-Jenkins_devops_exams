// Pipeline lineare con gate manuali per QA, STAGING e PROD.
// DEV sempre automatico; PROD consentita solo da la branche master.

pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKER_ID   = "sparaccae"
    MOVIE_IMAGE = "movie-service"
    CAST_IMAGE  = "cast-service"
    BUILD_TAG   = "v.${BUILD_ID}.0"

    // Jenkins credentials:
    // - DOCKER_HUB_PASS = Secret Text (Docker Hub token)
    // - config = Secret file (kubeconfig)
    DOCKER_PASS = credentials('DOCKER_HUB_PASS')
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push: tag de build') {
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin

          # movie-service (tag immuable)
          docker build -t ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} movie-service
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}

          # cast-service (tag immuable)
          docker build -t ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG} cast-service
          docker push ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}

          docker logout || true
        '''
      }
    }

    stage('DEV: alias & déploiement') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # alias :dev
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:dev
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:dev

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:dev
          docker push ${DOCKER_ID}/${CAST_IMAGE}:dev

          docker logout || true

          # déploiements Helm (namespace dev)
          helm upgrade --install exam-movie ./charts -n dev -f manifest/dev/values-dev-movie.yaml
          helm upgrade --install exam-cast  ./charts -n dev -f manifest/dev/values-dev-cast.yaml
        '''
      }
    }

    stage('Gate QA') {
      steps {
        script {
          timeout(time: 30, unit: 'MINUTES') {
            input message: 'Déployer en QA ?'
          }
        }
      }
    }

    stage('QA: alias & déploiement') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # alias :qa
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:qa
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:qa

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:qa
          docker push ${DOCKER_ID}/${CAST_IMAGE}:qa

          docker logout || true

          # déploiements Helm (namespace qa)
          helm upgrade --install exam-movie ./charts -n qa -f manifest/qa/values-qa-movie.yaml
          helm upgrade --install exam-cast  ./charts -n qa -f manifest/qa/values-qa-cast.yaml
        '''
      }
    }

    stage('Gate STAGING') {
      steps {
        script {
          timeout(time: 30, unit: 'MINUTES') {
            input message: 'Déployer en STAGING ?'
          }
        }
      }
    }

    stage('STAGING: alias & déploiement') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # alias :staging
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:staging
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:staging

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:staging
          docker push ${DOCKER_ID}/${CAST_IMAGE}:staging

          docker logout || true

          # déploiements Helm (namespace staging)
          helm upgrade --install exam-movie ./charts -n staging -f manifest/staging/values-staging-movie.yaml
          helm upgrade --install exam-cast  ./charts -n staging -f manifest/staging/values-staging-cast.yaml
        '''
      }
    }

    stage('Gate PROD') {
      steps {
        script {
          // sécurité: PROD uniquement depuis master
          def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
          if (!branchName.toLowerCase().contains('master')) {
            error "Déploiement PROD bloqué : branche courante = '${branchName}'. Seule la branche 'master' est autorisée."
          }
          timeout(time: 30, unit: 'MINUTES') {
            input message: 'Déployer en PRODUCTION ?'
          }
        }
      }
    }

    stage('PROD: alias & déploiement') {
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin || true

          # alias :prod
          docker pull ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${MOVIE_IMAGE}:${BUILD_TAG} ${DOCKER_ID}/${MOVIE_IMAGE}:prod
          docker push ${DOCKER_ID}/${MOVIE_IMAGE}:prod

          docker pull ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}
          docker tag  ${DOCKER_ID}/${CAST_IMAGE}:${BUILD_TAG}  ${DOCKER_ID}/${CAST_IMAGE}:prod
          docker push ${DOCKER_ID}/${CAST_IMAGE}:prod

          docker logout || true

          # déploiements Helm (namespace prod)
          helm upgrade --install exam-movie ./charts -n prod -f manifest/prod/values-prod-movie.yaml
          helm upgrade --install exam-cast  ./charts -n prod -f manifest/prod/values-prod-cast.yaml
        '''
      }
    }
  }

  post {
    always {
      sh 'docker system prune -f || true'
    }
  }
}
