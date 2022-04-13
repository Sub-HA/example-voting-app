pipeline {
    agent none

    stages {
// Result stages
        stage('result-build') {
            agent {
                docker {
                    image 'node:8.16.0-alpine'
                }
            }
            when {
                changeset "**/result/**"
            }
            steps {
                dir('result'){
                    sh "npm install"
                }
            }
        }
        stage('result-test') {
            agent {
                docker {
                    image 'node:8.16.0-alpine'
                }
            }
            when {
                changeset "**/result/**"
            }
            steps {
                dir('result'){
                    sh "npm install"
                    sh "npm test"
                }
            }
        }
        stage('result-docker-package') {
            agent any
            when {
                branch 'master'
                changeset "**/result/**"
            }
            steps {
                echo 'Packaging result app with docker'
                script {
                    docker.withRegistry('https://docker.hyca.io/', 'nexus-login') {
                        def resultImage = docker.build("docker.hyca.io/result:v${env.BUILD_ID}", "./result")
                        resultImage.push()
                        resultImage.push("latest")
                    }
                }
            }
        }

        stage('vote-build') {
            agent {
                docker {
                    image 'python:2.7.16-slim'
                    args '--user root'
                }
            }
            when {
                changeset "**/vote/**"
            }
            steps {
                echo 'Compile vote app'
                dir('vote'){
                    sh "pip install -r requirements.txt"
                }
            }
        }
        stage('vote-test') {
            agent {
                docker {
                    image 'python:2.7.16-slim'
                    args '--user root'
                }
            }
            when {
                changeset "**/vote/**"
            }
            steps {
                echo 'Running Unit Tests on vote app'
                dir('vote'){
                    sh "pip install -r requirements.txt"
                    sh "nosetests -v"
                }
            }
        }
        stage('Vote Integration') {
            agent any
            when {
                changeset "**/vote/**"
                branch 'master'
            }
            steps {
                echo 'Running Integration Tests on vote app'
                dir('vote'){
                    sh "integration_test.sh"
                }
            }
        }
        stage('vote-docker-package') {
            agent any
            when {
                branch 'master'
                changeset "**/vote/**"
            }
            steps {
                echo 'Packaging vote app with docker'
                script {
                    docker.withRegistry('https://docker.hyca.io/', 'nexus-login') {
                        def voteImage = docker.build("docker.hyca.io/vote:v${env.BUILD_ID}", "./vote")
                        voteImage.push()
                        voteImage.push("latest")
                    }
                }
            }
        }

//Worker stages
        stage('worker-build') {
            agent {
                docker {
                    image 'maven:3.6.1-jdk-8-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            when {
                changeset "**/worker/**"
            }
            steps {
                echo 'Compile worker app'
                dir('worker'){
                    sh "mvn compile"
                }
            }
        }
        stage('worker-test') {
            agent {
                docker {
                    image 'maven:3.6.1-jdk-8-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            when {
                changeset "**/worker/**"
            }
            steps {
                echo 'Running Unit Tests on worker app'
                dir('worker'){
                    sh "mvn clean test"
                }
            }
        }
        stage('worker-package') {
                agent {
                docker {
                    image 'maven:3.6.1-jdk-8-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            when {
                branch 'master'
                changeset "**/worker/**"
            }
            steps {
                echo 'Packageing worker app'
                dir('worker'){
                    sh "mvn package -DskipTests"
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }
        stage('worker-docker-package') {
            agent any
            when {
                branch 'master'
                changeset "**/worker/**"
            }
            steps {
                echo 'Packaging worker app with docker'
                script {
                    docker.withRegistry('https://docker.hyca.io/', 'nexus-login') {
                        def workerImage = docker.build("docker.hyca.io/worker:v${env.BUILD_ID}", "./worker")
                        workerImage.push()
                        workerImage.push("$env.BRANCH_NAME")
                        workerImage.push("latest")
                    }
                }
            }
        }

        stage('Sonarqube') {
            agent any
/*            when{
                branch 'master'
            }
*/
            tools {
                jdk "jdk-11" // the name you have given the JDK installation in Global Tool Configuration
            }

            environment{
                sonarpath = tool 'SonarScanner'
            }

            steps {
                    echo 'Running Sonarqube Analysis..'
                    withSonarQubeEnv('sonar-instavote') {
                    sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                    }
            }
            }

        stage("Quality Gate") {
            agent any
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                    }
            }
        }

        stage('deploy to dev'){
          agent any
          when{
            branch 'master'
          }
          steps{
            echo 'Deploy instavote app with docker compose'
            sh 'docker-compose up -d'
          }
      }
    }
    post {
        always{
            echo 'Build pipeline for worker is complete..'
        }
    }
}