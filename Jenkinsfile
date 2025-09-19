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

  // ✅ Security 전용 실행 토글
  parameters {
    booleanParam(name: 'SECURITY_ONLY', defaultValue: false,
      description: '체크하면 Checkout + Pre-Clean + OWASP Dependency-Check만 실행')
  }

  environment {
    SONAR_HOST_URL    = 'https://sonarcloud.io'
    SONAR_ORG         = 'minnet-sec'
    SONAR_PROJECT_KEY = 'MinNet-Sec_spring-petclinic-pipeline'

    // ✅ Dependency-Check 캐시 저장 위치(빌드마다 재사용)
    ODC_DATA_DIR      = "${WORKSPACE}\\odc-data"
  }

  // ✅ 멀티브랜치/일반 파이프라인 모두에서 동작하도록 함수화
  //    - 파라미터 또는 환경변수(SECURITY_ONLY=true) 둘 중 하나라도 true면 앞단 스킵
  //    - beforeAgent true 로 에이전트 띄우기 전 평가 → 시간 단축
  def secOnly = { ->
    return (params.SECURITY_ONLY?.toBoolean() ?: false) || (env.SECURITY_ONLY == 'true')
  }

  stages {
    stage('Checkout') {
      steps {
        echo "\033[44;1;37m\n=== ENTERING: CHECKOUT ===\n\033[0m"
        checkout scm
        // 디버깅용: 현재 토글 상태 확인
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
        // 캐시 폴더 보장
        bat 'if not exist "%ODC_DATA_DIR%" ( mkdir "%ODC_DATA_DIR%" )'
      }
    }

    stage('Build (Maven)') {
      when { beforeAgent true; expression { !secOnly() } }
      steps {
        echo "\033[44;1;37m\n=== ENTERING: BUILD (MAVEN) ===\n\033[0m"
        bat 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Test (JUnit)') {
      when { beforeAgent true; expression { !secOnly() } }
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
      when { beforeAgent true; expression { !secOnly() } }
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

          // (1) NVD 캐시만 갱신 — 네트워크 호출은 여기서만 수행
          bat """
            mvn -B org.owasp:dependency-check-maven:12.1.3:update-only ^
              -DdataDirectory="%ODC_DATA_DIR%" ^
              -Dnvd.api.key=%NVD_API_KEY% ^
              -Dnvd.api.delay=16000 ^
              -DfailOnError=false
          """

          // (2) 캐시로만 스캔 — 네트워크 차단
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
