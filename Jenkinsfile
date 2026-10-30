
// CI/CD for Product Catalog Service

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
    }

    tools {
        go 'Go1.22'
    }

    environment {
        DOCKER_IMAGE   = 'product-catalog'
        DOCKERFILE_DIR = 'src/product-catalog'
        K8S_MANIFEST   = 'kubernetes/productcatalog/deploy.yaml'
        GIT_REPO_URL   = 'https://github.com/Pandu2451M/Opentelemetry_project.git'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Dependencies') {
            steps {
                dir('src/product-catalog') {
                    sh 'go mod download'
                }
            }
        }

        stage('Lint') {
            steps {
                dir('src/product-catalog') {
                    sh '''
                        echo "Checking Go version..."
                        go version

                        echo "Checking golangci-lint version..."
                        golangci-lint --version

                        echo "Running Linter..."
                        golangci-lint run
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                dir('src/product-catalog') {
                    sh 'go build -o product-catalog-service main.go'
                }
            }
        }

        stage('Test') {
            steps {
                dir('src/product-catalog') {
                    sh 'go test ./...'
                }
            }
        }

        stage('Docker Build and Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -eu

                        echo "$DOCKERHUB_TOKEN" | docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        docker build \
                            -t "${DOCKERHUB_USER}/${DOCKER_IMAGE}:${BUILD_NUMBER}" \
                            "$DOCKERFILE_DIR"

                        docker push \
                            "${DOCKERHUB_USER}/${DOCKER_IMAGE}:${BUILD_NUMBER}"

                        docker logout
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    ),
                    usernamePassword(
                        credentialsId: 'github-token',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        IMAGE="${DOCKERHUB_USER}/product-catalog:${BUILD_NUMBER}"

                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        sed -i \
                          "s|^\\([[:space:]]*image:[[:space:]]*\\).*|\\1$IMAGE|" \
                          "$K8S_MANIFEST"

                        git add "$K8S_MANIFEST"

                        if ! git diff --cached --quiet; then
                            git commit -m "[CI]: Update product catalog image to $BUILD_NUMBER"

                            git push \
                              "https://x-access-token:${GITHUB_TOKEN}@github.com/Pandu2451M/Opentelemetry_project.git" \
                              HEAD:main
                        else
                            echo "No manifest changes (loop protection)"
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS — image ${env.DOCKER_IMAGE}:${env.BUILD_NUMBER} pushed and manifest updated."
        }

        failure {
            echo 'Pipeline failed — check the stage logs above.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}

