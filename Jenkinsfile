pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "mdhussain27"
        DOCKER_CRED_ID = 'Docker-Creds'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        SONAR_HOST_URL = "http://13.234.172.166:9000"
        NOTIFICATION_EMAIL = "mdhussainfaizan7@gmail.com"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github-token-auth', url: 'https://github.com/mdhussainfaizan7-arch/Automated-ci-cd-pipeline.git'
            }
        }

        stage("Build & Test Application") {
            steps {
                sh "mvn clean test package"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'SonarQube-token') { 
                        sh "mvn sonar:sonar -Dsonar.host.url=${SONAR_HOST_URL}"
                    }
                }    
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-token'
                }    
            }
        }

        // --- JFrog Artifactory stages omitted since no Maven JFrog repository was created ---
        // Uncomment and configure once you create your JFrog Artifactory repository:
        /*
        stage('Artifactory Configuration') {
            steps {
                rtServer (
                    id: "jfrog-server",
                    url: "http://13.234.172.166:8082/artifactory",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )      
            }
        }

        stage('Deploy Artifacts') {
            steps {
                rtMavenRun (
                    tool: "maven",
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }

        stage('Publish Build Info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "jfrog-server"
                )
            }
        }
        */

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CRED_ID) {
                        def docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernete') {
                        kubeconfig(credentialsId: 'kubernetes', serverUrl: '') {
                            sh 'kubectl apply -f regapp-deploy.yml'
                            sh 'kubectl apply -f regapp-service.yml'
                            sh 'kubectl rollout restart deployment.apps/regapp-deployment'
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            emailext (
                body: '''${SCRIPT, template="groovy-html.template"}''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed ❌", 
                mimeType: 'text/html',
                to: "${NOTIFICATION_EMAIL}"
            )
        }
        success {
            emailext (
                body: '''<html>
                    <body>
                        <h2>Build Successful! ✅</h2>
                        <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>Build Status:</strong> SUCCESS</p>
                        <p><strong>Docker Image:</strong> ${IMAGE_NAME}:${IMAGE_TAG}</p>
                        <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    </body>
                </html>''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful ✅", 
                mimeType: 'text/html',
                to: "${NOTIFICATION_EMAIL}"
            )
        }
    }
}
