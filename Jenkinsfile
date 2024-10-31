def cancelPreviousBuilds() {
    def jobName = env.JOB_NAME
    def buildNumber = env.BUILD_NUMBER.toInteger()

    /* Get job name */
    def currentJob = Jenkins.instance.getItemByFullName(jobName)

    /* Iterating over the builds for specific job */
    for (def build : currentJob.builds) {
        def exec = build.getExecutor()
        /* If there is a build that is currently running and it's not current build */
        if (build.isBuilding() && build.number.toInteger() < buildNumber && exec != null) {
            /* Then stop it */
            exec.interrupt(
                    Result.ABORTED,
                    new CauseOfInterruption.UserInterruption("Aborted by #${currentBuild.number}")
                )
            println("Aborted previously running build #${build.number}")
        }
    }
}

pipeline {
    agent { label "agent-${env.BRANCH_NAME}" }

    tools {
        go 'go20'
    }

    environment {
        DOCKER_USERNAME = 'your-docker-username'
        APP_NAME = 'your-apps-name'
        APP_DOMAIN = credentials("APP_DOMAIN-${env.BRANCH_NAME}")
        APP_ENV = credentials("APP_ENV-${env.BRANCH_NAME}")
        PORT = credentials("PORT-${env.BRANCH_NAME}")
        // etc
    }

    stages {
        stage('SCM') {
            steps {
                checkout scm
                script {
                    cancelPreviousBuilds()
                    sh 'docker system prune -a -f && echo 1 >  /proc/sys/vm/drop_caches'
                }
            }
        }

        stage('GitLeaks Scan') {
                agent {
                    docker {
                        image 'zricethezav/gitleaks:latest'
                        args '--entrypoint='
                    }
                }

                steps {
                    script {
                        try {
                        sh "gitleaks detect --source . --report-path analytics-${APP_NAME}-repo.json -v"
                        } catch (e) {
                        currentBuild.result = 'FAILURE'
                        }
                    }
                }
        }

        stage('Unit Testing') {
            steps {
                script {
                    sh 'go test ./... -short -coverprofile=coverage.out'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps{
                script {
                    def scannerHome = tool 'SonarQube'
                    withSonarQubeEnv() {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Build Image Docker') {
            when {
                expression {
                    env.BRANCH_NAME == 'staging' || env.BRANCH_NAME == 'master'
                }
            }
            steps {
                script {
                    def buildArgs = """\
                    --build-arg APP_NAME=${env.APP_NAME} \
                    --build-arg APP_DOMAIN=${env.APP_DOMAIN} \
                    --build-arg APP_ENV=${env.APP_ENV} \
                    --build-arg PORT=${env.PORT} \
                    --no-cache \
                    ."""

                    echo 'Bulding docker images'
                    ImageBuild = docker.build("${env.DOCKER_USERNAME}/${env.APP_NAME}:${env.BRANCH_NAME}", buildArgs)
                }
            }
        }

        stage('Push to Registry') {
            when {
                expression {
                    env.BRANCH_NAME == 'staging' || env.BRANCH_NAME == 'master'
                }
            }
            steps {
                script {
                    docker.withRegistry('', 'doocker-hub-something') {
                        ImageBuild.push()
                    }
                }
            }
        }

        stage('Stop & Remove Docker Image') {
            when {
                expression {
                    env.BRANCH_NAME == 'master'
                }
            }
            steps {
                script {
                    def dockerStop = "docker stop ${env.APP_NAME}"
                    def dockerRm = "docker remove ${env.APP_NAME}"
                    sh "${dockerStop} && ${dockerRm}"
                }
            }
        }

        stage('Running on Server Stag') {
            when {
                expression {
                    env.BRANCH_NAME == 'staging'
                }
            }
            agent { label 'agent-129' }
                steps {
                    sh 'kubectl delete pod -l app=svn-portal-be --namespace=your-name-space'
                }
        }

        stage('Running on Server Prod') {
            when {
                expression {
                    env.BRANCH_NAME == 'master'
                }
            }
            steps {
                script {
                    def dockerRun = "docker run -p 8011:8011 -d --name ${env.APP_NAME} ${env.DOCKER_USERNAME}/${env.APP_NAME}:${env.BRANCH_NAME}"
                    def dockerPull = "docker pull ${env.DOCKER_USERNAME}/${env.APP_NAME}:${env.BRANCH_NAME}"
                    def rmBuff = 'echo 1 >  /proc/sys/vm/drop_caches'
                    def rstrtNginx = 'service nginx restart'
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-something', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "echo $PASS | docker login -u $USER --password-stdin && ${dockerPull}"
                    }
                    sh "${dockerRun} && ${rmBuff} && ${rstrtNginx}"
                }
            }
        }
    }
}
