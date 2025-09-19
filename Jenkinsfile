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
    NVD_API_KEY = credentials('NVD_API_KEY')
  }
  stages {
    stage('Security (OWASP Dependency-Check)') {
      steps {
        echo "\033
