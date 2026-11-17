pipeline {
    options {
        timestamps()
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }
    agent {
        node { label 'translator && build && aws && ui' }
    }
    parameters {
        string(name: 'BUILD_VERSION', defaultValue: '', description: 'The build version to deploy (optional; skips the image build if set)')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS Region to deploy')
    }
    triggers {
        pollSCM('H/5 * * * *')
    }
    environment {
        IMAGE_NAME = "853771734544.dkr.ecr.us-east-1.amazonaws.com/translator-ui-trapi-summarizer"
        KUBERNETES_BLUE_CLUSTER_NAME = "translator-eks-ci-blue-cluster"
        DEPLOY_ENV = "ci"
        NAMESPACE = 'ui'
    }
    stages {
        stage('Build Version') {
            when { expression { return !params.BUILD_VERSION } }
            steps {
                script {
                    def generated = VersionNumber(
                        versionNumberString: 'v${BUILD_YEAR, XX}.${BUILD_MONTH, XX}${BUILD_DAY, XX}.${BUILDS_TODAY}',
                        projectStartDate:    '1970-01-01',
                        skipFailedBuilds:    true)
                    currentBuild.displayName = generated
                    env.BUILD_VERSION = generated
                    env.BUILD = 'true'
                }
            }
        }
        stage('Checkout source code') {
            steps {
                cleanWs()
                checkout scm
            }
        }
        stage('Build and push Docker image') {
            when { expression { return env.BUILD == 'true' } }
            steps {
                sh '''#!/bin/bash
                set -euo pipefail
                docker build --pull -t "$IMAGE_NAME:$BUILD_VERSION" .
                aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin 853771734544.dkr.ecr.us-east-1.amazonaws.com
                docker push "$IMAGE_NAME:$BUILD_VERSION"
                docker image rm "$IMAGE_NAME:$BUILD_VERSION" || true
                '''
            }
        }
        stage('Deploy to AWS EKS Blue') {
            agent {
                label 'translator && ci && deploy'
            }
            steps {
                configFileProvider([
                    configFile(fileId: 'values-ci.yaml', targetLocation: 'values-ncats.yaml'),
                    configFile(fileId: 'secrets.json', targetLocation: 'secrets.json'),
                    configFile(fileId: 'prepare.sh', targetLocation: 'prepare.sh')
                ]) {
                    sh '''#!/bin/bash
                    set -euo pipefail
                    aws --region "$AWS_REGION" eks update-kubeconfig --name "$KUBERNETES_BLUE_CLUSTER_NAME"
                    /bin/bash prepare.sh
                    cd translator-ops/ops/translator-ui-trapi-summarizer
                    /bin/bash deploy.sh
                    '''
                }
            }
            post {
                always {
                    echo "Clean up the workspace in deploy node!"
                    cleanWs()
                }
            }
        }
    }
}
