pipeline {
    agent any

    tools {
        nodejs 'node'
        go 'go-1.20'
    }

    stages {

        stage('Cleanup Workspace'){
            steps {
                script {
                    cleanWs()
                }
            }
        }

        stage('Git Checkout ') {
            steps {
                git credentialsId: 'github', branch: 'main', changelog: false, poll: false, url: 'https://github.com/GhazouaniHichem/microservices-weather-app-code.git'
            }
        }

        stage('Scan code to detect secrets') {
            steps {
                sh 'detect-secrets scan . --all-files'
            }
        }   

        stage('Code Build') {
            steps {
                dir('UI') {
                    sh "npm install"
                }

                dir('auth/src/main') {
                    sh "go build"
                }
            }
        }  

        stage('Snyk Dependency Check') {
            steps {

                dir('auth') { 
                    script {
                        sh 'syft . -o cyclonedx-json=auth.sbom.cdx.json'
                        sh 'grype sbom:./auth.sbom.cdx.json'
                    }
                }
                dir('UI') { 
                    script {
                        sh 'syft . -o cyclonedx-json=UI.sbom.cdx.json'
                        sh 'grype sbom:./UI.sbom.cdx.json'
                    }         
                }
                dir('weather') { 
                    script {
                        sh 'syft . -o cyclonedx-json=weather.sbom.cdx.json'
                        sh 'grype sbom:./weather.sbom.cdx.json'
                    }          
                }       
                
            }
        }

        stage('Sonarqube Analysis') {

            environment{
                SCANNER_HOME= tool 'sonar-scanner'
            }

            steps {
                dir('UI') {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            sh ''' $SCANNER_HOME/bin/sonar-scanner \
                            -Dsonar.projectName=UI-NodeJS-App \
                            -Dsonar.sources=. \
                            -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info \
                            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                            -Dsonar.projectKey=UI-NodeJS-App '''
                        }
                        timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
                            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
                dir('auth') {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=auth-Golang-App \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/*_test.go,**/vendor/**,**/testdata/* \
                            -Dsonar.inclusions=**/*.go \
                            -Dsonar.language=go \
                            -Dsonar.projectKey=auth-Golang-App '''
                        }
                        timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
                            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
                dir('weather') {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=weather-Python-App \
                            -Dsonar.sources=. \
                            -Dsonar.language=py \
                            -Dsonar.projectKey=weather-Python-App '''
                        }
                        timeout(time: 1, unit: 'HOURS') { // Just in case something goes wrong, pipeline will be killed after a timeout
                            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }
      
/*      stage('Run Unit Tests') {
            steps {

            }
        } */

        stage('Scan Dockerfiles with Checkov') {
             steps {
                 script {
                     docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                         try {
                             sh 'checkov --file auth/Dockerfile -o cli -o junitxml --output-file-path console,results.xml'
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                             throw err
                         }
                         try {
                             sh 'checkov --file UI/Dockerfile -o cli -o junitxml --output-file-path console,results.xml'
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                             throw err
                         }
                         try {
                             sh 'checkov --file weather/Dockerfile -o cli -o junitxml --output-file-path console,results.xml'
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'results.xml'
                             throw err
                         }
                     }
                 }
             }
         }
       
        stage('Docker Build & Push') {
            steps {
                   script {
                       withDockerRegistry([ credentialsId: 'docker-cred', url: '' ]) {
                            dir('auth') {
                                sh "docker build -t weatherapp-auth ."
                                sh "docker tag weatherapp-auth ghazouanihm/weatherapp-auth:${BUILD_NUMBER}"
                                sh "docker push ghazouanihm/weatherapp-auth:${BUILD_NUMBER}"
                            }
                            dir('UI') {
                                sh "docker build -t weatherapp-ui ."
                                sh "docker tag weatherapp-ui ghazouanihm/weatherapp-ui:${BUILD_NUMBER}"
                                sh "docker push ghazouanihm/weatherapp-ui:${BUILD_NUMBER}"
                            }
                            dir('weather') {
                                sh "docker build -t weatherapp-weather ."
                                sh "docker tag weatherapp-weather ghazouanihm/weatherapp-weather:${BUILD_NUMBER}"
                                sh "docker push ghazouanihm/weatherapp-weather:${BUILD_NUMBER}"
                            }
                        }
                   } 
            }
        }
        
        stage('Docker Image scan') {
            steps {
                    sh "trivy image ghazouanihm/weatherapp-auth:${BUILD_NUMBER}"
                    sh "trivy image ghazouanihm/weatherapp-ui:${BUILD_NUMBER}"
                    sh "trivy image ghazouanihm/weatherapp-weather:${BUILD_NUMBER}"
            }
        }

        stage('Apply argocd appliction file') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubernetes-config']) {
                        sh 'kubectl apply -f gitops'
                    }
                }
            }
        }

        stage('Trigger gitops pipeline') {
            environment {
                IMAGE_TAG = "${BUILD_NUMBER}"
            }
            steps {
                build job: 'weather-app-gitops-pipeline', parameters: [
                string(name: 'IMAGE_TAG', value: "${env.IMAGE_TAG}")
                ]
            }
        }   
    }

    post {
        always {
            script {
                def status = currentBuild.result ?: 'UNKOWN'
                def color
                switch(status) {
                    case 'SUCCESS':
                        color = 'good'
                        break
                    case 'Failure':
                        color = 'danger'
                        break
                    default:
                        color = 'warning'
                    
                }
                slackSend(channel: '#devops', message: "Update Deployment ${status.toLowerCase()} for ${env.JOB_NAME} (${env.BUILD_NUMBER}) - ${env.BUILD_URL}", iconEmoji: ':jenkins:', color: color)
            }
        }
    }
}