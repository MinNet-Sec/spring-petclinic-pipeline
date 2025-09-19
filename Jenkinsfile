// Big, colored banner at each stage
def stageBanner(String name) {
  def line = '═' * 80
  echo "\u001B[1;97;44m${line}\n>>> ENTERING STAGE: ${name.toUpperCase()} <<<\n${line}\u001B[0m"
}

pipeline {
  agent any

  tools {
    maven 'maven3'  // e.g., Maven 3.9.x
  }

  options {
    timestamps()
    ansiColor('xterm')   // colored console output
  }

  environment {
    // SonarCloud
    SONAR_HOST_URL    = 'https://sonarcloud.io'
    SONAR_PROJECT_KEY = 'MinNet-Sec_spring-petclinic-pipeline' // your project key
    SONAR_ORG         = 'minnet-sec'                           // your org key (lowercase)

    // OWASP Dependency-Check local data cache (relative to workspace; Windows path)
    DC_DATA_DIR = '.\\.dc-data'
  }

  stages {
    stage('Checkout') {
      steps {
        script { stageBanner('Checkout') }
        checkout scm
      }
    }

    stage('Build (Maven)') {
      steps {
        script { stageBanner('Build (Maven)') }
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Test (JUnit)') {
      steps {
        script { stageBanner('Test (JUnit)') }
        bat 'mvn -B test'
      }
      post {
        always { junit 'target/surefire-reports/*.xml' }
      }
    }

    stage('Code Quality (SonarCloud)') {
      steps {
        script { stageBanner('Code Quality (SonarCloud)') }
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
        script { stageBanner('Security (OWASP Dependency-Check)') }
        bat """
          mvn -B org.owasp:dependency-check-maven:9.2.0:check ^
            -DfailBuildOnCVSS=7 ^
            -DdataDirectory="%DC_DATA_DIR%"
        """
      }
      post {
        always {
          // Requires the Jenkins "OWASP Dependency-Check" plugin
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml', stopBuild: false
          archiveArtifacts artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }
  }

  post {
    success { echo 'Build, Test, Code Quality, Security all succeeded' }
    failure { echo 'Something failed — check Console Output' }
  }
}
