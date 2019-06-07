#!groovy
import bcgov.GitHubHelper

// --------------------
// Declarative Pipeline
// --------------------
pipeline {
  agent any

  environment {
    // Enable pipeline verbose debug output if greater than 0
    DEBUG_OUTPUT = 'false'

    // Get projects/namespaces from config maps
    DEV_PROJECT = new File('/var/run/configs/ns/project.dev').getText('UTF-8').trim()
    TEST_PROJECT = new File('/var/run/configs/ns/project.test').getText('UTF-8').trim()
    PROD_PROJECT = new File('/var/run/configs/ns/project.prod').getText('UTF-8').trim()
    TOOLS_PROJECT = new File('/var/run/configs/ns/project.tools').getText('UTF-8').trim()

    // Get application config from config maps
    REPO_OWNER = new File('/var/run/configs/jobs/repo.owner').getText('UTF-8').trim()
    REPO_NAME = new File('/var/run/configs/jobs/repo.name').getText('UTF-8').trim()
    APP_NAME = new File('/var/run/configs/jobs/app.name').getText('UTF-8').trim()
    APP_DOMAIN = new File('/var/run/configs/jobs/app.domain').getText('UTF-8').trim()

    // JOB_NAME should be the pull request/branch identifier (i.e. 'pr-5')
    JOB_NAME = JOB_BASE_NAME.toLowerCase()


    // SOURCE_REPO_* references git repository resources
    SOURCE_REPO_RAW = "https://raw.githubusercontent.com/${REPO_OWNER}/${REPO_NAME}/master"
    SOURCE_REPO_REF = 'master'
    SOURCE_REPO_URL = "https://github.com/${REPO_OWNER}/${REPO_NAME}.git"

    // HOST_ROUTE is the full domain route, without a path (ie. 'appname-dev.pathfinder.gov.bc.ca')
    DEV_HOST = "${APP_NAME}-dev.${APP_DOMAIN}"
    TEST_HOST = "${APP_NAME}-test.${APP_DOMAIN}"
    PROD_HOST = "${APP_NAME}.${APP_DOMAIN}"
    // will be added to the HOST_ROUTE
    PATH_ROOT = "/${APP_NAME}"
  }

 stages {
    stage('Init') {
      steps {
        notifyStageStatus('Init', 'PENDING')

        // Cancel any running builds in progress
        timeout(5) {
          echo "Cancelling previous ${APP_NAME}-${JOB_NAME} builds in progress..."
          abortAllPreviousBuildInProgress(currentBuild)
        }

        script {
          if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
            // Force OpenShift Plugin directives to be verbose
            openshift.logLevel(1)

            // Print all environment variables
            echo 'DEBUG - All pipeline environment variables:'
            echo sh(returnStdout: true, script: 'env')
          }
        }
      }
      post {
        success {
          echo 'Init successful'
          notifyStageStatus('Init', 'SUCCESS')
        }
        unsuccessful {
          echo 'Init failed'
          notifyStageStatus('Init', 'FAILURE')
        }
      }
    }

    stage('Build Prep.') {
      steps {
          notifyStageStatus('Build Prep.', 'PENDING')
          script {
            openshift.withCluster() {
              openshift.withProject(TOOLS_PROJECT) {
                if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
                  echo "DEBUG - Using project: ${openshift.project()}"
                }

                echo "Processing BuildConfig ${APP_NAME}-${JOB_NAME}-frontend-builder..."
                def bcFrontendBuilderTemplate = openshift.process('-f',
                  'openshift/frontend-builder.bc.yaml',
                  "REPO_NAME=${REPO_NAME}",
                  "JOB_NAME=${JOB_NAME}",
                  "SOURCE_REPO_URL=${SOURCE_REPO_URL}",
                  "SOURCE_REPO_REF=${SOURCE_REPO_REF}",
                  "APP_NAME=${APP_NAME}",
                  "PATH_ROOT=${PATH_ROOT}"
                )

                echo "Building ImageStream ${APP_NAME}-${JOB_NAME}-frontend-builder..."
                openshift.apply(bcFrontendBuilderTemplate).narrow('bc').startBuild('-w').logs('-f')
              }
            }
          }
      }
      post {
        success {
          echo 'Build Prep. successful'
          notifyStageStatus('Build Prep.', 'SUCCESS')
          script {
            openshift.withCluster() {
              openshift.withProject(TOOLS_PROJECT) {
                if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
                  echo "DEBUG - Using project: ${openshift.project()}"
                } else {
                  def bcFrontendBuilderConfig = openshift.selector('bc', "${APP_NAME}-${JOB_NAME}-frontend-builder")

                  if(bcFrontendBuilderConfig.exists()) {
                    echo "Removing BuildConfig ${APP_NAME}-${JOB_NAME}-frontend-builder..."
                    bcFrontendBuilderConfig.delete()
                  }
                }
              }
            }
          }
        }
        unsuccessful {
          echo 'Build Prep. failed'
          notifyStageStatus('Build Prep.', 'FAILURE')
        }
      }
    }

    stage('Build') {
      steps {
        notifyStageStatus('Build', 'PENDING')
        script {
          openshift.withCluster() {
            openshift.withProject(TOOLS_PROJECT) {
              if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
                echo "DEBUG - Using project: ${openshift.project()}"
              }

              echo "Processing BuildConfig ${APP_NAME}-${JOB_NAME}-backend..."
              def bcBackendTemplate = openshift.process('-f',
                'openshift/backend.bc.yaml',
                "REPO_NAME=${REPO_NAME}",
                "JOB_NAME=${JOB_NAME}",
                "SOURCE_REPO_URL=${SOURCE_REPO_URL}",
                "SOURCE_REPO_REF=${SOURCE_REPO_REF}",
                "APP_NAME=${APP_NAME}",
              )

              echo "Processing BuildConfig ${APP_NAME}-${JOB_NAME}-frontend..."
              def bcFrontendTemplate = openshift.process('-f',
                'openshift/frontend.bc.yaml',
                "REPO_NAME=${REPO_NAME}",
                "JOB_NAME=${JOB_NAME}",
                "SOURCE_REPO_URL=${SOURCE_REPO_URL}",
                "SOURCE_REPO_REF=${SOURCE_REPO_REF}",
                "APP_NAME=${APP_NAME}"
              )

              echo "Processing BuildConfig ${APP_NAME}-${JOB_NAME}-reverse-proxy..."
              def bcReverseProxyTemplate = openshift.process('-f',
                'openshift/reverse-proxy.bc.yaml',
                "REPO_NAME=${REPO_NAME}",
                "JOB_NAME=${JOB_NAME}",
                "SOURCE_REPO_URL=${SOURCE_REPO_URL}",
                "SOURCE_REPO_REF=${SOURCE_REPO_REF}",
                "APP_NAME=${APP_NAME}"
              )

              echo "Building ImageStream ${APP_NAME}-${JOB_NAME}-backend..."
              openshift.apply(bcBackendTemplate).narrow('bc').startBuild()

              echo "Building ImageStream ${APP_NAME}-${JOB_NAME}-frontend..."
              openshift.apply(bcFrontendTemplate).narrow('bc').startBuild()

              echo "Building ImageStream ${APP_NAME}-${JOB_NAME}-reverse-proxy..."
              openshift.apply(bcReverseProxyTemplate).narrow('bc').startBuild()

              timeout(10) {
                openshift.selector("bc", "${APP_NAME}-${JOB_NAME}-backend").related('builds').untilEach(1) {
                  return (it.object().status.phase == "Complete")
                }
                openshift.selector("bc", "${APP_NAME}-${JOB_NAME}-frontend").related('builds').untilEach(1) {
                  return (it.object().status.phase == "Complete")
                }
                openshift.selector("bc", "${APP_NAME}-${JOB_NAME}-reverse-proxy").related('builds').untilEach(1) {
                  return (it.object().status.phase == "Complete")
                }
              }
            }
          }
        }
      }
      post {
        success {
          echo 'Build Success'
          notifyStageStatus('Build', 'SUCCESS')
          echo 'Cleanup BuildConfigs...'
          script {
            openshift.withCluster() {
              openshift.withProject(TOOLS_PROJECT) {
                if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
                  echo "DEBUG - Using project: ${openshift.project()}"
                } else {
                  def bcBackendConfig = openshift.selector('bc', "${APP_NAME}-${JOB_NAME}-backend")
                  def bcFrontendConfig = openshift.selector('bc', "${APP_NAME}-${JOB_NAME}-frontend")
                  def bcReverseProxyConfig = openshift.selector('bc', "${APP_NAME}-${JOB_NAME}-reverse-proxy")

                  if(bcBackendConfig.exists()) {
                    echo "Removing BuildConfig ${APP_NAME}-${JOB_NAME}-backend..."
                    bcBackendConfig.delete()
                  }
                  if(bcFrontendConfig.exists()) {
                    echo "Removing BuildConfig ${APP_NAME}-${JOB_NAME}-frontend..."
                    bcFrontendConfig.delete()
                  }
                  if(bcReverseProxyConfig.exists()) {
                    echo "Removing BuildConfig ${APP_NAME}-${JOB_NAME}-reverse-proxy..."
                    bcReverseProxyConfig.delete()
                  }
                }
              }
            }
          }
        }
        unsuccessful {
          echo 'Build failed'
          notifyStageStatus('Build', 'FAILURE')
        }
      }
    }

    stage('Deploy - Dev') {
      steps {
        script {
          deployStage('Dev', DEV_PROJECT, DEV_HOST, PATH_ROOT)
        }
      }
      post {
        success {
          createDeploymentStatus(DEV_PROJECT, 'SUCCESS', DEV_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Dev', 'SUCCESS')
        }
        unsuccessful {
          createDeploymentStatus(DEV_PROJECT, 'FAILURE', DEV_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Dev', 'FAILURE')
        }
      }
    }

    stage('Deploy - Test') {
      steps {
        script {
          deployStage('Test', TEST_PROJECT, TEST_HOST, PATH_ROOT)
        }
      }
      post {
        success {
          createDeploymentStatus(TEST_PROJECT, 'SUCCESS', TEST_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Test', 'SUCCESS')
        }
        unsuccessful {
          createDeploymentStatus(TEST_PROJECT, 'FAILURE', TEST_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Test', 'FAILURE')
        }
      }
    }

    stage('Deploy - Prod') {
      steps {
        script {
          deployStage('Prod', PROD_PROJECT, PROD_HOST, PATH_ROOT)
        }
      }
      post {
        success {
          createDeploymentStatus(PROD_PROJECT, 'SUCCESS', PROD_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Prod', 'SUCCESS')
        }
        unsuccessful {
          createDeploymentStatus(PROD_PROJECT, 'FAILURE', PROD_HOST, PATH_ROOT)
          notifyStageStatus('Deploy - Prod', 'FAILURE')
        }
      }
    }
  }
}


// ------------------
// Pipeline Functions
// ------------------

// Parameterized deploy stage
def deployStage(String stageEnv, String projectEnv, String hostEnv, String pathEnv) {
  if (!stageEnv.equalsIgnoreCase('Dev')) {
    input("Deploy to ${projectEnv}?")
  }

  notifyStageStatus("Deploy - ${stageEnv}", 'PENDING')

  openshift.withCluster() {
    openshift.withProject(projectEnv) {
      if(DEBUG_OUTPUT.equalsIgnoreCase('true')) {
        echo "DEBUG - Using project: ${openshift.project()}"
      }

      echo "Checking for ConfigMaps and Secrets in project ${openshift.project()}..."
      if(!(openshift.selector('cm', "cmsg-config").exists() &&
      openshift.selector('secret', "cmsg-client").exists())) {
        echo 'Some ConfigMaps and/or Secrets are missing. Please consult the openshift readme for details.'
        throw e
      }

      // "move" images from tools project to our target environment and deploy from there...
      echo "Tagging Image ${APP_NAME}-${JOB_NAME}-backend:latest..."
      openshift.tag("${TOOLS_PROJECT}/${APP_NAME}-${JOB_NAME}-backend:latest", "${APP_NAME}-${JOB_NAME}-backend:latest")

      echo "Tagging Image ${APP_NAME}-${JOB_NAME}-frontend:latest..."
      openshift.tag("${TOOLS_PROJECT}/${APP_NAME}-${JOB_NAME}-frontend:latest", "${APP_NAME}-${JOB_NAME}-frontend:latest")

      echo "Tagging Image ${APP_NAME}-${JOB_NAME}-reverse-proxy:latest..."
      openshift.tag("${TOOLS_PROJECT}/${APP_NAME}-${JOB_NAME}-reverse-proxy:latest", "${APP_NAME}-${JOB_NAME}-reverse-proxy:latest")

      echo "Processing DeploymentConfig ${APP_NAME}-${JOB_NAME}-backend..."
      def dcBackendTemplate = openshift.process('-f',
        'openshift/backend.dc.yaml',
        "REPO_NAME=${REPO_NAME}",
        "JOB_NAME=${JOB_NAME}",
        "NAMESPACE=${projectEnv}",
        "APP_NAME=${APP_NAME}"
      )

      echo "Processing DeploymentConfig ${APP_NAME}-${JOB_NAME}-frontend..."
      def dcFrontendTemplate = openshift.process('-f',
        'openshift/frontend.dc.yaml',
        "REPO_NAME=${REPO_NAME}",
        "JOB_NAME=${JOB_NAME}",
        "NAMESPACE=${projectEnv}",
        "APP_NAME=${APP_NAME}",
        "PATH_ROOT=${pathEnv}"
      )

      echo "Applying Deployment ${APP_NAME}-${JOB_NAME}-backend..."
      createDeploymentStatus(projectEnv, 'PENDING', hostEnv, pathEnv)
      def dcBackendConfig = openshift.apply(dcBackendTemplate).narrow('dc')

      echo "Applying Deployment ${APP_NAME}-${JOB_NAME}-frontend..."
      createDeploymentStatus(projectEnv, 'PENDING', hostEnv, pathEnv)
      def dcFrontendConfig = openshift.apply(dcFrontendTemplate).narrow('dc')

      // Wait for new deployment to roll out
      timeout(10) {
        openshift.selector("dc", "${APP_NAME}-${JOB_NAME}-backend").related('pods').untilEach(1) {
          return (it.object().status.phase == "Running")
        }
        openshift.selector("dc", "${APP_NAME}-${JOB_NAME}-frontend").related('pods').untilEach(1) {
          return (it.object().status.phase == "Running")
        }
      }

      // Do reverse proxy after back and front are running...

      echo "Processing DeploymentConfig ${APP_NAME}-${JOB_NAME}-reverse-proxy..."
      def dcProxyTemplate = openshift.process('-f',
        'openshift/reverse-proxy.dc.yaml',
        "REPO_NAME=${REPO_NAME}",
        "JOB_NAME=${JOB_NAME}",
        "NAMESPACE=${projectEnv}",
        "APP_NAME=${APP_NAME}",
        "HOST_ROUTE=${hostEnv}",
        "PATH_ROOT=${pathEnv}"
      )

      echo "Applying Deployment ${APP_NAME}-${JOB_NAME}-frontend..."
      createDeploymentStatus(projectEnv, 'PENDING', hostEnv, pathEnv)
      def dcProxyConfig = openshift.apply(dcProxyTemplate).narrow('dc')

      timeout(5){
        openshift.selector("dc", "${APP_NAME}-${JOB_NAME}-reverse-proxy").related('pods').untilEach(1) {
          return (it.object().status.phase == "Running")
        }
      }

    }
  }
}

// --------------------
// Supporting Functions
// --------------------

// Notify stage status and pass to Jenkins-GitHub library
def notifyStageStatus(String name, String status) {
  def sha1 = GIT_COMMIT
  if(JOB_BASE_NAME.startsWith('PR-')) {
    sha1 = GitHubHelper.getPullRequestLastCommitId(this)
  }

  GitHubHelper.createCommitStatus(
    this, sha1, status, BUILD_URL, '', "Stage: ${name}"
  )
}

// Create deployment status and pass to Jenkins-GitHub library
def createDeploymentStatus (String environment, String status, String hostEnv, String pathEnv) {
  def ghDeploymentId = new GitHubHelper().createDeployment(
    this,
    SOURCE_REPO_REF,
    [
      'environment': environment,
      'task': "deploy:pull:${CHANGE_ID}"
    ]
  )

  new GitHubHelper().createDeploymentStatus(
    this,
    ghDeploymentId,
    status,
    ['targetUrl': "https://${hostEnv}${pathEnv}"]
  )

  if (status.equalsIgnoreCase('SUCCESS')) {
    echo "${environment} deployment successful at https://${hostEnv}${pathEnv}"
  } else if (status.equalsIgnoreCase('PENDING')) {
    echo "${environment} deployment pending..."
  } else if (status.equalsIgnoreCase('FAILURE')) {
    echo "${environment} deployment failed"
  }
}

// Creates a comment and pass to Jenkins-GitHub library
def commentOnPR(String comment) {
  if(JOB_BASE_NAME.startsWith('PR-')) {
    GitHubHelper.commentOnPullRequest(this, comment)
  }
}

