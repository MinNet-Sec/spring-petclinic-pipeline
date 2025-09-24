pipeline {
  // Run this pipeline on any available Jenkins agent node
  agent any

  tools {
    // Use Maven tool (configured as 'maven3' in Jenkins global tool settings)
    maven 'maven3'
  }

  options {
    timestamps()
    // Enable ANSI color codes in console log for colored output
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '15')) // Keep only the last 15 build records
  }

  parameters {
    booleanParam(
      name: 'SKIP_STYLE',
      defaultValue: true,
      description: 'Skip Checkstyle and nohttp checks during CI'
    )
    // SKIP_STYLE parameter:
    // This option allows skipping Checkstyle and nohttp style checks.
    // These tools enforce coding standards, but they can slow down the build
    // or even fail it for minor style issues.
    // For this educational pipeline, the default is 'true' so the build runs
    // faster and more reliably. Set to 'false' to enable full style checks.
  }

  environment {
    SONAR_HOST_URL    = 'https://sonarcloud.io'
    SONAR_ORG         = 'minnet-sec'
    SONAR_PROJECT_KEY = 'MinNet-Sec_spring-petclinic-pipeline'
    ODC_DATA_DIR      = "${WORKSPACE}\\odc-data"  // Local folder for OWASP Dependency-Check cache/data
  }

  stages {
    stage('Checkout') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CHECKOUT ===\n\033[0m"
        checkout scm
        echo "SKIP_STYLE: ${params.SKIP_STYLE}"
        bat 'java -version || ver'
        bat 'mvn -v'
      }
    }

    stage('Pre-Clean (OWASP cache)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: PRE-CLEAN (OWASP CACHE) ===\n\033[0m"
        bat 'if exist ".dc-data" ( rmdir /s /q ".dc-data" )'
        bat 'if exist "target\\owasp-dc-data" ( rmdir /s /q "target\\owasp-dc-data" )'
        bat 'if not exist "%ODC_DATA_DIR%" ( mkdir "%ODC_DATA_DIR%" )'
      }
    }

    stage('Build (Maven)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: BUILD (MAVEN) ===\n\033[0m"
        // Run Maven in batch mode (-B), skip tests (-DskipTests), and optionally skip style checks
        bat """
          mvn -B -DskipTests ${params.SKIP_STYLE ? '-Dcheckstyle.skip=true -Dnohttp.check.skip=true' : ''} clean package
        """
        // Archive the generated JAR into Jenkins
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Test (JUnit)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: TEST (JUNIT) ===\n\033[0m"
        // Execute unit tests; optionally skip style checks
        bat """
          mvn -B test ${params.SKIP_STYLE ? '-Dcheckstyle.skip=true -Dnohttp.check.skip=true' : ''} 
        """
      }
      post {
        always {
          // Publish JUnit test results so Jenkins can display them in the UI
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Code Quality (SonarCloud)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CODE QUALITY (SONARCLOUD) ===\n\033[0m"
        // Use SonarQube environment settings (credentials, URL, etc.)
        withSonarQubeEnv('SonarCloud') {
          // Optionally skip style checks while sending analysis to SonarCloud
          bat """
            mvn -B -DskipTests ${params.SKIP_STYLE ? '-Dcheckstyle.skip=true -Dnohttp.check.skip=true' : ''} sonar:sonar ^
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
        // Securely load the NVD API key from Jenkins Credentials
        withCredentials([string(credentialsId: 'NVD_API_KEY', variable: 'NVD_API_KEY')]) {
          // (1) Update the local NVD/CVE cache into ODC_DATA_DIR (reduces rate-limit issues)
          bat """
            mvn -B org.owasp:dependency-check-maven:12.1.3:update-only ^
              -DdataDirectory="%ODC_DATA_DIR%" ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -Dnvd.api.delay=16000 ^
              -DfailOnError=false
          """
          // (2) Run the vulnerability scan USING ONLY the local cache (no network call)
          bat """
            mvn -B org.owasp:dependency-check-maven:12.1.3:check ^
              -DautoUpdate=false ^
              -DdataDirectory="%ODC_DATA_DIR%" ^
              -Dformats=HTML,XML,JSON ^
              -Danalyzer.ossIndex.enabled=false ^
              -DfailOnError=false
          """
        }
      }
      post {
        always {
          // Archive generated reports so they are downloadable from Jenkins
          archiveArtifacts artifacts: 'target/dependency-check-report.*', fingerprint: true, allowEmptyArchive: true
        }
      }
    }

    stage('Deploy (Local CMD)') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: DEPLOY (LOCAL CMD) ===\n\033[0m"
        bat '''
        echo %DATE% %TIME% - Deploy stage sanity check
        echo WORKSPACE=%WORKSPACE%
        dir "target"
        '''
      }
    }
  }

  post {
    success { 
      echo "\033[42;1;37m\n=== PIPELINE COMPLETED SUCCESSFULLY ===\n\033[0m"
    }
    failure { 
      echo "\033[41;1;37m\n=== PIPELINE FAILED — SEE CONSOLE OUTPUT FOR DETAILS ===\n\033[0m"
    }
  }
}
