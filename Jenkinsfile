pipeline {
    agent any
    stages {
        stage ('Check-Git-Secrets') {
            steps {
                sh 'rm -f trufflehog'
                sh 'docker run --rm dxa4481/trufflehog --json https://github.com/vanutimascarenhas/tasks-backend.git | tee trufflehog'
                sh 'docker run --rm dxa4481/trufflehog --json https://github.com/vanutimascarenhas/tasks-frontend.git | tee trufflehog'
                sh 'docker run --rm dxa4481/trufflehog --json https://github.com/vanutimascarenhas/tasks-api-test.git | tee trufflehog'
                sh 'docker run --rm dxa4481/trufflehog --json https://github.com/vanutimascarenhas/tasks-functional-tests.git | tee trufflehog'
                sh 'ls -la'
                sh 'cat trufflehog'
                script {
                    def exitCode = sh script: 'cat trufflehog | grep -q branch ; echo $?', returnStatus: true
                    boolean existeSecrets = exitCode == 0
                    println exitCode
                    println existeSecrets
                    if (existeSecrets) {
                        currentBuild.result = 'ABORTED'
                        error('echo Secrets identificadas no codigo-fonte')
                    }
                }
                
            }
        }
        stage ('Build Backend') {
            steps {
                sh 'mvn cleans packages -DskipTests'
            }
        }
        stage ('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage ('Sonar Analysis') {
            environment {
                scannerHome = tool 'SONAR_SCANNER'
            }
            steps {
                withSonarQubeEnv('SONAR_LOCAL') {
                    sh "${scannerHome}/bin/sonar-scanner -e -Dsonar.projectKey=DeployBack -Dsonar.host.url=http://192.168.72.130:9000 -Dsonar.login=564d57baabda2404bb56ba49f549b3a9a6af0fdb -Dsonar.java.binaries=target -Dsonar.coverage.exclusions=**/.mvn/**,**/src/test/**,**/model/**,**Application.java"
                }
            }
        }
        stage ('Quality Gate') {
            steps {
                sleep(20)
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true, credentialsId: 'SonarToken'
                }
            }
        }
        stage ('Deploy Backend') {
            steps {
                deploy adapters: [tomcat8(credentialsId: 'TomcatLogin', path: '', url: 'http://localhost:8001/')], contextPath: 'tasks-backend', war: 'target/tasks-backend.war'
            }
        }
        stage ('API Test') {
            steps {
                dir('api-test') {
                    git credentialsId: 'github_login', url: 'https://github.com/vanutimascarenhas/tasks-api-test'
                    sh 'mvn test'
                }
            }
        }
        stage ('Deploy Frontend') {
            steps {
                dir('frontend') {
                    git credentialsId: 'github_login', url: 'https://github.com/vanutimascarenhas/tasks-frontend'
                    sh 'mvn clean package'
                    deploy adapters: [tomcat8(credentialsId: 'TomcatLogin', path: '', url: 'http://localhost:8001/')], contextPath: 'tasks', war: 'target/tasks.war'
                }
            }   
        }
        stage ('Functional Test') {
            steps {
                dir('functional-test') {
                    git credentialsId: 'github_login', url: 'https://github.com/vanutimascarenhas/tasks-functional-tests'
                    sh 'mvn test'
                }
            }
        }
        stage ('Deploy Prod') {
            steps {
                sh 'docker-compose build'
                sh 'docker-compose up -d'
            }
        }
        stage ('Health Check') {
            steps {
                sleep(20)
                dir('functional-test') {
                    sh 'mvn verify -Dskip.surefire.tests'
                }
            }
        }
    } 
    post {
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml, api-test/target/surefire-reports/*.xml, functional-test/target/surefire-reports/*.xml, functional-test/target/failsafe-reports/*.xml'
            archiveArtifacts artifacts: 'target/tasks-backend.war, frontend/target/tasks.war', followSymlinks: false, onlyIfSuccessful: true
        }
        unsuccessful {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build $BUILD_NUMBER has failed', to: 'vanuti.testes+jenkins@gmail.com'
        }
        fixed {
            emailext attachLog: true, body: 'See the attached log below', subject: 'Build is fixed!', to: 'vanuti.testes+jenkins@gmail.com'
        }
    }  
}