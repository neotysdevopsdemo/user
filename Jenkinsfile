

pipeline {
    agent  { label 'master' }
    tools {
        go 'go'
        jdk 'jdk8'
    }
  environment {
    VERSION="0.1"
    APP_NAME = "user"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    NEOLOAD_ASCODEFILE="$WORKSPACE/test/neoload/user_neoload.yaml"
    NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/user_anomalieDection.json"
    DOCKER_COMPOSE_TEMPLATE="$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose.template"
    DOCKER_COMPOSE_LG_FILE = "$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose-neoload.yml"
    BASICCHECKURI="health"
    CUSTOMERURI="customers"
    CARDSURI="cards"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"
  }
  stages {
      stage('Checkout') {
          agent { label 'master' }
          steps {
              git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                      branch :'master'
          }
      }
    stage('Go build') {
      steps {

          sh '''
            export CODE_DIR=$PWD

            export GOPATH=$PWD

           mkdir -p src/github.com/neotysdevopsdemo/user/
           go get -v github.com/Masterminds/glide
           cp -R ./api src/github.com/neotysdevopsdemo/user/
           cp -R ./db src/github.com/neotysdevopsdemo/user/
           cp -R ./main.go src/github.com/neotysdevopsdemo/user/
           cp -R ./users src/github.com/neotysdevopsdemo/user/
           cp -R ./glide.* src/github.com/neotysdevopsdemo/user/
           cd src/github.com/neotysdevopsdemo/user  && ls -lsa

            go version

           

          
          '''

      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
        steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker pull ${GROUP}/${APP_NAME}:DEV-0.1"
                sh "docker tag ${GROUP}/${APP_NAME}:DEV-0.1 ${TAG_DEV}"
                sh "docker build -t ${TAG}-db:${COMMIT} $WORKSPACE/docker/user-db/"
                sh "docker login --username=${USER} --password=${TOKEN}"
                sh "docker push ${TAG_DEV}"
                sh "docker push ${TAG}-db:${COMMIT}"

            }

        }
    }

     stage('create docker netwrok') {

                                      steps {
                                           sh "docker network create ${APP_NAME} || true"

                                      }
                       }
    stage('Deploy to dev namespace') {
        steps {
            sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},'  $WORKSPACE/docker-compose.yml"
            sh "sed -i 's,TAGDB_TO_REPLACE,${TAG}-db:${COMMIT},'  $WORKSPACE/docker-compose.yml"
            sh "sed -i 's,TO_REPLACE,${APP_NAME},' $WORKSPACE/docker-compose.yml"
            sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

        }

    }

      stage('Start NeoLoad infrastructure') {

                                 steps {
                                            sh "cp -f ${DOCKER_COMPOSE_TEMPLATE} ${DOCKER_COMPOSE_LG_FILE}"
                                            sh "sed -i 's,TO_REPLACE,${APP_NAME},'  ${DOCKER_COMPOSE_LG_FILE}"
                                            sh "sed -i 's,TOKEN_TOBE_REPLACE,$NLAPIKEY,'  ${DOCKER_COMPOSE_LG_FILE}"
                                            sh 'docker-compose -f ${DOCKER_COMPOSE_LG_FILE} up -d'
                                            sleep 15

                                        }

                            }

    stage('Run functional check in dev') {


      steps {
          sleep 90
          sh "cp $WORKSPACE/monspec/user_anomalieDection.json  $WORKSPACE/test/neoload/load_template/custom-resources/"


          sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/CUSTOMER_TO_REPLACE/${CUSTOMERURI}/' $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/CARDS_TO_REPLACE/${CARDSURI}/'  $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/' $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's,JSONFILE_TO_REPLACE,user_anomalieDection.json,' $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  $WORKSPACE/test/neoload/user_neoload.yaml"
          sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,' $WORKSPACE/test/neoload/user_neoload.yaml"


           sh "mkdir $WORKSPACE/test/neoload/neoload_project"
          sh "cp $WORKSPACE/test/neoload/user_neoload.yaml $WORKSPACE/test/neoload/load_template/"
          sh "cd $WORKSPACE/test/neoload/load_template/ ; zip -r $WORKSPACE/test/neoload/neoload_project/neoloadproject.zip ./*"


           sh "docker run --rm \
                     -v $WORKSPACE/test/neoload/neoload_project/:/neoload-project \
                     -e NEOLOADWEB_TOKEN=$NLAPIKEY \
                     -e TEST_RESULT_NAME=FuncCheck_user_${VERSION}_${BUILD_NUMBER} \
                     -e SCENARIO_NAME=UserLoad \
                     -e CONTROLLER_ZONE_ID=defaultzone \
                     -e AS_CODE_FILES=user_neoload.yaml \
                     -e LG_ZONE_IDS=defaultzone:1 \
                     --network ${APP_NAME} --user root \
                      neotys/neoload-web-test-launcher:latest"



      }
    }
    stage('Mark artifact for staging namespace') {
        steps {

            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker login --username=${USER} --password=${TOKEN}"

                sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                sh "docker push ${TAG_STAGING}"
                sh "docker tag ${TAG}-db:${COMMIT} ${TAG}-db-stagging:${VERSION}"
                sh "docker push ${TAG}-db-stagging:${VERSION}"
            }

        }

    }

  }
  post {
      always {
          sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
          sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
          cleanWs()
          sh 'docker volume prune'
      }

        }
}
