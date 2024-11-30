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
            label 'build-tools-template'
            defaultContainer 'python39'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Git Checkout'
                container('python39') {
                    withCredentials([string(credentialsId: 'gh-sachajw-walle-secret-text', variable: 'GITHUB_PAT')]) {
                        sh 'git clone https://${GITHUB_PAT}@github.com/sachajw/ortelius-jenkins-demo-app.git'
                    }
                }
            }
        }

        stage('Git Committer') {
            steps {
                echo 'Identifying Git Committer'
                container('python39') {
                    script {
                        sh 'git config --global --add safe.directory ${WORKSPACE}'
                        env.GIT_COMMIT_USER = sh(
                            script: "git log -1 --pretty=format:'%an'",
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        stage('Surefire Report') {
            steps {
                echo 'Generating Ortelius Report'
                container('maven39') {
                    sh '''
                        ./mvnw clean install site surefire-report:report
                        tree
                    '''
                }
            }
        }

        stage('Setup') {
            steps {
                echo 'Ortelius Setup'
                container('python39') {
                    sh '''
                        pip install ortelius-cli
                        rm -rf docker-hello-world-spring-boot
                        git clone https://github.com/dstar55/docker-hello-world-spring-boot
                        cd docker-hello-world-spring-boot
                        dh envscript --envvars component.toml --envvars_sh ${WORKSPACE}/dhenv.sh

                        echo Logging into Docker
                        echo ${DHPASS} | docker login -u ${DHUSER} --password-stdin ${DHURL}

                        echo Building and Pushing Docker Image
                        . ${WORKSPACE}/dhenv.sh
                        docker build --tag ${DOCKERREPO}:${IMAGE_TAG} .
                        docker push ${DOCKERREPO}:${IMAGE_TAG}
                        echo export DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERREPO}:${IMAGE_TAG} | cut -d: -f2 | cut -c-12) >> ${WORKSPACE}/dhenv.sh

                        echo Capturing SBOM
                        . ${WORKSPACE}/dhenv.sh
                        curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b .
                        ./syft packages ${DOCKERREPO}:${IMAGE_TAG} --scope all-layers -o cyclonedx-json > ${WORKSPACE}/cyclonedx.json
                        cat ${WORKSPACE}/cyclonedx.json

                        echo Creating Component with Build Data and SBOM
                        . ${WORKSPACE}/dhenv.sh
                        dh updatecomp --rsp component.toml --deppkg "cyclonedx@${WORKSPACE}/cyclonedx.json"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Publishing HTML Report'
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'target/site',
                reportFiles: 'surefire-report.html',
                reportName: 'Surefire Reports'
            ])
        }

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
                            footer: 'Wall-E loves you!',
                            link: env.BUILD_URL,
                            result: currentBuild.currentResult,
                            title: env.JOB_NAME,
                            webhookURL: DISCORD
            }
        }
    }
}
