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
                sh 'detect-secrets -C . scan --disable-plugin HexHighEntropyString Base64HighEntropyString > .secrets.baseline '
            }
        }   

        stage('Code Build') {
            steps {
                dir('microservices/UI') {
                    sh "npm install"
                }

                dir('microservices/auth/src/main') {
                    sh "go build"
                }
            }
        }  

        stage('Project Dependency-Check') {
            steps {

                dir('microservices/auth') { 
                    script {
                        sh 'syft . -o cyclonedx-json=auth.sbom.cdx.json'
                        sh 'grype sbom:./auth.sbom.cdx.json'
                    }
                }
                dir('microservices/UI') { 
                    script {
                        sh 'syft . -o cyclonedx-json=UI.sbom.cdx.json'
                        sh 'grype sbom:./UI.sbom.cdx.json'
                    }         
                }
                dir('microservices/weather') { 
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

                dir('microservices/UI') {
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
                dir('microservices/auth') {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=auth-Golang-App \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/*_test.go,**/vendor/**,**/testdata/* \
                            -Dsonar.inclusions=**/*.go \
                            -Dsonar.language=go \
                            -Dsonar.projectKey=auth-Golang-App '''
                        }
                        timeout(time: 1, unit: 'HOURS') { 
                            def qg = waitForQualityGate() 
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
                dir('microservices/weather') {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=weather-Python-App \
                            -Dsonar.sources=. \
                            -Dsonar.language=py \
                            -Dsonar.projectKey=weather-Python-App '''
                        }
                        timeout(time: 1, unit: 'HOURS') { 
                            def qg = waitForQualityGate() 
                            if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }
      

        stage('Scan Dockerfiles with Checkov') {
             steps {
                 script {
                     docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                         try {
                             sh 'checkov --file microservices/auth/Dockerfile -o cli -o junitxml --output-file-path console,auth-check-results.xml'
                             junit skipPublishingChecks: true, testResults: 'auth-check-results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'auth-check-results.xml'
                             throw err
                         }
                         try {
                             sh 'checkov --file microservices/UI/Dockerfile -o cli -o junitxml --output-file-path console,UI-check-results.xml'
                             junit skipPublishingChecks: true, testResults: 'UI-check-results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'UI-check-results.xml'
                             throw err
                         }
                         try {
                             sh 'checkov --file microservices/weather/Dockerfile -o cli -o junitxml --output-file-path console,weather-check-results.xml'
                             junit skipPublishingChecks: true, testResults: 'weather-check-results.xml'
                         } catch (err) {
                             junit skipPublishingChecks: true, testResults: 'weather-check-results.xml'
                             throw err
                         }
                     }
                 }
             }
         }
       
        stage('Docker Build Images') {
            steps {
                   script {
                        dir('microservices/auth') {
                            sh "docker build -t weatherapp-auth ."
                        }
                        dir('microservices/UI') {
                            sh "docker build -t weatherapp-ui ."
                        }
                        dir('microservices/weather') {
                            sh "docker build -t weatherapp-weather ."
                        }
                   } 
            }
        }
        
        stage('Scan Docker Images') {
            steps {
                    sh "trivy image weatherapp-auth"
                    sh "trivy image weatherapp-ui"
                    sh "trivy image weatherapp-weather"
            }
        }

        stage('Docker Push Images to DockerHub') {
            steps {
                    
                    script {

                        int x = "${BUILD_NUMBER}" as Integer; 
                        def TAG = "${x/10}"


                        withDockerRegistry([ credentialsId: 'docker-cred', url: '' ]) {
                            dir('microservices/auth') {
                                sh "docker tag weatherapp-auth ghazouanihm/weatherapp-auth:${TAG}"
                                sh "docker push ghazouanihm/weatherapp-auth:${TAG}"
                            }
                            dir('microservices/UI') {
                                sh "docker tag weatherapp-ui ghazouanihm/weatherapp-ui:${TAG}"
                                sh "docker push ghazouanihm/weatherapp-ui:${TAG}"
                            }
                            dir('microservices/weather') {
                                sh "docker tag weatherapp-weather ghazouanihm/weatherapp-weather:${TAG}"
                                sh "docker push ghazouanihm/weatherapp-weather:${TAG}"
                            }
                        }
                   } 
            }
        }

        stage('Deploy argocd applictions') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubernetes-config']) {
                        sh 'kubectl apply -f argocd-applications'
                    }
                }
            }
        }

        stage('Trigger gitops pipeline') {
            int x = "${BUILD_NUMBER}" as Integer; 
            def TAG = "${x/10}"
            steps {
                build job: 'weather-app-gitops-pipeline', parameters: [
                string(name: 'IMAGE_TAG', value: "${TAG}")
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