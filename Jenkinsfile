pipeline {
    environment {
        QUAYUSER = credentials('quay-pangarabbit')
        QUAYPASS = credentials('quay-pangarabbit')
        DOCKERREPO = 'quay.io'
        DHUSER = credentials('dh-pangarabbit')
        DHPASS = credentials('dh-pangarabbit')
        DHORG = 'pangarabbit'
        DHPROJECT = 'ortelius-jenkins-demo-app'
        DHURL = 'https://ortelius.pangarabbit.com'
        DISCORD = credentials('pangarabbit-discord-jenkins')
        REPORT_DIR = "${env.JENKINS_HOME}/reports"
    }

    agent {
        kubernetes {
            cloud 'PangaRabbit K3s'
            defaultContainer 'python3'
            inheritFrom 'python3'
            namespace 'app'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'GitCheckout'
                container('python3') {
                    withCredentials([string(credentialsId: 'gh-sachajw-walle-secret-text', variable: 'GITHUB_PAT')]) {
                        sh 'git clone https://${GITHUB_PAT}@github.com/dstar55/docker-hello-world-spring-boot.git'
                    }
                }
            }
        }
        stage('Git Committer') {
            steps {
                echo 'Git Committer'
                container('python3') {
                    script {
                        // Mark the directory as safe to prevent Git errors
                        sh 'git config --global --add safe.directory ${WORKSPACE}'

                        // Get the user who made the latest commit
                        env.GIT_COMMIT_USER = sh(
                        script: "git log -1 --pretty=format:'%an'",
                        returnStdout: true
                    ).trim()
                    }
                }
            }
        }
        stage('Setup') {
            steps {
                echo 'Setup Python Container'
                container('python3') {
                    sh '''
                        apt-get update && apt-get install -y docker.io
                        pip install ortelius-cli
                        rm -rf docker-hello-world-spring-boot
                        git clone https://github.com/dstar55/docker-hello-world-spring-boot
                        cd docker-hello-world-spring-boot
                        dh envscript --envvars component.toml --envvars_sh ${WORKSPACE}/dhenv.sh
                    '''
                }
            }
        }
        stage('Docker Login') {
            steps {
                echo 'Docker Login'
                container('python3') {
                    sh '''
                        echo ${DHPASS} | docker login -u ${DHUSER} --password-stdin ${DHURL}
                    '''
                }
            }
        }
        stage('Build and Push Image') {
            steps {
                echo 'Build and Push Image'
                container('python3') {
                    sh '''
                        . ${WORKSPACE}/dhenv.sh
                        docker build --tag ${DOCKERREPO}:${IMAGE_TAG} .
                        docker push ${DOCKERREPO}:${IMAGE_TAG}

                        # This line determines the docker digest for the image
                        echo export DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERREPO}:${IMAGE_TAG} | cut -d: -f2 | cut -c-12) >> ${WORKSPACE}/dhenv.sh
                    '''
                }
            }
        }
        stage('Capture SBOM') {
            steps {
                echo 'Capture SBOM'
                container('python3') {
                    sh '''
                        . ${WORKSPACE}/dhenv.sh
                        # install Syft
                        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b .

                        # create the SBOM
                        ./syft packages ${DOCKERREPO}:${IMAGE_TAG} --scope all-layers -o cyclonedx-json > ${WORKSPACE}/cyclonedx.json

                        # display the SBOM
                        cat ${WORKSPACE}/cyclonedx.json
                    '''
                }
            }
        }
        stage('Create Component with Build Data and SBOM') {
            steps {
                echo 'Create Component with Build Data and SBOM'
                container('python3') {
                    sh '''
                        . ${WORKSPACE}/dhenv.sh
                        dh updatecomp --rsp component.toml --deppkg "cyclonedx@${WORKSPACE}/cyclonedx.json"
                    '''
                }
            }
        }
    }
        stage('Ortelius Report') {
            steps {
                echo 'Ortelius Report'
                    sh '''
                        sh './mvnw clean install site surefire-report:report'
                        sh 'tree'
                    '''
            }
        }
    }
    post {
        always {
            echo 'Sending Discord Notification'
            withCredentials([string(credentialsId: 'pangarabbit-discord-jenkins', variable: 'DISCORD')]) {
                discordSend description: """
                                    Result: ${currentBuild.currentResult}
                                    Service: ${env.JOB_NAME}
                                    Build Number: [#${env.BUILD_NUMBER}](${env.BUILD_URL})
                                    Branch: ${env.GIT_BRANCH}
                                    Commit User: ${env.GIT_COMMIT_USER}
                                    Duration: ${currentBuild.durationString}
                                """,
                                footer: 'Wall-E love you!',
                                link: env.BUILD_URL,
                                result: currentBuild.currentResult,
                                title: env.JOB_NAME,
                                webhookURL: DISCORD
            }
        }
    }
    post {
        always {
            echo 'Publishing HTML Report'
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'target/site',
                reportFiles: 'surefire-report.html',
                reportName: 'SurefireReports',
                reportTitles: ''.
                useWrapperFileDirectly: true
            ])
        }
    }
