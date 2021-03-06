pipeline {
    agent {
        kubernetes {
              // Change the name of jenkins-maven label to be able to use yaml configuration snippet
              label "maven-dind"
              // Inherit from Jx Maven pod template
              inheritFrom "maven"
              // Add pod configuration to Jenkins builder pod template
              yamlFile "maven-dind.yaml"
        }
    }
    environment {
      ORG                  = "activiti"
      APP_NAME             = "activiti-cloud-notifications-graphql"
      VERSION              = jx_release_version()
      CHARTMUSEUM_CREDS    = credentials('jenkins-x-chartmuseum')
      GITHUB_CHARTS_REPO   = "https://github.com/${ORG}/activiti-cloud-helm-charts.git"
      GITHUB_HELM_REPO_URL = "https://${ORG}.github.io/activiti-cloud-helm-charts/"
      RELEASE_BRANCH       = "master"
    }
    stages {
      stage("CI Build and push snapshot") {
        when {
          branch "PR-*"
        }
        environment {
          PROJECT_VERSION     = maven_project_version()      
          VERSION = "$PROJECT_VERSION".replaceAll("SNAPSHOT","$BRANCH_NAME-$BUILD_NUMBER-SNAPSHOT")
          PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
          HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
        }
        steps {
          container("maven") {
            sh "mvn versions:set -DnewVersion=$VERSION"
            sh "mvn install -DskipITs=false"
            sh "export VERSION=$VERSION && skaffold build -f skaffold.yaml"

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$VERSION"

            // Let's build chart to check for any errors
            dir("./charts/$APP_NAME") {
              sh "make preview"
            }

            sh "mvn deploy -DskipTests"
          }
        }
      }
      stage("Build Release") {
        when {
          branch "${RELEASE_BRANCH}"
        }
        steps {
          container("maven") {
            // ensure we're not on a detached head
            sh "git checkout ${RELEASE_BRANCH}"
            sh "git config --global credential.helper store"

            sh "jx step git credentials"
            // so we can retrieve the version in later steps
            sh "echo $VERSION > VERSION"
            sh "mvn versions:set -DnewVersion=$VERSION"

            sh "mvn clean install -DskipIT=false"

            dir ("./charts/$APP_NAME") {
              sh "make build tag"
            }
            sh "export VERSION=$VERSION && skaffold build -f skaffold.yaml"

            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$VERSION"

            sh "mvn deploy -DskipTests"
          }
        }
      }
      stage("Promote to Environments") {
        when {
          branch "${RELEASE_BRANCH}"
        }
        steps {
          container("maven") {
            dir ("./charts/$APP_NAME") {
              sh "jx step changelog --generate-yaml=false --version v$VERSION"

              // publish to github
              retry(5) { 
                sh "make github"
              }
              sh 'sleep 8'

              // Update versions
              retry(2) { 
                sh "make updatebot/push-version"
              }

            }
          }
        }
      }
    }  
    
    post {
        failure {
           slackSend(
             channel: "#activiti-community-builds",
             color: "danger",
             message: "$BUILD_URL"
           )
        } 
        always {
            cleanWs()
        }
    }
}

def jx_release_version() {
    container('maven') {
        return sh( script: "echo \$(jx-release-version)", returnStdout: true).trim()
    }
}

def maven_project_version() {
    container('maven') {
        return sh( script: "echo \$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout -f pom.xml)", returnStdout: true).trim()
    }
}
