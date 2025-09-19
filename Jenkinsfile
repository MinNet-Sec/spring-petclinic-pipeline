pipeline {
  agent any

  tools {
    maven 'maven3'
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

    stage('Pre-Clean (OWASP cache)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: PRE-CLEAN (OWASP CACHE) ===\n\033[0m"
        // I remove leftover Dependency-Check data so Checkstyle(NoHttp) won’t flag http:// links.
        bat 'if exist ".dc-data" ( rmdir /s /q ".dc-data" )'
        bat 'if exist "target\\owasp-dc-data" ( rmdir /s /q "target\\owasp-dc-data" )'
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
          // I store Dependency-Check data under target/ so it won’t interfere with Checkstyle scans.
          bat """
            mvn -B org.owasp:dependency-check-maven:9.2.0:check ^
              -DskipTests ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -DdataDirectory=target\\owasp-dc-data ^
              -Dformats=HTML,XML,JSON ^
              -DfailOnError=true
          """
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true, allowEmptyArchive: true
        }
      }
    }
  }

  post {
    success {
      echo '\nBuild, tests, and checks succeeded.\n'
    }
    failure {
      echo '\nSomething failed — check the Console Output.\n'
    }
  }
}
