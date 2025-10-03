pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Ensure docker alias (fix TLS hostname)') {
            steps {
                sh '''
                  set -eu
                  if getent hosts docker; then
                    echo "docker already resolves"
                  else
                    echo "127.0.0.1 docker" >> /etc/hosts
                  fi
                '''
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/vrushti54/securedevops-nodeapp-22167521.git'
            }
        }

        stage('Install (Node in Docker)') {
            steps {
                sh '''
                  set -e
                  echo "--- using Node inside Docker ---"
                  docker run --rm -v $PWD:/app -w /app node:18-alpine sh -lc "
                    node -v
                    npm -v
                    npm ci
                  "
                '''
            }
        }

        stage('Test (Node in Docker)') {
            steps {
                sh '''
                  docker run --rm -v $PWD:/app -w /app node:18-alpine sh -lc "
                    npm test --silent || true
                  "
                '''
            }
        }

        stage('Security Scan (OWASP DC)') {
            steps {
                sh '''
                  mkdir -p dep-report
                  docker run --rm \
                    -v $PWD:/src -w /src \
                    -v $PWD/dep-report:/report \
                    owasp/dependency-check:latest \
                    --project nodeapp --scan /src --format HTML --out /report --enableExperimental || true
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker build -t vrushti672/securedevops-nodeapp-22167521:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_HUB_USER', passwordVariable: 'DOCKER_HUB_PASS')]) {
                    sh '''
                      echo "$DOCKER_HUB_PASS" | docker login -u "$DOCKER_HUB_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
                  docker tag vrushti672/securedevops-nodeapp-22167521:${BUILD_NUMBER} vrushti672/securedevops-nodeapp-22167521:latest
                  docker push vrushti672/securedevops-nodeapp-22167521:${BUILD_NUMBER}
                  docker push vrushti672/securedevops-nodeapp-22167521:latest
                '''
            }
        }

        stage('Run Container (port 3001 -> 8080)') {
            steps {
                sh '''
                  docker stop nodeapp-test || true
                  docker rm nodeapp-test || true
                  docker run -d --name nodeapp-test -p 3001:8080 vrushti672/securedevops-nodeapp-22167521:${BUILD_NUMBER}
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline complete"
            archiveArtifacts artifacts: 'dep-report/*', allowEmptyArchive: true
        }
    }
}
