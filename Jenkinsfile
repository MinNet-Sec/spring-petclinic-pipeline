pipeline {
  agent any
    
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build (Maven)') {
      steps {
        // Since this is a Windows agent, use bat instead of sh
        bat 'mvn -B -DskipTests clean package'
        // Message to verify build artifact
        echo 'Build artifact should be under target/*.jar'
      }
    }
      
    stage('Test (JUnit)') {
      steps {
        bat 'mvn -B test'
      }
      post {
        always {
          // Collect JUnit reports (records failures as well)
          junit 'target/surefire-reports/*.xml'
        }
      }
    }
  }
  post {
    success { echo 'Build & Test succeeded ' }
    failure { echo 'Build or Test failed  — check Console Output' }
  }
}
