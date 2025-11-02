pipeline {
  agent any
  options { timestamps(); ansiColor('xterm') }

  environment {
    // FR: Identité DockerHub (à adapter si besoin)
    DOCKER_ID    = "sparaccae"          // ton compte DockerHub
    DOCKER_IMAGE = "datascientestapi"   // nom d’image (même logique que ton exemple)
    DOCKER_TAG   = "v.${BUILD_ID}.0"    // tag incrémental simple par build
  }

  stages {

    stage('Docker Build') {
      // FR: Construction de l’image à partir du Dockerfile du repo
      steps {
        sh '''
          docker rm -f jenkins || true
          docker build --no-cache --pull -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
          docker tag $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG $DOCKER_ID/$DOCKER_IMAGE:latest
        '''
      }
    }

    stage('Smoke local (run + test)') {
      // FR: Démarre le conteneur localement et vérifie l’endpoint
      steps {
        sh '''
          docker rm -f jenkins || true
          docker run -d -p 80:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          sleep 8
          curl -sf localhost >/dev/null
        '''
      }
    }

    stage('Docker Push') {
      // FR: Publie l’image dans DockerHub (tag versionné + latest)
      environment {
        DOCKER_PASS = credentials('DOCKER_HUB_PASS') // Secret texte Jenkins
      }
      steps {
        sh '''
          echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
          docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          docker push $DOCKER_ID/$DOCKER_IMAGE:latest
          docker logout
        '''
      }
    }

    stage('Deploy DEV (toujours)') {
      // FR: Déploiement Helm en dev, en réutilisant le chart du dépôt
      environment { KUBECONFIG = credentials('config') } // Secret file kubeconfig
      steps {
        sh '''
          rm -rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config

          # FR: on part du values.yaml fourni par le chart et on force repository + tag
          cp charts/values.yaml values.yml
          sed -i "s#^\\s*repository:.*#  repository: docker.io/${DOCKER_ID}/${DOCKER_IMAGE}#g" values.yml
          sed -i "s#^\\s*tag:.*#  tag: ${DOCKER_TAG}#g" values.yml

          # FR: release distincte ("exam") pour ne pas impacter d’autres projets
          helm upgrade --install exam charts --values=values.yml --namespace dev
        '''
      }
    }

    stage('Deploy QA (seulement master)') {
      // FR: QA auto mais uniquement sur la branche master
      when { branch 'master' }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          rm -rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s#^\\s*repository:.*#  repository: docker.io/${DOCKER_ID}/${DOCKER_IMAGE}#g" values.yml
          sed -i "s#^\\s*tag:.*#  tag: ${DOCKER_TAG}#g" values.yml

          helm upgrade --install exam charts --values=values.yml --namespace qa
        '''
      }
    }

    stage('Deploy STAGING (seulement master)') {
      // FR: STAGING auto mais uniquement sur master
      when { branch 'master' }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          rm -rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s#^\\s*repository:.*#  repository: docker.io/${DOCKER_ID}/${DOCKER_IMAGE}#g" values.yml
          sed -i "s#^\\s*tag:.*#  tag: ${DOCKER_TAG}#g" values.yml

          helm upgrade --install exam charts --values=values.yml --namespace staging
        '''
      }
    }

    stage('Deploy PROD (manuel, master uniquement)') {
      // FR: Déploiement en production soumis à approbation manuelle et seulement depuis master
      when { branch 'master' }
      steps {
        script {
          timeout(time: 15, unit: 'MINUTES') {
            input message: 'Déployer en production ?', ok: 'Oui'
          }
        }
      }
      environment { KUBECONFIG = credentials('config') }
      steps {
        sh '''
          rm -rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp charts/values.yaml values.yml
          sed -i "s#^\\s*repository:.*#  repository: docker.io/${DOCKER_ID}/${DOCKER_IMAGE}#g" values.yml
          sed -i "s#^\\s*tag:.*#  tag: ${DOCKER_TAG}#g" values.yml

          helm upgrade --install exam charts --values=values.yml --namespace prod
        '''
      }
    }
  }

  post {
    always {
      // FR: Nettoyage du conteneur de test local
      sh 'docker rm -f jenkins || true'
    }
  }
}
