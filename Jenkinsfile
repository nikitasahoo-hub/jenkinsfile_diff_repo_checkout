pipeline {
  agent any

  environment {
    GIT_USER = "devops"
    GIT_BASE = "http://devops@192.168.80.135/code"
    NODE_REPO = "${GIT_BASE}/nodejs.git"
    TOMCAT_REPO = "${GIT_BASE}/tomcat1.git"
    SMARTSTORE_REPO = "${GIT_BASE}/smartstore.git"
    GPT_REPO = "${GIT_BASE}/gpt.git"
    ANGULAR_REPO = "${GIT_BASE}/anguler.git"
    GIT_CREDENTIALS_ID = "5a9f069e-d5ba-49ce-958d-0966e5d1ff37"
  }

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'dev', description: 'Git Branch Name')
  }

  stages {
    stage('Clone Repositories') {
      steps {
        script {
          git branch: params.BRANCH_NAME, url: env.NODE_REPO, changelog: false, poll: false, credentialsId: '5a9f069e-d5ba-49ce-958d-0966e5d1ff37'
          sh 'mkdir -p apps && mv * apps/ || true'
          dir('apps') {
            git branch: params.BRANCH_NAME, url: env.TOMCAT_REPO, credentialsId: '5a9f069e-d5ba-49ce-958d-0966e5d1ff37'
            git branch: params.BRANCH_NAME, url: env.SMARTSTORE_REPO, credentialsId: '5a9f069e-d5ba-49ce-958d-0966e5d1ff37'
            git branch: params.BRANCH_NAME, url: env.GPT_REPO, credentialsId: '5a9f069e-d5ba-49ce-958d-0966e5d1ff37'
            git branch: params.BRANCH_NAME, url: env.ANGULAR_REPO, credentialsId: '5a9f069e-d5ba-49ce-958d-0966e5d1ff37'
          }
        }
      }
    }

    stage('Detect Changes') {
      steps {
        script {
          def changedApps = []
          
          changedApps += sh(script: "git diff --name-only origin/main...HEAD | grep nodejs && echo 'node'", returnStatus: true) == 0 ? ['node'] : []
          changedApps += sh(script: "git diff --name-only origin/main...HEAD | grep tomcat1 && echo 'java'", returnStatus: true) == 0 ? ['java'] : []
          changedApps += sh(script: "git diff --name-only origin/main...HEAD | grep smartstore && echo 'smartstore'", returnStatus: true) == 0 ? ['smartstore'] : []
          changedApps += sh(script: "git diff --name-only origin/main...HEAD | grep gpt && echo 'python'", returnStatus: true) == 0 ? ['python'] : []
          changedApps += sh(script: "git diff --name-only origin/main...HEAD | grep anguler && echo 'angular'", returnStatus: true) == 0 ? ['angular'] : []


          if (changedApps.isEmpty()) {
            currentBuild.result = 'ABORTED'
            error "No relevant changes to build or deploy."
          } else {
            env.CHANGED_APPS = changedApps.join(',')
            echo "Detected changed apps: ${env.CHANGED_APPS}"
          }
        }
      }
    }

    stage('Build and Deploy') {
      steps {
        script {
          def server = params.BRANCH_NAME == 'main' ? '192.168.80.60' :
                       params.BRANCH_NAME == 'qa' ? '192.168.80.101' :
                       '192.168.80.61'

          env.CHANGED_APPS.split(',').each { app ->
            def activeSlot = sh(script: "ssh user@$server 'readlink /opt/tomcat/${app}-current | grep blue && echo blue || echo green'", returnStdout: true).trim()
            def newSlot = (activeSlot == 'blue') ? 'green' : 'blue'
            def deployPath = "/opt/tomcat/${app}-${newSlot}"

            if (app == "node") {
              dir('apps/nodejs') {
                sh "npm install && npm run build"
                sh "ssh user@$server 'mkdir -p ${deployPath}'"
                sh "scp -r dist/* user@$server:${deployPath}/"
              }
            } else if (app == "java") {
              dir('apps/tomcat1') {
                sh "mvn clean package"
                sh "ssh user@$server 'mkdir -p ${deployPath}/webapps'"
                sh "scp target/*.war user@$server:${deployPath}/webapps/"
              }
            } else if (app == "python") {
              dir('apps/gpt') {
                sh "pip install -r requirements.txt"
                sh "ssh user@$server 'mkdir -p ${deployPath}'"
                sh "scp -r * user@$server:${deployPath}/"
              }
            } else if (app == "angular") {
              dir('apps/anguler') {
                sh "npm install && npm run build"
                sh "ssh user@$server 'mkdir -p ${deployPath}'"
                sh "scp -r dist/* user@$server:${deployPath}/"
              }
            } else if (app == "smartstore") {
              dir('apps/smartstore') {
                sh "npm install && npm run build"
                sh "ssh user@$server 'mkdir -p ${deployPath}'"
                sh "scp -r dist/* user@$server:${deployPath}/"
              }
            }

            def isHealthy = sh(script: "curl -sf http://${server}/${app}/health || echo unhealthy", returnStdout: true).trim() != "unhealthy"

            if (isHealthy) {
              sh """
                ssh user@$server 'ln -sfn ${deployPath} /opt/tomcat/${app}-current'
                ssh user@$server 'systemctl restart ${app}-service'
              """
            } else {
              echo "Health check failed. Rolling back to ${activeSlot} for ${app}"
              sh """
                ssh user@$server 'ln -sfn /opt/tomcat/${app}-${activeSlot} /opt/tomcat/${app}-current'
                ssh user@$server 'systemctl restart ${app}-service'
              """
              error("Rollback triggered for ${app} due to failed health check.")
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deployment completed successfully."
    }
    failure {
      echo "❌ Deployment failed. Manual intervention might be required."
    }
  }
}
