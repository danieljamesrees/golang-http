pipeline {
  agent any
  environment {
    ORG = 'danieljamesrees'
    APP_NAME = 'golang-http'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        dir('/home/jenkins/go/src/github.com/danieljamesrees/golang-http') {
          checkout scm
          sh "make linux"
          sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
        }
        dir('/home/jenkins/go/src/github.com/danieljamesrees/golang-http/charts/preview') {
          sh "make preview"
          sh "jx preview --app $APP_NAME --dir ../.."
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        dir('/home/jenkins/go/src/github.com/danieljamesrees/golang-http') {
          git 'https://github.com/danieljamesrees/golang-http.git'

          // so we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"
          sh "jx step tag --version \$(cat VERSION)"
          sh "make build"
          sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        dir('/home/jenkins/go/src/github.com/danieljamesrees/golang-http/charts/golang-http') {
          sh "jx step changelog --version v\$(cat ../../VERSION)"

          // release the helm chart
          sh "jx step helm release"

          // promote through all 'Auto' promotion Environments
          sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
        }
      }
    }
  }
}
