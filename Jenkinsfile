pipeline {
    agent any

    environment {
        WORKSPACE = "/home/solar/jenkins-prod/${env.JOB_NAME}"
        NEXUS_CREDENTIAL_ID = 'Nexus-Credential'
        // Add other environment variables as needed
    }

    tools {
        maven 'localMaven'
        jdk 'localJdk'
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
            post {
                success {
                    echo 'Archiving artifacts'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('Checkstyle Code Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Checkstyle analysis results'
                }
            }
        }

        stage('SonarQube Inspection') {
            steps {
                withSonarQubeEnv('SonarQube') { 
                    withCredentials([string(credentialsId: 'NHL-SonarQube-Token', variable: 'NHL-SonaQube-Token')]) {
                        sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=demo \
                          -Dsonar.host.url=http://10.0.0.115:9000 \
                          -Dsonar.login=38d4894edbc5eab1cc29f705e67fa1e3e0f4884b
                        """
                    }
                }
            }
        }

        stage("Nexus Artifact Uploader") {
            steps {
                script {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: 'http://10.0.0.116:8081', // Corrected to use full URL with protocol
                        groupId: 'webapp',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: 'maven-project-releases',
                        credentialsId: "${NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [
                                artifactId: 'webapp',
                                classifier: '',
                                file: "${WORKSPACE}/webapp/target/webapp.war",
                                type: 'war'
                            ]
                        ]
                    )
                }
            }
        }

        stage('Deploy to Development Env') {
            environment {
                HOSTS = 'dev'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
                }
            }
        }

        // Add more stages as needed

    }

    post {
        always {
            echo 'Sending Slack Notifications.'
            slackSend(
                channel: '#cicd-pipeline-project-alerts-3', // Update with your channel name
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info: ${env.BUILD_URL}"
            )
        }
    }
}

