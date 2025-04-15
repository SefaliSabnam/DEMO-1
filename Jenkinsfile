pipeline {
  agent any

  environment {
    IMAGE_NAME = "sefali26/grafana-ec2"
    INSTANCE_NAME = "MONITORING-GRAFANA"
    REGION = "ap-south-1"
    DOCKER_HUB_CREDENTIALS = 'DOCKER_HUB_TOKEN'
    EC2_SSH_KEY = 'ec2-ssh-key'
    AWS_CREDENTIALS = 'AWS-DOCKER-CREDENTIALS'
  }

  options {
    skipStagesAfterUnstable()
    skipDefaultCheckout()
  }

  stages {

    stage('Clean Workspace') {
      steps {
        cleanWs()
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        bat "docker build -t %IMAGE_NAME% ."
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${DOCKER_HUB_CREDENTIALS}",
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          bat """
            echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin
            docker push %IMAGE_NAME%
          """
        }
      }
    }

    stage('Deploy to EC2') {
      when {
        branch 'main'
      }
      steps {
        withCredentials([
          sshUserPrivateKey(
            credentialsId: "${EC2_SSH_KEY}",
            keyFileVariable: 'KEY_FILE',
            usernameVariable: 'USER'
          )
        ]) {
          withAWS(credentials: "${AWS_CREDENTIALS}", region: "${REGION}") {
            script {
              def ec2_ip = bat(
                script: """
                  @echo off
                  for /f "tokens=* usebackq" %%i in (`aws ec2 describe-instances ^
                    --filters "Name=tag:Name,Values=${INSTANCE_NAME}" "Name=instance-state-name,Values=running" ^
                    --query "Reservations[*].Instances[*].PublicIpAddress" ^
                    --output text`) do (
                      echo %%i
                  )
                """,
                returnStdout: true
              ).trim()

              if (!ec2_ip || ec2_ip == 'None') {
                error("No running EC2 instance found with name '${INSTANCE_NAME}' in region '${REGION}'")
              }

              echo "EC2 Instance Public IP: ${ec2_ip}"

              // Fix key file permission for Windows Jenkins agent
              bat """
                icacls "%KEY_FILE%" /inheritance:r
                icacls "%KEY_FILE%" /grant:r "SYSTEM:F"
                icacls "%KEY_FILE%" /grant:r "Administrators:F"
              """

              // SSH into EC2 and deploy Docker container
              bat """
                set EC2_IP=${ec2_ip}
                echo Deploying to EC2: %EC2_IP%
                ssh -o StrictHostKeyChecking=no -i "%KEY_FILE%" %USER%@%EC2_IP% ^
                  "docker pull ${IMAGE_NAME} && docker stop grafana || true && docker rm grafana || true && docker run -d --name grafana -p 3000:3000 ${IMAGE_NAME}"
              """
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo ' Grafana deployed successfully. Access it via the EC2 public IP on port 3000.'
    }
    failure {
      echo ' Deployment failed. Check Jenkins logs for details.'
    }
  }
}
