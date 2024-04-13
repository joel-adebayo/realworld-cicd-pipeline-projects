def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
    'UNSTABLE': 'danger'
]
pipeline {
  agent any
  environment {
    WORKSPACE = "${env.WORKSPACE}"
    NEXUS_CREDENTIAL_ID = 'Nexus-Credentials'
    SONAR_TOKEN = 'sonarQube-token'
    //NEXUS_USER = "$NEXUS_CREDS_USR"
    //NEXUS_PASSWORD = "$Nexus-Token"
    //NEXUS_URL = "172.31.18.62:8081"
    //NEXUS_REPOSITORY = "maven_project"
    //NEXUS_REPO_ID    = "maven_project"
    //ARTVERSION = "${env.BUILD_ID}"
  }
  tools {
    maven 'M3'
    jdk 'localJDK'
  }
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package'
      }
      post {
        success {
          echo ' now Archiving '
          archiveArtifacts artifacts: '**/*.war'
        }
      }
    }
    stage('Unit Test'){
        steps {
            sh 'mvn test'
        }
    }
    stage('Integration Test'){
        steps {
          sh 'mvn verify -DskipUnitTests'
        }
    }
    stage ('Checkstyle Code Analysis'){
        steps {
            sh 'mvn checkstyle:checkstyle'
        }
        post {
            success {
                echo 'Generated Analysis Result'
            }
        }
    }
    stage('SonarQube Inspection') {
        steps {
            withSonarQubeEnv('SonarQube') { 
                withCredentials([string(credentialsId: 'sonarQube-token', variable: 'SONAR_TOKEN')]) {
                sh """
                mvn sonar:sonar \
                -Dsonar.projectKey=maven-java-webapp \
                -Dsonar.host.url=https://172.31.87.210:9000 \
                -Dsonar.login=$SONAR_TOKEN
                """
                }
            }
        }
    }
    stage('SonarQube GateKeeper') {
        steps {
          timeout(time : 1, unit : 'HOURS'){
          waitForQualityGate abortPipeline: true
          }
       }
    }
    stage("Nexus Artifact Uploader"){
        steps{
           nexusArtifactUploader(
              nexusVersion: 'nexus3',
              protocol: 'http',
              nexusUrl: '172.31.80.65:8081',
              groupId: 'webapp',
              version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
              repository: 'maven-project-release',  //"${NEXUS_REPOSITORY}",
              credentialsId: "${NEXUS_CREDENTIAL_ID}",
              artifacts: [
                  [artifactId: 'webapp',
                  classifier: '',
                  file: "${WORKSPACE}/webapp/target/webapp.war",
                  type: 'war']
              ]
           )
        }
    }
    stage('Deploy to Development Env') {
        environment {
            HOSTS = 'dev'
        }
        steps {
            withCredentials([usernamePassword(credentialsId: 'Ansible-Credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
            }
        }
    }
    stage('Deploy to Staging Env') {
        environment {
            HOSTS = 'stage'
        }
        steps {
            withCredentials([usernamePassword(credentialsId: 'Ansible-Credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
            }
        }
    }
    stage('Quality Assurance Approval') {
        steps {
            input('Do you want to proceed?')
        }
    }
    stage('Deploy to Production Env') {
        environment {
            HOSTS = 'prod'
        }
        steps {
            withCredentials([usernamePassword(credentialsId: 'Ansible-Credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
                sh "ansible-playbook -i ${WORKSPACE}/ansible-config/aws_ec2.yaml ${WORKSPACE}/deploy.yaml --extra-vars \"ansible_user=$USER_NAME ansible_password=$PASSWORD hosts=tag_Environment_$HOSTS workspace_path=$WORKSPACE\""
            }
         }
      }
   }
  post {
    always {
        echo 'Slack Notifications.'
        slackSend channel: '#cicd-pipeline-project-jadebayo', //update and provide your channel name
        color: COLOR_MAP[currentBuild.currentResult],
        message: "*${currentBuild.currentResult}:* Job Name '${env.JOB_NAME}' build ${env.BUILD_NUMBER} \n Build Timestamp: ${env.BUILD_TIMESTAMP} \n Project Workspace: ${env.WORKSPACE} \n More info at: ${env.BUILD_URL}"
    }
  }
}

