    def COLOR_MAP = [
            'SUCCESS': 'good',
            'FAILURE': 'danger'
    ]

    pipeline {
        agent any
        tools {
            maven "MAVEN3"
            jdk "OracleJDK8"
        }

        environment {
            SNAP_REPO = 'vprofile-snapshot'
            RELEASE_REPO = 'vprofile-release'
            CENTRAL_REPO = 'vpro-maven-central'
            NEXUS_GRP_REPO = 'vpro-maven-group'
            NEXUS_USER = 'admin'
            NEXUS_PASS = 'pinspire@1234'
            NEXUSIP = '10.0.0.152'
            NEXUSPORT = '8081'
            NEXUS_LOGIN = 'nexuslogin'
            SONARSERVER = 'sonarserver'
            SONARSCANNER = 'sonarscanner'
            registryCredential = 'ecr:us-east-2:awscreds'
            appRegistry = '943441234686.dkr.ecr.us-east-2.amazonaws.com/vprofileapp'
            vprofileRegistry ='https://943441234686.dkr.ecr.us-east-2.amazonaws.com'
        }

        stages {
            stage('Build') {
                steps {
                    sh 'mvn -s settings.xml -DskipTests install'
                }
                post {
                    success {
                        echo "Now Arhiving....."
                        archiveArtifacts artifacts: '**/*.war'
                    }
                }
            }

            stage('Test') {
                steps {
                    sh 'mvn test'
                }
            }

            stage('Checkstyle Analysis') {
                steps {
                    sh 'mvn -s settings.xml checkstyle:checkstyle'
                }
            }

            stage('Code Analysis With SonarQube') {
                environment {
                    scannerHome = tool "${SONARSCANNER}"
                }
                steps {
                    withSonarQubeEnv("${SONARSERVER}") {
                        sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                                        -Dsonar.projectName=vprofile-repo \
                                        -Dsonar.projectVersion=1.0 \
                                        -Dsonar.sources=src/ \
                                        -Dsonar.java.binaries=target \
                                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                                        -Dsonar.java.checkstyle.reportPaths=target/checkstyleresult.xml'''
                    }

                }


            }

//            stage('Quality Gate') {
//                steps {
//                    timeout(time: 01, unit: 'MINUTES') {
//                        waitForQualityGate abortPipeline:true
//                    }
//                }
//            }

            stage('Upload Artifact') {
                steps {
                    nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                            groupId: 'QA',
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: "${RELEASE_REPO}",
                            credentialsId: "${NEXUS_LOGIN}",
                            artifacts: [
                                    [artifactId: 'vproapp' ,
                                     classifier: '',
                                     file: 'target/vprofile-v2.war',
                                     type: 'war']
                            ]
                    )
                }


            }

            stage('Build App Image') {
                steps {
                    script {
                        dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "Docker-files/app/multistage/")
                    }
                }
            }

            stage('Upload App Image to ECR') {
                steps {
                    script {
                        docker.withRegistry( vprofileRegistry, registryCredential ) {
                            dockerImage.push("$BUILD_NUMBER")
                            dockerImage.push('latest')
                        }
                    }
                }
            }



        }
        post{
            always {
                echo 'Slack Notifications'
                slackSend channel: '  #pipeline-notification',
                        color: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
            }
        }


    }