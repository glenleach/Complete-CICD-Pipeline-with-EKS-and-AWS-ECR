#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = '844181426527.dkr.ecr.eu-west-2.amazonaws.com'
        DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
        AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws_secret_access_key')
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    echo 'incrementing app version...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\${parsedVersion.majorVersion}.\\${parsedVersion.minorVersion}.\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('build app') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('build image') {
            steps {
                script {
                    echo "building the docker image..."
                    sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                    sh 'aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin ${DOCKER_REPO_SERVER}'
                    sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
                }
            }
        }
        stage('deploy') {
            environment {
                APP_NAME = 'java-maven-app'
            }
            steps {
                script {
                   echo 'deploying docker image...'
                   sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                   sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update'){
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'Github-PAT-Token', passwordVariable: 'PASS', usernameVariable: 'USER')]){
                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/glenleach/Complete-CICD-Pipeline-with-EKS-and-AWS-ECR"
                        sh 'git add .'
                        sh 'git config --global user.email "ci-bot@example.com"'
                        sh 'git config --global user.name "CI Bot"'
                        sh 'git commit -m "ci: version bump"'
                        sh 'rm -rf .git/rebase-merge'
                        sh 'git pull --rebase origin jenkins-jobs'
                        sh 'git push origin HEAD:refs/heads/jenkins-jobs'
                    }
                }
            }
        }
    }
}
