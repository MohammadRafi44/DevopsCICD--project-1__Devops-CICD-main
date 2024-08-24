pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
        // docker 'docker-latest'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        // DOCKERHUB_CREDENTIALS = credentials('jenkins-docker-id')
    }

    stages {
        stage('GIT-CHECKOUT') {
            steps {
                checkout changelog: false, poll: false, scm: scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/MohammadRafi44/devops-cicd-main.git']])
            }
        }
        
        stage('CODE-COMPILE') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('SONARQUBE-ANALYSIS'){
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName-Devops-CICD \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=Devops-CICD '''
                }
            }
        }
        stage('TRIVY-SCAN'){
            steps {
                sh 'trivy fs --security-checks vuln,config /var/lib/jenkins/workspace/devops-cicd-pipeline'
                // sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('CODE-BUILD'){
            steps {
                sh " mvn clean install"
            }
        }

        stage('DOCKER-BUILD'){
            steps {
                withDockerRegistry(credentialsId: 'jenkins-docker-id', url: 'https://index.docker.io/v1/') {
                    sh "docker build -t cicd-devops ."
                }
            }
        }

        stage('DOCKER-PUBLISH'){
            steps {
                withDockerRegistry(credentialsId: 'jenkins-docker-id', url: 'https://index.docker.io/v1/') {
                    sh "docker tag cicd-devops mohammadrafi44/cicd-devops:$BUILD_ID"
                    sh "docker push mohammadrafi44/cicd-devops:$BUILD_ID"
                }
            }
        }
        stage("DOCKER-IMAGE-CLEANUP"){
            steps {
                script {
                    echo 'docker images cleanup started'
                    sh 'docker system prune -af'
                    echo 'docker images cleanup finished'
                }
            }
        }
        stage("K8S-DEPLOY"){
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-token-for-jenkins', namespace: 'jenkins', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.4.114:6443') {
                        sh "sed -i 's|mohammadrafi44/cicd-devops:.*|mohammadrafi44/cicd-devops:${BUILD_ID}|' deploymentservice.yaml"
                        sh "kubectl apply -f deploymentservice.yaml"
                        sleep 20
                    }
                }
            }
        }
        stage("VERIFY-DEPLOYMENT"){
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s-token-for-jenkins', namespace: 'jenkins', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.4.114:6443') {
                        sh "kubectl get pods"
                        sh "kubectl get svc"
                    }
                }
            }
        }
    }
}
