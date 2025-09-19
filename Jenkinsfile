pipeline {
  agent any
  tools { maven 'maven3' }

  options {
    timestamps()          
    // ansiColor('xterm')  
  }

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
  }

  post {
    success { echo 'Build & Test succeeded ' }
    failure { echo 'Build or Test failed  — check Console Output' }
  }
}
