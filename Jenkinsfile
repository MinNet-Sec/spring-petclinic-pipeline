pipeline {
  agent any
  tools { maven 'maven3' }  // Maven name registered in Manage Jenkins > Tools
  options {
    timestamps()
    
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build (Maven)') {
      steps {
        // Skip tests and generate only the JAR artifact
        bat 'mvn -B -DskipTests clean package'
        // Archive the generated JAR (can be checked under Artifacts on the left side of the console)
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }
    stage('Test (JUnit)') {
      steps {
        bat 'mvn -B test'
      }
      post {
        always {
          // Collect JUnit reports created by Maven Surefire
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
