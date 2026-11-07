pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  triggers { githubPush() }

  environment {
    IMAGE     = 'mals77703/kanban-dashboard'
    LIVE_NAME = 'kanban-app'
    CAND_NAME = 'kanban-candidate'
    LIVE_PORT = '80'
    CAND_PORT = '8081'
    RUN_OPTS  = '--memory=256m --cpus=0.5 --restart unless-stopped'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_SHA = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
          env.TAG     = "${env.BUILD_NUMBER}-${env.GIT_SHA}"
        }
        echo "Building ${env.IMAGE}:${env.TAG}"
      }
    }

    stage('Docker Build') {
      steps { sh 'docker build -t $IMAGE:$TAG .' }
    }

    stage('Registry Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                          usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
            docker logout
          '''
        }
      }
    }

    stage('Deploy Candidate') {
      steps {
        sh '''
          docker pull $IMAGE:$TAG
          docker rm -f $CAND_NAME || true
          docker run -d --name $CAND_NAME $RUN_OPTS -p $CAND_PORT:8080 $IMAGE:$TAG
        '''
      }
    }

    stage('Health Check') {
      steps { script { waitHealthy(env.CAND_NAME, env.CAND_PORT) } }
    }

    stage('Promote to Live') {
      steps {
        sh '''
          docker inspect -f '{{.Config.Image}}' $LIVE_NAME > .prev_image 2>/dev/null || : > .prev_image
          docker rm -f $LIVE_NAME || true
          docker run -d --name $LIVE_NAME $RUN_OPTS -p $LIVE_PORT:8080 $IMAGE:$TAG
          docker rm -f $CAND_NAME || true
        '''
        script { waitHealthy(env.LIVE_NAME, env.LIVE_PORT) }
      }
    }
  }

  post {
    success { echo "SUCCESS: ${env.IMAGE}:${env.TAG} is live" }
    failure {
      echo "FAILED: attempting rollback"
      sh '''
        docker logs --tail 50 $CAND_NAME || true
        docker rm -f $CAND_NAME || true
        if [ -s .prev_image ] && ! curl -fs http://localhost:$LIVE_PORT/ > /dev/null; then
          PREV=$(cat .prev_image)
          docker rm -f $LIVE_NAME || true
          docker run -d --name $LIVE_NAME $RUN_OPTS -p $LIVE_PORT:8080 $PREV
          echo "ROLLED BACK to $PREV"
        fi
      '''
    }
    always { sh 'docker image prune -f || true' }
  }
}

// Passes only if Docker HEALTHCHECK = healthy AND the endpoint returns HTML
def waitHealthy(String name, String port) {
  sh """
    for i in \$(seq 1 20); do
      STATUS=\$(docker inspect -f '{{.State.Health.Status}}' ${name})
      echo "health=\$STATUS (attempt \$i)"
      [ "\$STATUS" = "healthy" ] && break
      sleep 3
    done
    [ "\$STATUS" = "healthy" ]
    curl -fsS http://localhost:${port}/ | grep -qi '<html'
    docker ps --filter name=${name}
  """
}
