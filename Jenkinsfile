// ===============================
// Jenkins Pipeline for Spring PetClinic
// OS: Windows agent (uses `bat` steps)
// Stages: Checkout → Build → Test → Code Quality (SonarCloud) → Security (OWASP DC)
// Tools used: Maven, JUnit (via Surefire), SonarCloud, OWASP Dependency-Check
// Prereqs (see notes at bottom of this file).
// ===============================

pipeline {
  agent any

  tools {
    maven 'maven3'   // Manage Jenkins → Tools → Maven installations 
  }

  options {
    timestamps()                 
    ansiColor('xterm')          
    buildDiscarder(logRotator(numToKeepStr: '15')) // keep last 15 builds
  }

  environment {
   
    SONAR_HOST_URL     = 'https://sonarcloud.io'
    SONAR_ORG          = 'minnet-sec'                       
    SONAR_PROJECT_KEY  = 'MinNet-Sec_spring-petclinic-pipeline' 
  }

  stages {

    stage('Checkout') {
      steps {
        echo '\n================== ENTERING: CHECKOUT ==================\n'
        checkout scm
        // Quick visibility: show current Java & Maven versions
        bat 'java -version || ver'
        bat 'mvn -v'
      }
    }

    stage('Build (Maven)') {
      steps {
        echo '\n================== ENTERING: BUILD (MAVEN) ==================\n'
        // -DskipTests so packaging is quick; we run tests in the next stage
        bat 'mvn -B -DskipTests clean package'
        // Save the built jar as a build artifact
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Test (JUnit)') {
      steps {
        echo '\n================== ENTERING: TEST (JUNIT) ==================\n'
        bat 'mvn -B test'
      }
      post {
        always {
          // Publish JUnit results to Jenkins Test Report
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Code Quality (SonarCloud)') {
      steps {
        echo '\n================== ENTERING: CODE QUALITY (SONARCLOUD) ==================\n'
  
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
        echo '\n================== ENTERING: SECURITY (OWASP DEPENDENCY-CHECK) ==================\n'
        // Requires: OWASP Dependency-Check Jenkins plugin installed
        // Also requires an NVD API key stored as a Jenkins Secret Text credential with ID "NVD_API_KEY"
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
          // Keep the XML/HTML/JSON reports with the build
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true, allowEmptyArchive: true
          // Publish the XML report to Jenkins UI (requires the Jenkins plugin)
          dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
        }
      }
    }
  }

  post {
    success {
      echo '\n Build & Tests & Checks succeeded.\n'
    }
    failure {
      echo '\n Something failed — check the Console Output.\n'
    }
  }
}


