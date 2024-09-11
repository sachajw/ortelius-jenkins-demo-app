pipeline {
  environment {
    DHUSER = "admin"
    DHPASS = "admin"
    DHORG = "pangarabbit"
    DHPROJECT = "ortelius-jenkins-demo-app"
    DOCKERREPO = "quay.io/ortelius/hello-world"
    DHURL = "https://ortelius.pangarabbit.com"
  }

  agent {
    kubernetes {
      yaml """
      apiVersion: v1
      kind: Pod
      spec:
        containers:
        - name: python-jenkins-agent
          image: python:3.9
          tty: true
      """
    }
  }

  stages {
    stage('Setup') {
      steps {
        container('python-jenkins-agent') {
        sh '''
            pip install deployhub
            git clone https://github.com/dstar55/docker-hello-world-spring-boot
            cd docker-hello-world-spring-boot
            dh envscript --envvars component.toml --envvars_sh ${WORKSPACE}/dhenv.sh
        '''
        }
      }
    stage('Build and push image') {
      steps{
        sh '''
            source ${WORKSPACE}/dhenv.sh
            docker build --tag ${DOCKERREPO}:${IMAGE_TAG} .
            docker push ${DOCKERREPO}:${IMAGE_TAG}

            # This line determines the docker digest for the image
            echo export DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${DOCKERREPO}:${IMAGE_TAG} | cut -d: -f2 | cut -c-12) >> ${WORKSPACE}/dhenv.sh
        '''
      }
    }
    stage('Capture SBOM') {
      steps{
        sh '''
            source ${WORKSPACE}/dhenv.sh
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
      steps{
        sh '''
            source ${WORKSPACE}/dhenv.sh
            dh updatecomp --rsp component.toml --deppkg "cyclonedx@${WORKSPACE}/cyclonedx.json"
        '''
        }
      }
    }
  }
}