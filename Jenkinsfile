#!/usr/bin/env groovy

pipeline {
  agent any
  environment {
    DOCKER_CONTAINER_SERVER_NAME = "DB-backup-restore-${BUILD_NUMBER}"
  }

  stages {
    stage('Prepare') {
      steps {
        sh '[ -d mysql_data ] || mkdir mysql_data'
        sh '[ -d mysql_out ] || mkdir mysql_out'
        sh 'sudo chown 1001:1001 ./mysql_data && sudo chown 1001:1001 ./mysql_out'
        sh 'ls -la `pwd`'
      }
    }
    stage('Start server') {
      steps {
        sh "docker run --name ${DOCKER_CONTAINER_SERVER_NAME} --rm -i -e MYSQL_ROOT_PASSWORD=secret -v `pwd`/mysql_data:/var/lib/mysql:rw percona/percona-server:5.7"
        sh 'ls -la `pwd`/mysql_data'
      }
    }
  }
  post {
    always {
      cleanWs()
      sh "docker stop ${DOCKER_CONTAINER_SERVER_NAME}"
    }
  }
}