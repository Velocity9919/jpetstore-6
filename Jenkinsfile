pipeline{
    agent any
    tools {
        jdk 'jdk11'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {
        stage ('clean Workspace'){
            steps{
                cleanWs()
            }
        }

        stage ('checkout scm') {
            steps {
                git branch: 'master', url: 'https://github.com/Velocity9919/jpetstore-6.git'
            }
        }

        stage ('maven compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage ('maven test') {
            steps {
                sh 'mvn test'
            }
        }

        stage ('maven package'){
            steps{
                sh 'mvn clean package -DskipTests=true'
            }
        }

        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Petshop \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Petshop '''
                }
            }
        }

        stage("quality gate"){
            steps {
                script {
                  waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            }
        }

        stage("OWASP Dependency Check"){
            steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage ('Build and push to docker hub'){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'dockerhub-cred', toolName: 'docker') {
                        sh "docker build -t petshop ."
                        sh "docker tag petshop nareshbabu1991/petshop:latest"
                        sh "docker push nareshbabu1991/petshop:latest"
                    }
                }
            }
        }

        stage('TRIVY'){
            steps{
                sh "trivy image nareshbabu1991/petshop:latest > trivy.txt"
            }
        }

        stage('deploy to tomcat'){
            steps{
                sshagent(['tomcat-privatekey']) {
                    sh "scp -o StrictHostKeyChecking=no target/jpetstore.war ubuntu@172.31.41.55:8080:/opt/tomcat/webapps"
                }
            }
            post {
                always {
                    emailext attachLog: true,
                    subject: "'${currentBuild.result}'",
                    body: "Project: ${env.JOB_NAME}<br/>" +
                    "Build Number: ${env.BUILD_NUMBER}<br/>" +
                    "URL: ${env.BUILD_URL}<br/>",
                    to: 'ynareshbabu1991@gmail.com',
                    attachmentsPattern: 'trivy.txt'
                }
            }
        }
    }
}
