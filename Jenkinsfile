pipeline {
  agent any
  tools {
    maven 'maven3'   // Maven tool configured in Jenkins Global Tool Configuration
  }
  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15'))
  }
  environment {
    SONAR_HOST_URL    = 'https://sonarcloud.io'
    SONAR_ORG         = 'minnet-sec'
    SONAR_PROJECT_KEY = 'MinNet-Sec_spring-petclinic-pipeline'
  }
  stages {
    stage('Checkout') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CHECKOUT ===\n\033[0m"
        checkout scm
        bat 'java -version || ver'
        bat 'mvn -v'
      }
    }
    stage('Build (Maven)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: BUILD (MAVEN) ===\n\033[0m"
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }
    stage('Test (JUnit)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: TEST (JUNIT) ===\n\033[0m"
        bat 'mvn -B test'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
        }
      }
    }
    stage('Code Quality (SonarCloud)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CODE QUALITY (SONARCLOUD) ===\n\033[0m"
        withSonarQubeEnv('SonarCloud') {
          bat """
            mvn -B -DskipTests sonar:sonar ^
              -Dsonar.projectKey=%SONAR_PROJECT_KEY% ^
              -Dsonar.organization=%SONAR_ORG% ^
              -Dsonar.host.url=%SONAR_HOST_URL%
          """
        }
      }
    }
    stage('Security (OWASP Dependency-Check)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: SECURITY (OWASP DEPENDENCY-CHECK) ===\n\033[0m"
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
          bat """
            mvn -B org.owasp:dependency-check-maven:9.2.0:check ^
              -DskipTests ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -Dformats=HTML,XML,JSON ^
              -DfailOnError=true
          """
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true, allowEmptyArchive: true
          // Uncomment if plugin installed:
          // dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        }
      }
    }
  }
  post {
    success {
      echo '\nBuild & Tests & Checks succeeded.\n'
    }
    failure {
      echo '\nSomething failed — check the Console Output.\n'
    }
  }
}
