pipeline {
  environment {
    DOCKER_ID = "maxjokar2020" // Your Docker Hub ID
    DOCKER_IMAGE = "datascientestapi"
    DOCKER_TAG = "v.${BUILD_ID}.0"
  }
  agent any

  stages {
    stage('Docker Build') {
      steps {
        script {
          sh '''
          docker rm -f jenkins || true
          docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
          sleep 6
          '''
        }
      }
    }

    stage('Docker Run') {
      steps {
        script {
          sh '''
          docker run -d -p 80:80 --name jenkins $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          sleep 10
          '''
        }
      }
    }

    stage('Test Acceptance') {
      steps {
        script {
          sh 'curl localhost'
        }
      }
    }

    stage('Docker Push') {
      environment {
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
      }
      steps {
        script {
          sh '''
          docker login -u $DOCKER_ID -p $DOCKER_PASS
          docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
          '''
        }
      }
    }

    stage('Deploy to Dev') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install app fastapi --values=values.yml --namespace dev
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        script {
          sh '''
          rm -Rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install app fastapi --values=values.yml --namespace staging
          '''
        }
      }
    }

    stage('Deploy to Prod') {
      environment {
        KUBECONFIG = credentials("config")
      }
      steps {
        timeout(time: 15, unit: "MINUTES") {
          input message: 'Do you want to deploy in production?', ok: 'Yes'
        }
        script {
          sh '''
          rm -Rf .kube && mkdir .kube
          cat $KUBECONFIG > .kube/config
          cp fastapi/values.yaml values.yml
          sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
          kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install app fastapi --values=values.yml --namespace prod
          '''
        }
      }
    }
  }
}

