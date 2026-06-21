pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        DOCKER_HUB_PASS_CRED_ID = 'DOCKER_HUB_PASS'
        DOCKERHUB_NAMESPACE     = 'henokoder'
        KUBECONFIG_CRED_ID      = 'config'
        CHART_PATH              = 'charts'
        COMPOSE_PROJECT_NAME    = 'moviesapp'
        MOVIE_IMAGE             = "${DOCKERHUB_NAMESPACE}/movie-service"
        CAST_IMAGE              = "${DOCKERHUB_NAMESPACE}/cast-service"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHORT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG     = "${env.BUILD_NUMBER}-${env.GIT_SHORT_SHA}"
                    echo "Branche : ${env.BRANCH_NAME} | Tag image : ${env.IMAGE_TAG}"
                }
            }
        }

        stage("Resolution de l'environnement cible") {
            steps {
                script {
                    def envMap = [
                        'develop': [ns: 'dev',     auto: true,  moviePort: 30001, castPort: 30002],
                        'QA'     : [ns: 'qa',      auto: true,  moviePort: 30003, castPort: 30004],
                        'staging': [ns: 'staging', auto: true,  moviePort: 30005, castPort: 30006],
                        'main' : [ns: 'prod',    auto: false, moviePort: 30007, castPort: 30008],
                        'main' : [ns: 'prod',    auto: false, moviePort: 30007, castPort: 30008],
                    ]

                    def cfg = envMap[env.BRANCH_NAME]
                    if (cfg == null) {
                        echo "Branche '${env.BRANCH_NAME}' non mappee a un environnement : build + tests uniquement, pas de deploiement."
                        env.TARGET_NS   = ''
                        env.IS_PROD     = 'false'
                        env.AUTO_DEPLOY = 'false'
                    } else {
                        env.TARGET_NS      = cfg.ns
                        env.IS_PROD        = (cfg.ns == 'prod') ? 'true' : 'false'
                        env.AUTO_DEPLOY    = cfg.auto.toString()
                        env.MOVIE_NODEPORT = cfg.moviePort.toString()
                        env.CAST_NODEPORT  = cfg.castPort.toString()
                        echo "Namespace cible : ${env.TARGET_NS} | Deploiement automatique : ${env.AUTO_DEPLOY}"
                    }
                }
            }
        }

        stage('Lint du chart Helm') {
            when { expression { env.TARGET_NS != '' } }
            steps {
                sh "helm lint ${CHART_PATH}"
            }
        }

        stage('Build des images (docker-compose)') {
            when { expression { env.TARGET_NS != '' } }
            steps {
                sh """
                  docker-compose -f docker-compose.yml build movie_service cast_service

                  docker tag ${COMPOSE_PROJECT_NAME}_movie_service:latest ${MOVIE_IMAGE}:${IMAGE_TAG}
                  docker tag ${COMPOSE_PROJECT_NAME}_movie_service:latest ${MOVIE_IMAGE}:latest
                  docker tag ${COMPOSE_PROJECT_NAME}_cast_service:latest ${CAST_IMAGE}:${IMAGE_TAG}
                  docker tag ${COMPOSE_PROJECT_NAME}_cast_service:latest ${CAST_IMAGE}:latest
                """
            }
        }

        stage("Tests d'integration (docker-compose)") {
            when { expression { env.TARGET_NS != '' } }
            steps {
                sh 'docker-compose -f docker-compose.yml up -d'
                sh 'sleep 10'
                sh '''
                  echo "Verification movie-service..."
                  curl -fsS http://localhost:8001/api/v1/movies/docs > /dev/null

                  echo "Verification cast-service..."
                  curl -fsS http://localhost:8002/api/v1/casts/docs > /dev/null

                  echo "Smoke tests OK"
                '''
            }
            post {
                always {
                    sh 'docker-compose -f docker-compose.yml down -v || true'
                }
            }
        }

        stage('Push images vers Docker Hub') {
            when { expression { env.TARGET_NS != '' } }
            steps {
                withCredentials([string(credentialsId: env.DOCKER_HUB_PASS_CRED_ID, variable: 'DOCKER_HUB_PASS')]) {
                    sh 'echo $DOCKER_HUB_PASS | docker login -u $DOCKERHUB_NAMESPACE --password-stdin'
                }
                sh """
                  docker push ${MOVIE_IMAGE}:${IMAGE_TAG}
                  docker push ${MOVIE_IMAGE}:latest
                  docker push ${CAST_IMAGE}:${IMAGE_TAG}
                  docker push ${CAST_IMAGE}:latest
                """
            }
        }

        stage('Deploiement automatique (dev / qa / staging)') {
            when {
                allOf {
                    expression { env.TARGET_NS != '' }
                    expression { env.AUTO_DEPLOY == 'true' }
                }
            }
            steps {
                script {
                    deployToNamespace(env.TARGET_NS, env.MOVIE_NODEPORT, env.CAST_NODEPORT)
                }
            }
        }

        stage('Validation manuelle - PRODUCTION') {
            when {
                expression { env.IS_PROD == 'true' }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deployer en PRODUCTION (namespace prod) l'image taguee ${env.IMAGE_TAG} depuis la branche ${env.BRANCH_NAME} ?",
                          ok: 'Deployer en production'
                }
            }
        }

        stage('Deploiement - PRODUCTION') {
            when {
                expression { env.IS_PROD == 'true' }
            }
            steps {
                script {
                    deployToNamespace(env.TARGET_NS, env.MOVIE_NODEPORT, env.CAST_NODEPORT)
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline OK (branche: ${env.BRANCH_NAME}, namespace: ${env.TARGET_NS ?: 'aucun'})"
        }
        failure {
            echo "Echec du pipeline (branche: ${env.BRANCH_NAME})"
        }
    }
}

def deployToNamespace(String namespace, String moviePort, String castPort) {
    withCredentials([file(credentialsId: env.KUBECONFIG_CRED_ID, variable: 'KUBECONFIG')]) {
        sh """
          helm upgrade --install movie-service ${env.CHART_PATH} \
            --namespace ${namespace} --create-namespace \
            --set image.repository=${env.MOVIE_IMAGE} \
            --set image.tag=${env.IMAGE_TAG} \
            --set service.nodePort=${moviePort} \
            --wait --timeout 3m

          helm upgrade --install cast-service ${env.CHART_PATH} \
            --namespace ${namespace} --create-namespace \
            --set image.repository=${env.CAST_IMAGE} \
            --set image.tag=${env.IMAGE_TAG} \
            --set service.nodePort=${castPort} \
            --wait --timeout 3m
        """
    }
}