pipeline {

    agent any
/*
	tools {
        maven "maven3"
    }
*/
    environment {
        registry = "chibykiss/vproappdock"
        //ARTVERSION = "${env.BUILD_ID}"
        registryCredentials = 'dockerhub'
        sonarToken = credentials('kube-sonar-token')
    }

    stages{

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    /*sh '''${scannerHome}/bin/sonar-scanner */
                    sh '''sonar-scanner \
                    -Dsonar.projectKey=vibetek-analysis_kubecicd \
                   -Dsonar.organization=vibetek-analysis \
                   -Dsonar.sources=src/ \
                   -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.login=$sonarToken
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('BUILD DOCKER APP IMAGE'){
            steps{
                script{
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage('UPLOAD IMAGE'){
            steps{
                script{
                    docker.withRegistry('registryCredentials'){
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push('Latest')
                    }
                }
            }
        }

        stage('REMOVE UNUSED DOCKER IMAGE'){
            steps{
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }

        stage('KUBERNETES DEPLOY'){
            agent {label 'KOPS'}
            steps{
                sh "helm upgrade --install  --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }


}