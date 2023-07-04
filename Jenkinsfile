pipeline {

    agent any
	tools {
        maven "Maven3"
        jdk "Oracle JDK8"
    }
    environment {
        registry= "manishbg/vproappdock"
        registryCredential = "dockerhub"
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
                                scannerHome = tool 'sonar4.8'
                            }

                            steps {
                                withSonarQubeEnv('sonar-pro') {
                                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                                   -Dsonar.projectName=vprofile-repo \
                                   -Dsonar.projectVersion=1.0 \
                                   -Dsonar.sources=src/ \
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

        stage("Build App Image") {
            steps {
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }

        }

        stage("Upload Image") {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }


        }

        stage("Remove unused docker image") {
            steps {
                sh "docker rmi $registry:$BUILD_NUMBER"
            }

        }

        stage("Kubernetes deploy") {
            agent {label "KOPS"}
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=$registry:$BUILD_NUMBER --namespace prod"
            }
        }

    }

    post {
                 always {
                    archiveArtifacts artifacts: 'target/*.war', onlyIfSuccessful: true
                    emailext to: "manishbhagat280@gmail.com",
                    subject: "jenkins build: ${currentBuild.currentResult}: ${env.JOB_NAME}",
                    body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}",
                    attachmentsPattern: '*.war'
                 }
            }


}
