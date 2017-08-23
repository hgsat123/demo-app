#!groovy

String GIT_VERSION

node {

  def buildEnv
  def devAddress

  stage ('Checkout') {
    deleteDir()
    checkout scm
    GIT_VERSION = sh (
      script: 'git describe --tags',
      returnStdout: true
    ).trim()
  }

  stage ('Build Custom Environment') {
    buildEnv = docker.build("build_env:${GIT_VERSION}", 'custom-build-env')
  }

  buildEnv.inside {

    stage ('Build') {
      sh 'sbt compile'
      sh 'sbt sampleClient/universal:stage'
    }

    stage ('Test') {
      parallel (
        'Test Server' : {
          sh 'sbt server/test'
        },
        'Test Sample Client' : {
          sh 'sbt sampleClient/test'
        }
      )
    }

    stage ('Prepare Docker Image') {
      sh 'sbt server/docker:stage'
    }
  }

  stage ('Build and Push Docker Image') {

  withCredentials([[$class: "UsernamePasswordMultiBinding", usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS', credentialsId: 'docker-hub-credentials']]) {
      sh 'docker login --username $DOCKERHUB_USER --password $DOCKERHUB_PASS'
    }

     def serverImage = docker.build("hgsat123/myapp:${GIT_VERSION}", 'server/target/docker/stage')

   serverImage.push()
   sh 'docker logout'
  }

  stage ('Deploy to DEV') {
    devAddress = deployContainer("hgsat123/myapp:${GIT_VERSION}", 'dev')
  }

  stage ('Verify DEV') {
      sh "kubectl get po -o wide --namespace=dev | grep Running"
  }
}

stage 'Deploy to LIVE'
  timeout(time:2, unit:'DAYS') {
    input message:'Approve deployment to LIVE?'
  }
  node {
    deployContainer("hgsat123/myapp:${GIT_VERSION}", 'live')
    stage ('Verify LIVE') {
      sh "kubectl get po -o wide --namespace=live | grep Running"
   }
  }

def deployContainer(image, env) {
    withCredentials([[$class: "FileBinding", credentialsId: 'kubeconfig', variable: 'KUBE_CONFIG']]) {
      def kubectl = "kubectl  --kubeconfig=\$KUBE_CONFIG --context=${env}"
      sh "${kubectl} set image deployment/my-demo my-demo=${image}"
      sh "${kubectl} rollout status deployment/my-demo"
      return sh (
        script: "kubectl get service/my-demo --namespace=${env} -o jsonpath='{.spec.clusterIP}'",
        returnStdout: true
      ).trim()
    }
}


// vim: set syntax=groovy :
