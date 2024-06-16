def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]

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
                    withCredentials([string(credentialsId: 'NHL-SonarQube-Token', variable: 'NHL_SONARQUBE_TOKEN')]) {
                        sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=demo \
                          -Dsonar.host.url=http://10.0.0.115:9000 \
                          -Dsonar.login=${NHL_SONARQUBE_TOKEN}
                        """
                    }
                }
            }
        }

        stage("Nexus Artifact Uploader") {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'webapp', classifier: '', file: '${env.WORKSPACE}/webapp/target/webapp.war"', type: 'war']], credentialsId: 'Nexus-Credential', groupId: 'webapp', nexusUrl: '10.0.0.116:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'maven-project-releases', version: '${env.BUILD_ID}-${env.BUILD_TIMESTAMP}'
            }
        }

        stage('Deploy to Development Env') {
            environment {
                HOSTS = 'dev'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'Ansible-Credential', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                    sh "ansible-playbook -i ${env.WORKSPACE}/ansible-config/aws_ec2.yaml ${env.WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=${env.WORKSPACE}\""
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