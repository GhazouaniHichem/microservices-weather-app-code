pipeline {
    agent any
    
    tools {
        jdk 'jdk11'
        maven 'maven3'
    }
    
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
                git credentialsId: 'github', branch: 'main', changelog: false, poll: false, url: 'https://github.com/GhazouaniHichem/MERN-app-CI-CD-pipeline.git'
            }
        }
        
        stage('Code Build') {
            steps {
                dir('client') {
                    sh "npm install"
                }
                dir('server') {
                    sh "npm install"
                }
            }
        }
        
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
                dir('client') {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-App \
                        -Dsonar.sources=. \
                        -Dsonar.css.node=. \
                        -Dsonar.projectKey=NodeJS-App '''
                    }
                }
                dir('server') {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=NodeJS-App \
                        -Dsonar.sources=. \
                        -Dsonar.css.node=. \
                        -Dsonar.projectKey=NodeJS-App '''
                    }
                }
            }
        }
        
/*         stage('OWASP Dependency Check') {
            steps {
                dir('server') { 
                   dependencyCheck additionalArguments: '--scan ./   ', odcInstallation: 'DP'
                   dependencyCheckPublisher pattern: '**//*dependency-check-report.xml'           
                }
            }
        }
*/        
        stage('Docker Build & Push') {
            steps {
                   script {
                       withDockerRegistry([ credentialsId: 'docker-cred', url: '' ]) {
                            dir('client') {
                                sh "docker build -t movies-mern-app-frontend ."
                                sh "docker tag webapp ghazouanihm/movies-mern-app-frontend:${BUILD_NUMBER}"
                                sh "docker push ghazouanihm/movies-mern-app-frontend:${BUILD_NUMBER}"
                            }
                            dir('server') {
                                sh "docker build -t movies-mern-app-backend ."
                                sh "docker tag webapp ghazouanihm/movies-mern-app-backend:${BUILD_NUMBER}"
                                sh "docker push ghazouanihm/movies-mern-app-backend:${BUILD_NUMBER}"
                            }
                        }
                   } 
            }
        }
        
        stage('Docker Image scan') {
            steps {
                    sh "trivy image ghazouanihm/movies-mern-app-frontend:${BUILD_NUMBER}"
                    sh "trivy image ghazouanihm/movies-mern-app-backend:${BUILD_NUMBER}"
            }
        }

        stage('Trigger gitops pipeline') {
            environment {
                IMAGE_TAG = "${BUILD_NUMBER}"
            }
            steps {
                build job: 'mern-gitops-pipeline', parameters: [
                string(name: 'IMAGE_TAG', value: "${env.IMAGE_TAG}")
                ]
            }
        }

        
    }
}