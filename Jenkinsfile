pipeline {
  agent any
  tools { maven 'maven3' }

  options { timestamps() }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build (Maven)') {
      steps {
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Test (JUnit)') {
      steps { bat 'mvn -B test' }
      post { always { junit 'target/surefire-reports/*.xml' } }
    }


    stage('Code Quality (SonarCloud)') {
      steps {
        withSonarQubeEnv('SonarCloud') {   //  name from Manage Jenkins > System 
          bat '''
            mvn -B -DskipTests sonar:sonar ^
              -Dsonar.projectKey=MinNet-Sec_spring-petclinic-pipeline ^
              -Dsonar.organization=MinNet-Sec ^
              -Dsonar.host.url=https://sonarcloud.io
          '''
        }
      }
    }
  }

  post {
    success { echo 'Build, Test, Code Quality all succeeded ' }
    failure { echo 'Something failed — check Console Output ' }
  }
}
