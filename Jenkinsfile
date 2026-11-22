
pipeline {
    agent any

    environment {
        IMAGE_NAME = "raviiprakashh/gitops-node-app"
        GITOPS_REPO = "https://github.com/Ravi-Pr4kash/GitOps-K8S-Config.git"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out application source code'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "jenkins-${BUILD_NUMBER}"

                    sh 'docker build -t "$IMAGE_NAME:$IMAGE_TAG" .'
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" --password-stdin

                        docker push "$IMAGE_NAME:$IMAGE_TAG"

                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        ASKPASS=$(mktemp)
                        trap 'rm -f "$ASKPASS"' EXIT

                        cat > "$ASKPASS" <<'EOF'
#!/bin/sh
case "$1" in
    *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
    *Password*) printf '%s\\n' "$GIT_PASSWORD" ;;
esac
EOF

                        chmod 700 "$ASKPASS"
                        export GIT_ASKPASS="$ASKPASS"
                        export GIT_TERMINAL_PROMPT=0

                        rm -rf gitops-config
                        git clone "$GITOPS_REPO" gitops-config

                        cd gitops-config

                        sed -i '' \
                          "s/^    tag: .*/    tag: ${IMAGE_TAG}/" \
                          helm/gitops-platform/values.yaml

                        git config user.name "Jenkins"
                        git config user.email "jenkins@localhost"

                        git add helm/gitops-platform/values.yaml
                        git commit -m "Update Node app image to ${IMAGE_TAG}"
                        git push origin HEAD
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }
    }
}
