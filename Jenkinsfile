
pipeline {
    agent any
    tools {
        maven 'Maven'
    }
    environment {
        DOCKER_REPO_SERVER = '922854651834.dkr.ecr.us-east-1.amazonaws.com'
        DOCKER_REPO = "${DOCKER_REPO_SERVER}/java-maven-app"
    }
    stages {
          stage('increment version') {
            steps {
                script {
                    echo "Incrementing app version"
                    sh 'mvn build-helper:parse-version versions:set \
                       -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                       versions:commit'
                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
          }

        stage('build jar') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'

                }
            }
        }

        stage('build image') {
            when {
                expression {
                    BRANCH_NAME == "master"
                }
            }
            steps {
                script {
                    echo "building the docker image..."
                    withCredentials([usernamePassword(credentialsId: 'ecr-credentials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t ${DOCKER_REPO}:${IMAGE_NAME} ."
                        sh 'echo $PASS | docker login -u $USER --password-stdin ${DOCKER_REPO_SERVER}'
                        sh "docker push ${DOCKER_REPO}:${IMAGE_NAME}"
                    }
                }
            }
        }

        stage('deploy') {
            when {
                expression {
                    BRANCH_NAME == "master"
                }
            }
            environment {
                    AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
                    AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
                    APP_NAME = 'java-maven-app'
            }

            steps {
                script {
                    echo 'deploying the application...'
                    sh 'envsubst < kubernetes/deployment.yaml | kubectl apply -f -'
                    sh 'envsubst < kubernetes/service.yaml | kubectl apply -f -'
                }
            }
        }
        stage('commit version update') {
              when {
                  expression {
                      BRANCH_NAME == "master"
                  }
              }
              steps {
                  script {
                      withCredentials([usernamePassword(credentialsId: 'jenkinspush', passwordVariable: 'PAT' , usernameVariable: 'USER')]) {
                          sh 'git remote set-head origin master'
                          sh 'git config --global user.email "jenkins@example.com"'
                          sh 'git config --global user.name "jenkins"'
                          sh 'git status'
                          sh 'git add .'
                          sh 'git branch'
                          sh 'git config --list'
                          sh "git remote set-url origin https://${PAT}@github.com/irschad/ci-cd-eks-ecr.git"
                          sh "git commit -m 'ci: version bump'"
                          sh 'git push origin HEAD:master'
                    }
                  }
              }
          }

    }
}
