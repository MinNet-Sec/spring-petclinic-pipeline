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
      name: 'SECURITY_ONLY',
      defaultValue: false,
      description: 'Checkout + Pre-Clean + OWASP Dependency-Check만 실행'
    )
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
        echo "SECURITY_ONLY(param/env): ${params.SECURITY_ONLY} / ${env.SECURITY_ONLY}"
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
      when {
        beforeAgent true      // Evaluate the condition BEFORE allocating an agent
        expression {
          //  when SECURITY_ONLY is true, skip Build/Test/CodeQuality
          !( (params.SECURITY_ONLY?.toBoolean() ?: false) || (env.SECURITY_ONLY == 'true') )
        }
      }
      steps {
        echo "\033[44;1;37m\n=== ENTERING: BUILD (MAVEN) ===\n\033[0m"
        // Run Maven in batch mode (-B), skip tests (-DskipTests), clean old build and package the app
        // This generates the executable JAR file for the project
        bat 'mvn -B -DskipTests clean package' //  skip test on build stage
        // Archive the generated JAR into Jenkins
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true  // allows tracking the file across builds and jobs
      }
    }

    stage('Test (JUnit)') {
      when {
        beforeAgent true
        expression {
          !( (params.SECURITY_ONLY?.toBoolean() ?: false) || (env.SECURITY_ONLY == 'true') )
        }
      }
      steps {
        echo "\033[44;1;37m\n=== ENTERING: TEST (JUNIT) ===\n\033[0m"
        // Run Maven tests in batch mode (-B = no interactive input)
        // This executes all unit tests written with JUnit
        bat 'mvn -B test'
      }
      post {
        always {
          // Publish JUnit test results so Jenkins can display them in the UI
          // XML reports are located in target/surefire-reports
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Code Quality (SonarCloud)') {
      when {
        beforeAgent true
        expression {
          !( (params.SECURITY_ONLY?.toBoolean() ?: false) || (env.SECURITY_ONLY == 'true') )
        }
      }
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CODE QUALITY (SONARCLOUD) ===\n\033[0m"
        // Use SonarQube environment settings (credentials, URL, etc.)
        // Then run Maven with the Sonar plugin to analyze the project
        withSonarQubeEnv('SonarCloud') {
          // Run Maven in batch mode, skip tests, trigger Sonar analysis
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
          // (2)  Run the vulnerability scan USING ONLY the local cache (no network call)
          bat """
            mvn -B org.owasp:dependency-check-maven:12.1.3:check ^
              -DautoUpdate=false ^
              -DdataDirectory="%ODC_DATA_DIR%" ^
              -Dformats=HTML,XML,JSON ^
              -DfailOnError=true
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
  }

  post {
    success { echo '\nBuild, tests, and checks succeeded.\n' }
    failure { echo '\nSomething failed — check the Console Output.\n' }
  }
}
