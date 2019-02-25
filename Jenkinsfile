class Globals {
  static boolean userInput = true
  static boolean didTimeout = false
}

pipeline {
  agent none
  stages {
    stage('Linting') {
      agent any
      steps {
        sh 'make lint'
      }
      post {
        always {
          sh 'make stop'
        }
      }
    }

    stage('Test') {
      agent any
      steps {
        sh 'make test'
      }
      post {
        always {
          sh 'make stop'
        }
      }
    }


    stage('Acceptance Tests') {
      agent none
      when {
        branch 'master'
        beforeAgent true
      }
      steps {
        build 'govwifi-acceptance-tests/master'
      }
    }

    stage('Publish stable tag') {
      agent any
      when {
        branch 'master'
        beforeAgent true
      }

      steps {
        publishStableTag()
      }
    }

    stage('Deploy to staging') {
      agent any
      when{
        branch 'master'
        beforeAgent true
      }

      steps {
        deploy_staging()
      }
    }

    stage('Confirm deploy to production') {
      agent none
      when {
        branch 'master'
        beforeAgent true
      }
      steps {
        wait_for_input('production')
      }
    }


    stage('Deploy to production') {
      agent any
      when {
        branch 'master'
        beforeAgent true
      }

      steps {
        deploy_production()
      }
    }
  }
}

def wait_for_input(deploy_environment) {
  if (deployCancelled()) {
    return;
  }
  try {
    timeout(time: 5, unit: 'MINUTES') {
      input "Do you want to deploy to ${deploy_environment}?"
    }
  } catch (err) {
    def user = err.getCauses()[0].getUser()

    if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
      Globals.didTimeout = true
      echo "Release window timed out, to deploy please re-run"
    } else {
      Globals.userInput = false
      echo "Aborted by: [${user}]"
    }
  }
}

def deploy_staging() {
  deploy('staging')
}

def deploy_production() {
  if(deployCancelled()) {
    return
  }
  deploy('production')
}

def deploy(deploy_environment) {
  echo "${deploy_environment}"

  sh('git fetch')
  sh('git checkout stable')

  docker.withRegistry(env.AWS_ECS_API_REGISTRY) {
    sh("eval \$(aws ecr get-login --no-include-email)")
    def appImage = docker.build(
      "govwifi/authorisation-api:${deploy_environment}",
      "--build-arg BUNDLE_INSTALL_CMD='bundle install --without test' ."
    )
    appImage.push()
  }

  if(deploy_environment == 'production') {
    cluster_name = 'wifi-api-cluster'
    service_name = 'authorisation-api-service-wifi'
    regions = ['eu-west-1', 'eu-west-2']
  } else {
    cluster_name = 'staging-api-cluster'
    service_name = 'authorisation-api-service-staging'
    regions = ['eu-west-2']
  }

  regions.each {
    sh("aws ecs update-service --force-new-deployment --cluster ${cluster_name} --service ${service_name} --region ${it}")
  }
}

def publishStableTag() {
  sshagent(credentials: ['govwifi-jenkins']) {
    sh('export GIT_SSH_COMMAND="ssh -oStrictHostKeyChecking=no"')
    sh("git tag -f stable HEAD")
    sh("git push git@github.com:alphagov/govwifi-authentication-api.git --force --tags")
  }
}

def deployCancelled() {
  if(Globals.didTimeout || Globals.userInput == false) {
    return true
  }
}
