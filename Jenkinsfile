pipeline {
    environment {
        QUAYUSER = credentials('quay-pangarabbit')
        QUAYPASS = credentials('quay-pangarabbit')
        DOCKERREPO = "quay.io"
        DHUSER = credentials('dh-pangarabbit')
        DHPASS = credentials('dh-pangarabbit')
        DHORG = "pangarabbit"
        DHPROJECT = "ortelius-jenkins-demo-app"
        DHURL = "https://ortelius.pangarabbit.com"
        DISCORD = credentials('pangarabbit-discord-jenkins')
    }

    agent {
        kubernetes {
            cloud 'PangaRabbit K8s'
            defaultContainer 'python3'
            inheritFrom 'python3'
            namespace 'app'
        }
    }

    stages {
        stage('Setup') {
            steps {
                sh '''
                    apt-get update && apt-get install -y docker.io
                    pip install ortelius-cli
                    git clone https://github.com/dstar55/docker-hello-world-spring-boot
                    cd docker-hello-world-spring-boot
                    dh envscript --envvars component.toml --envvars_sh ${WORKSPACE}/dhenv.sh
                '''
            }
        }
        stage('Docker Login') {
            steps {
                sh '''
                    docker login -u ${DHUSER} --password-stdin ${DHURL}

                '''
            }
        }
        stage('Build and push image') {
            steps {
                sh '''
                    . ${WORKSPACE}/dhenv.sh
                    if [ -z "${IMAGE_TAG}" ]; then
                        IMAGE_TAG="latest"
                    fi
                    docker build --tag ${DOCKERREPO}:${IMAGE_TAG} .
                    docker push ${DOCKERREPO}:${IMAGE_TAG}

                    # This line determines the docker digest for the image
                    echo export DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERREPO}:${IMAGE_TAG} | cut -d: -f2 | cut -c-12) >> ${WORKSPACE}/dhenv.sh
                '''
            }
        }
        stage('Capture SBOM') {
            steps {
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
        stage('Create Component with Build Data and SBOM') {
            steps {
                sh '''
                    . ${WORKSPACE}/dhenv.sh
                    dh updatecomp --rsp component.toml --deppkg "cyclonedx@${WORKSPACE}/cyclonedx.json"
                '''
            }
        }
        post {
            always {
                discordSend description: "Ortelius Demo App",
                            footer: "Footer Text",
                            link: env.BUILD_URL,
                            result: currentBuild.currentResult,
                            title: env.JOB_NAME,
                            webhookURL: ${DISCORD}
            }
        }
    }
}
