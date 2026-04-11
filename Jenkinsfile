pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kaniko-build
spec:
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:3355.v388858a_47b_33-17
      tty: true
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command:
        - /busybox/cat
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      emptyDir: {}
"""
    }
  }

  environment {
    IMAGE_NAME = "swiber/myapp"
    IMAGE_TAG  = "${BUILD_NUMBER}"
  }

  stages {
    stage('Checkout App') {
      steps {
        checkout scm
      }
    }

    stage('Prepare Docker Auth') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            mkdir -p /home/jenkins/agent/docker-config
            AUTH=$(printf "%s:%s" "$DOCKER_USER" "$DOCKER_PASS" | base64 | tr -d '\n')
            cat > /home/jenkins/agent/docker-config/config.json <<EOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$AUTH"
    }
  }
}
EOF
          '''
        }
      }
    }

    stage('Build & Push Image with Kaniko') {
      steps {
        container('kaniko') {
          sh '''
            cp /home/jenkins/agent/docker-config/config.json /kaniko/.docker/config.json
            /kaniko/executor \
              --context "${WORKSPACE}" \
              --dockerfile "${WORKSPACE}/Dockerfile" \
              --destination "${IMAGE_NAME}:${IMAGE_TAG}"
          '''
        }
      }
    }

    stage('Update Infra Repo') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-creds',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_PASS'
        )]) {
          sh '''
            set -e
            rm -rf infra-repo
            git clone https://${GIT_USER}:${GIT_PASS}@github.com/sw1ber/infra-repo.git
            cd infra-repo

            sed -i 's#image: .*#image: '"${IMAGE_NAME}:${IMAGE_TAG}"'#' k8s/deployment.yaml

            git config user.name "jenkins"
            git config user.email "jenkins@local"
            git add k8s/deployment.yaml
            git commit -m "update image to ${IMAGE_NAME}:${IMAGE_TAG}" || true
            git push origin main
          '''
        }
      }
    }
  }
}