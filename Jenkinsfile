pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "${DOCKERHUB_USER}/k8s-cicd-demo"
    KUBECONFIG_CREDENTIAL = "kubeconfig-file"   // Jenkins credential ID for kubeconfig
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set Docker Tag') {
      steps {
        script {
          // Use short git commit hash as unique tag
          env.DOCKER_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          echo "Using Docker tag: ${env.DOCKER_TAG}"
        }
      }
    }

    stage('Build Image') {
      steps {
        script {
          sh """
            cd app
            docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
            docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
          """
        }
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerHub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PWD')]) {
          sh """
            echo \$DH_PWD | docker login -u \$DH_USER --password-stdin
            docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
            docker push ${DOCKER_IMAGE}:latest
          """
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG')]) {
          sh """
            kubectl apply -f k8s/deployment.yaml
            kubectl apply -f k8s/service.yaml
            kubectl set image deployment/k8s-cicd-demo web=${DOCKER_IMAGE}:${DOCKER_TAG} --record
            kubectl rollout status deployment/k8s-cicd-demo --timeout=120s
          """
        }
      }
    }

    stage('Verify') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG')]) {
          sh "kubectl get pods -o wide"
          sh "kubectl get svc k8s-cicd-demo-svc -o wide"
        }
      }
    }
  }

  post {
    always {
      sh "docker logout || true"
    }
  }
}
