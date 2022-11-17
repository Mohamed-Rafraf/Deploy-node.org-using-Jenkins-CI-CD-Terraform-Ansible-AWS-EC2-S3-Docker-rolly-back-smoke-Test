pipeline {
  agent any

  environment {
    registry = "hossamalsankary/nodejs_app"
    registryCredential = 'docker_credentials'
    ANSIBLE_PRIVATE_KEY = credentials('secritfile')
  }
  stages {

    stage("install dependencies") {

      steps {

        sh 'npm install'

      }

    }
    stage("Test") {

      steps {

        sh 'npm run  test:unit'

      }

    }

    stage("Build") {

      steps {

        sh 'npm run build'
      }

    }
    stage("Build Docker Image") {
      steps {

        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }

    stage("push image to docker hup") {
      steps {
        script {
          docker.withRegistry('', registryCredential) {
            dockerImage.push()
          }
        }
      }
    }
    stage("Smoke Test ") {
      steps {

        sh ' docker run --name test_$BUILD_NUMBER -d -p 5000:8080 $registry:$BUILD_NUMBER '
        sh 'sleep 2'
        sh 'curl localhost:5000'
      }
      post {

        success {
          sh 'docker stop test_$BUILD_NUMBER '
          sh 'docker system prune --volumes -a -f '
        }
        failure {
          sh 'docker system prune --volumes -a -f '
        }
      }

    }

    stage("Deply IAC ") {
      steps {
        withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          dir("terraform-aws-instance") {
            sh 'terraform init'
            sh 'terraform destroy --auto-approve'
            sh 'terraform apply --auto-approve'
          }
        }
      }
      post {

        success {
          echo "we  successful deploy IAC"
        }
        failure {
          withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {

            dir("terraform-aws-instance") {
              sh 'terraform destroy --auto-approve'

            }
          }
        }
      }
    }
    stage("ansbile") {
      steps {
        dir("./terraform-aws-instance") {
          withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {

            sh 'ansible-playbook -i ansbile/inventory/inventory --extra-vars ansible_ssh_host=$(terraform output  -raw server_ip) --extra-vars  IMAGE_NAME=$registry:$BUILD_NUMBER --private-key=$ANSIBLE_PRIVATE_KEY ./ansbile/inventory/deploy.yml '

          }
        }
      }
    }

  }

  post {

    success {

      echo "========A executed successfully========"
      sh "docker stop test_$BUILD_NUMBER &&docker system prune --volumes -a -f "
    }
    failure {
      echo "========A execution failed========"
      sh "docker stop test_$BUILD_NUMBER system prune --volumes -a -f "

    }
  }
}