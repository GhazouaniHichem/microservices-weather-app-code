pipeline {
    agent any
    
    
    environment{
        SCANNER_HOME= tool 'sonar-scanner'
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
        
/*        stage('Code Build') {
            steps {
                dir('client') {
                    sh "npm install"
                }
                dir('server') {
                    sh "npm install"
                }
            }
        }
*/        
/*         stage('Run Test Cases') {
            steps {
                dir('client') {
                    sh "npm run test"
                }
                dir('server') {
                    sh "npm run test" 
                }
            }
        }
*/        
        stage('Sonarqube Analysis') {
            steps {
                dir('UI') {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-App \
                        -Dsonar.sources=. \
                        -Dsonar.css.node=. \
                        -Dsonar.projectKey=NodeJS-App '''
                    }
                }
                dir('auth') {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=auth \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=**/*_test.go,**/vendor/**,**/testdata/* \
                        -Dsonar.projectKey=auth '''
                    }
                }
                dir('weather') {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=weather \
                        -Dsonar.sources=. \
                        -Dsonar.language=py \
                        -Dsonar.projectKey=weather '''
                    }
                }
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dir('auth') { 
                   dependencyCheck additionalArguments: '--scan ./   ', odcInstallation: 'DPCHECK'
                   dependencyCheckPublisher pattern: '**//*dependency-check-report.xml'           
                }
                dir('UI') { 
                   dependencyCheck additionalArguments: '--scan ./   ', odcInstallation: 'DPCHECK'
                   dependencyCheckPublisher pattern: '**//*dependency-check-report.xml'           
                }
                dir('weather') { 
                   dependencyCheck additionalArguments: '--scan ./   ', odcInstallation: 'DPCHECK'
                   dependencyCheckPublisher pattern: '**//*dependency-check-report.xml'           
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
}