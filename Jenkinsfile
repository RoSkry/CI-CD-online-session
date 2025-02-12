pipeline {
  agent any
  stages {
    stage('Check Code Quality') {
      steps {
          script {
            docker.image('python:3.9').inside {c ->
              sh '''
              ls -la
              pwd
              mount
              python -m venv .venv
              . .venv/bin/activate
              pip install pylint
              pip install -r requirements.txt
              pylint --exit-zero --report=y --output-format=json:pylint-report.json,colorized ./*.py
              '''
              publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: './',
                    reportFiles: 'pylint-report.json',
                    reportName: 'pylint Scan',
                    reportTitles: 'pylint Scan'
                ]
            }
          }
      }
    }
    
    stage('build') {
      steps {
        script {
          checkout scm
          def customImage = docker.build("${registry}:${env.BUILD_ID}")
        }

      }
    }
    
    stage('unit-test') {
      steps {
        script {
          docker.image("${registry}:${env.BUILD_ID}").inside {c ->
          sh 'python app_test.py'}
        }

      }
    }
    
    stage('http-test') {
      steps {
        script {
          docker.image("${registry}:${env.BUILD_ID}").withRun('-p 9005:9000') {c ->
          sh "sleep 5; curl -i http://localhost:9005/test_string"}
        }

      }
      post {
         success {
            echo 'Code quality check passed'
         }
         failure {
            echo 'Code quality check failed'
         }
     }
    }
    
    stage('Scan Docker Image for Vulnerabilities') {
      steps {
        script {
          sh 'mkdir -p reports'
          sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl > html.tpl'
          def vulnerabilities = sh(script: "trivy image --ignore-unfixed --exit-code 0 --severity HIGH,MEDIUM,LOW --format template --template '@html.tpl' -o reports/image-scan.html --no-progress ${registry}:${env.BUILD_ID}", returnStdout: true).trim()
          echo "Vulnerability Report:\n${vulnerabilities}"
          publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'image-scan.html',
                    reportName: 'Trivy Scan',
                    reportTitles: 'Trivy Scan'
                ]
          
          sh "trivy image --ignore-unfixed --exit-code 1 --severity CRITICAL --no-progress ${registry}:${env.BUILD_ID}"
        }
      }
    }

    stage('publish') {
      steps {
        script {
          docker.withRegistry('', 'dockerhub_id') {
            docker.image("${registry}:${env.BUILD_ID}").push('latest')
            docker.image("${registry}:${env.BUILD_ID}").push("${env.BUILD_ID}")
          }
        }

      }
    }
    stage('Deploy') {
      steps {
        sh 'docker stop flask-app || true; docker rm flask-app || true;docker run -d --name flask-app -p 9000:9000 rostdocker/flask-app:latest'
      }
    }
    
    stage('Validation') {
      steps {
        sh 'curl -i http://localhost:9000/test_string'
      }
    }

  }
  environment {
    registry = 'rostdocker/flask-app'
  }
}
