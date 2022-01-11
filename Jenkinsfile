pipeline {
    agent any
    stages {
        stage ('Build Backend') {
            steps {
                sh 'mvn clean package -DskipTest=true'
            }
        }
    }
    stages {
        stage ('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
    }
}