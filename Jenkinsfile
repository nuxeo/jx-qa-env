/*
 * (C) Copyright 2020 Nuxeo (http://nuxeo.com/) and others.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 * Contributors:
 *     Nelson Silva <nsilva@nuxeo.com>
 */
 properties([
  [$class: 'GithubProjectProperty', projectUrlStr: 'https://github.com/nuxeo/jx-qa-env/'],
  [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', daysToKeepStr: '60', numToKeepStr: '60', artifactNumToKeepStr: '5']],
  disableConcurrentBuilds(),
])

void setGitHubBuildStatus(String context, String message, String state) {
  step([
    $class: 'GitHubCommitStatusSetter',
    reposSource: [$class: 'ManuallyEnteredRepositorySource', url: 'https://github.com/nuxeo/jx-qa-env'],
    contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: context],
    statusResultSource: [$class: 'ConditionalStatusResultSource', results: [[$class: 'AnyBuildResult', message: message, state: state]]],
  ])
}

pipeline {
  agent {
    label 'jenkins-jx-base'
  }
  environment {
    JX_VERSION = '2.0.1849'
    NAMESPACE = 'qa'
    SERVICE_ACCOUNT = 'jenkins'
  }
  stages {
    stage('Upgrade Jenkins X platform') {
      steps {
        setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X platform', 'PENDING')
        container('jx-base') {
          echo "Upgrade Jenkins X ${NAMESPACE} platform"
          script {
            // get the existing docker config
            def dockerConfig = sh(
              script: "jx step credential -s jenkins-docker-cfg -k config.json | tr -d '\\n'",
              returnStdout: true
            ).trim();

            withEnv(["DOCKER_REGISTRY_CONFIG=${dockerConfig}"]) {
              sh """
              # initialize Helm without installing Tiller
              helm init --client-only --service-account ${SERVICE_ACCOUNT}

              # add local chart repository
              helm repo add jenkins-x http://chartmuseum.jenkins-x.io

              # replace env vars in values.yaml
              envsubst < values.yaml > myvalues.yaml

              # upgrade Jenkins X platform
              jx upgrade platform --namespace=${NAMESPACE} \
                --version ${JX_VERSION} \
                --local-cloud-environment \
                --always-upgrade \
                --cleanup-temp-files=true \
                --batch-mode

              # log jenkins deployment image
              kubectl get deployments.apps jenkins -n ${NAMESPACE} -oyaml -o'jsonpath={ .spec.template.spec.containers[0].image }'

              # restart Jenkins pod
              kubectl scale deployment jenkins -n ${NAMESPACE} --replicas 0
              kubectl scale deployment jenkins -n ${NAMESPACE} --replicas 1
            """
            }
          }
        }
      }
      post {
        success {
          setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X platform', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('upgrade', 'Upgrade Jenkins X platform', 'FAILURE')
        }
      }
    }
    stage('Perform Git release') {
      when {
        branch 'master'
      }
      steps {
        setGitHubBuildStatus('release', 'Release', 'PENDING')
        container('jx-base') {
          script {
            VERSION = sh(returnStdout: true, script: 'jx-release-version')
          }
          withEnv(["VERSION=${VERSION}"]) {
            sh """
              # ensure we're not on a detached head
              git checkout master

              # create the Git credentials
              jx step git credentials
              git config credential.helper store

              # Git tag
              jx step tag -v ${VERSION}

              # Git release
              jx step changelog -v v${VERSION}
            """
          }
        }
      }
      post {
        always {
          step([$class: 'JiraIssueUpdater', issueSelector: [$class: 'DefaultIssueSelector'], scm: scm])
        }
        success {
          setGitHubBuildStatus('release', 'Release', 'SUCCESS')
        }
        failure {
          setGitHubBuildStatus('release', 'Release', 'FAILURE')
        }
      }
    }
  }
}
