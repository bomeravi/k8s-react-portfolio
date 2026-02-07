pipeline {
  agent any

  options {
    timestamps()
  }

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Docker image tag to set in Helm values (leave empty to use v${BUILD_NUMBER})')
  }

  environment {
    K8S_VALUES_FILE = 'k8s/helm/react-portfolio/values.yaml'
    K8S_CHART_FILE = 'k8s/helm/react-portfolio/Chart.yaml'
    GIT_CREDENTIALS_ID = 'k8s-git-ssh'
    GIT_USER_NAME = 'jenkins'
    GIT_USER_EMAIL = 'jenkins@local'
  }

  stages {
    stage('Resolve Tag') {
      steps {
        script {
          def tag = params.IMAGE_TAG?.trim()
          env.IMAGE_TAG = tag ? tag : "v${env.BUILD_NUMBER}"
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Update Image Tag') {
      steps {
        sh '''
          set -euo pipefail
          git config user.name "${GIT_USER_NAME}"
          git config user.email "${GIT_USER_EMAIL}"

          sed -i "s/^  tag:.*/  tag: ${IMAGE_TAG}/" "${K8S_VALUES_FILE}"
          sed -i "s/^appVersion:.*/appVersion: \\"${IMAGE_TAG}\\"/" "${K8S_CHART_FILE}"

          if [ -z "$(git status --porcelain)" ]; then
            echo "No changes to commit."
            exit 0
          fi

          git add "${K8S_VALUES_FILE}" "${K8S_CHART_FILE}"
          git commit -m "chore: bump image tag to ${IMAGE_TAG}"
        '''
      }
    }

    stage('Push') {
      steps {
        sshagent([env.GIT_CREDENTIALS_ID]) {
          sh 'git push origin HEAD'
        }
      }
    }
  }
}
