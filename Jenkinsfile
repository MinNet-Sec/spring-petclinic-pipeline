pipeline {
  agent any
  tools { maven 'maven3' }
  options { timestamps(); ansiColor('xterm'); buildDiscarder(logRotator(numToKeepStr: '15')) }

  stages {
    stage('Checkout') { steps { checkout scm; bat 'java -version || ver'; bat 'mvn -v' } }
    stage('Pre-Clean (OWASP cache)') {
      steps {
        bat 'if exist ".dc-data" ( rmdir /s /q ".dc-data" )'
        bat 'if exist "target\\owasp-dc-data" ( rmdir /s /q "target\\owasp-dc-data" )'
      }
    }
    stage('Security (OWASP Dependency-Check)') {
      steps {
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
          bat """
            mvn -B org.owasp:dependency-check-maven:9.2.0:check ^
              -DskipTests ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -Dnvd.api.delay=300000 ^
              -Dformats=HTML,XML,JSON ^
              -DfailOnError=true
          """
        }
      }
    }
  }
}
