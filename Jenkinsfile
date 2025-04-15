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
        // Clone main repo
        git branch: params.BRANCH_NAME, url: env.NODE_REPO, changelog: false, poll: false, credentialsId: env.GIT_CREDENTIALS_ID

        // Create apps folder
        sh 'mkdir -p apps'

        // Clone each project into its own subdirectory
        dir('apps/tomcat') {
          git branch: params.BRANCH_NAME, url: env.TOMCAT_REPO, credentialsId: env.GIT_CREDENTIALS_ID
        }
        dir('apps/smartstore') {
          git branch: params.BRANCH_NAME, url: env.SMARTSTORE_REPO, credentialsId: env.GIT_CREDENTIALS_ID
        }
        dir('apps/gpt') {
          git branch: params.BRANCH_NAME, url: env.GPT_REPO, credentialsId: env.GIT_CREDENTIALS_ID
        }
        dir('apps/angular') {
          git branch: params.BRANCH_NAME, url: env.ANGULAR_REPO, credentialsId: env.GIT_CREDENTIALS_ID
        }
      }
      }
    }

    stage('Detect Changes') {
      steps {
        script {
          def changedApps = []
          
          changedApps += sh(script: "git diff --name-only HEAD~1 HEAD | grep nodejs && echo 'node'", returnStatus: true) == 0 ? ['node'] : []
          changedApps += sh(script: "git diff --name-only HEAD~1 HEAD | grep tomcat1 && echo 'java'", returnStatus: true) == 0 ? ['java'] : []
          changedApps += sh(script: "git diff --name-only HEAD~1 HEAD | grep smartstore && echo 'smartstore'", returnStatus: true) == 0 ? ['smartstore'] : []
          changedApps += sh(script: "git diff --name-only HEAD~1 HEAD | grep gpt && echo 'python'", returnStatus: true) == 0 ? ['python'] : []
          changedApps += sh(script: "git diff --name-only HEAD~1 HEAD | grep anguler && echo 'angular'", returnStatus: true) == 0 ? ['angular'] : []

          def deploymentMarkerPath = "/opt/tomcat/node" // You could replace this with an actual file or status check
          def alreadyDeployed = sh(script: "test -f ${deploymentMarkerPath} && echo 'yes'", returnStatus: true) == 0
          
          if (changedApps.isEmpty() && alreadyDeployed) {
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
          sshagent(credentials : ['ce2ee6d6-5457-4b1e-b0d1-7be789b0334a']){
             sh """
                  mkdir -p ~/.ssh
                  ssh-keyscan -H $server >> ~/.ssh/known_hosts
                  ssh -o StrictHostKeyChecking=no user3@$server "sudo chown -R user3:tomcat /opt/tomcat"
                 """
            env.CHANGED_APPS.split(',').each { app ->
            def activeSlot = sh(script: "ssh -o StrictHostKeyChecking=no user3@$server 'sudo readlink /opt/tomcat/${app}-current | grep blue && echo blue || echo green'", returnStdout: true).trim()
            def newSlot = (activeSlot == 'blue') ? 'green' : 'blue'
            def deployPath = "/opt/tomcat/${app}-${newSlot}"

            if (app == "node") {
              dir('apps/nodejs') {
                sh "npm install && npm run build"
                sh "ssh -o StrictHostKeyChecking=no user3@$server 'sudo mkdir -p ${deployPath}'"
                sh "scp -o StrictHostKeyChecking=no -r dist/* user3@$server:${deployPath}/"
              }
            } else if (app == "java") {
              dir('apps/tomcat1') {
                sh "mvn clean package"
                sh "ssh -o StrictHostKeyChecking=no user3@$server 'sudo mkdir -p ${deployPath}/webapps'"
                sh "scp -o StrictHostKeyChecking=no target/*.war user3@$server:${deployPath}/webapps/"
              }
            } else if (app == "python") {
              dir('apps/gpt') {
                sh "pip install -r requirements.txt"
                sh "ssh user3@$server 'sudo mkdir -p ${deployPath}'"
                sh "scp -r * user3@$server:${deployPath}/"
              }
            } else if (app == "angular") {
              dir('apps/anguler') {
                sh "npm install && npm run build"
                sh "ssh -o StrictHostKeyChecking=no user3@$server 'sudo mkdir -p ${deployPath}'"
                sh "scp -o StrictHostKeyChecking=no -r dist/* user3@$server:${deployPath}/"
              }
            } else if (app == "smartstore") {
              dir('apps/smartstore') {
                sh "npm install && npm run build"
                sh "ssh -o StrictHostKeyChecking=no user3@$server 'sudo mkdir -p ${deployPath}'"
                sh "scp -o StrictHostKeyChecking=no -r dist/* user3@$server:${deployPath}/"
              }
            }

            // def isHealthy = sh(script: "curl -sf http://${server}/${app}/health || echo unhealthy", returnStdout: true).trim() != "unhealthy"

            // if (isHealthy) {
            //   sh """
            //     ssh -o StrictHostKeyChecking=no user3@$server 'sudo ln -sfn ${deployPath} /opt/tomcat/${app}-current'
            //     // ssh -o StrictHostKeyChecking=no user3@$server 'sudo systemctl restart ${app}-service'
            //   """
            // } else {
            //   echo "Health check failed. Rolling back to ${activeSlot} for ${app}"
            //   sh """
            //     ssh -o StrictHostKeyChecking=no user3@$server 'sudo ln -sfn /opt/tomcat/${app}-${activeSlot} /opt/tomcat/${app}-current'
            //     // ssh -o StrictHostKeyChecking=no user3@$server 'sudo systemctl restart ${app}-service'
            //   """
            //   error("Rollback triggered for ${app} due to failed health check.")
            // }
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
