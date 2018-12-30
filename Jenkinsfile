pipeline {
  agent {
    label "jenkins-nodejs"
  }
  environment {
    ORG = 'dragoonsbets'
    APP_NAME = 'dragoons-ui'
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
        container('nodejs') {
          sh "npm install"
          sh "CI=true DISPLAY=:99 npm test"
          sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
          sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          dir('./charts/preview') {
            sh "make preview"
            sh "jx preview --app $APP_NAME --dir ../.."
          }
        }
      }
    }
    // stage('Code Quality - SonarQube') {
    //     environment {
    //         // This has to be the name of the scanner configured in Global Settings Jenkins
    //         scannerHome = tool 'SQScanner32'
    //     }
    //     steps {
    //       container('nodejs') {
    //         withSonarQubeEnv('SonarQube 7.4 Com - Dragoons') {
    //             sh "${scannerHome}/bin/sonar-scanner" +
    //             " -Dsonar.projectVersion=$BRANCH_NAME-build-$BUILD_NUMBER"
    //         }
    //         timeout(time: 10, unit: 'MINUTES') {
    //             // Will halt the pipeline until SonarQube notifies Jenkins whether quality gate is passed through webhook setup earlier
    //             waitForQualityGate abortPipeline: true
    //         }
    //       }
    //     }
    // }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        container('nodejs') {

          // ensure we're not on a detached head
          sh "git checkout master"
          sh "git config --global credential.helper store"
          sh "jx step git credentials"

          // Start Sentry Release
          sh "npm install @sentry/cli"
          // sh "export SENTRY_AUTH_TOKEN="
          // sh "export SENTRY_ORG=Dragoons"
          // sh "export SENTRY_URL=https://sentry.drogon.dragoons.io"
          // sh "export SENTRY_LOG_LEVEL=debug"
          // sh "export REL_VERSION=\$(./node_modules/@sentry/cli/sentry-cli releases propose-version)"

          // Create a release
          // sh "./node_modules/@sentry/cli/sentry-cli releases -o dragoons new \$(./node_modules/@sentry/cli/sentry-cli releases propose-version) --project dragoons-ui  --log-level debug"
          sh "./node_modules/@sentry/cli/sentry-cli releases new \$(./node_modules/@sentry/cli/sentry-cli releases propose-version) --project dragoons-ui"

          // Associate commits with the release
          sh "./node_modules/@sentry/cli/sentry-cli releases set-commits --auto \$(./node_modules/@sentry/cli/sentry-cli releases propose-version)"

          // now that we are not in a detached head we can retrieve the version in later steps
          sh "echo \$(jx-release-version) > VERSION"
          sh "jx step tag --version \$(cat VERSION)"
          sh "npm install"
          sh "CI=true DISPLAY=:99 npm test"
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
        container('nodejs') {
          dir('./charts/dragoons-ui') {
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
  post {
        always {
          cleanWs()
        }
  }
}
