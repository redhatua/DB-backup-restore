#!/usr/bin/env groovy

pipeline {
  agent any
  environment {
    DOCKER_CONTAINER_SERVER_NAME = "DB-backup-restore-${BUILD_NUMBER}"
    DOCKER_SERVER_CMD = "docker exec -i ${DOCKER_CONTAINER_SERVER_NAME} sh -c"
  }

  stages {
    stage('Prepare') {
      steps {
        sh '[ -d mysql_data ] || mkdir mysql_data'
        sh '[ -d mysql_out ] || mkdir mysql_out'
        sh 'sudo chown 1001:1001 ./mysql_data && sudo chown 1001:1001 ./mysql_out'
      }
    }
    stage('Start source server') {
      steps {
        sh "docker run --name ${DOCKER_CONTAINER_SERVER_NAME}  -d -e MYSQL_ROOT_PASSWORD=secret -v `pwd`/mysql_data:/var/lib/mysql:rw percona/percona-server:5.7"
        sh "${DOCKER_SERVER_CMD} 'while ! mysqladmin ping -u root -psecret -h localhost -P 3306 --protocol=tcp ; do sleep 30; done'"
        sh 'echo MySQL server is up and running'
      }
    }
    stage('Prepare source server') {
      steps {
        sh "${DOCKER_SERVER_CMD} 'echo \"CREATE DATABASE bench; USE bench; CREATE TABLE IF NOT EXISTS test (id INT AUTO_INCREMENT PRIMARY KEY, text VARCHAR(255), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP) ENGINE=INNODB;\" | mysql -uroot -psecret -h localhost -P 3306 --protocol=tcp'"
        sh "${DOCKER_SERVER_CMD} 'for i in \$(for i in \$(seq -w 1 \$(shuf -i 100-200 -n 1)); do echo \$i; done) ; do mysql -uroot -psecret -h localhost -P 3306 --protocol=tcp -e \"INSERT INTO bench.test (text) VALUES (\$i)\" ; done"
        sh "${DOCKER_SERVER_CMD} 'mysql -uroot -psecret -h localhost -P 3306 --protocol=tcp -e \"SELECT MAX(id) FROM bench.test\"'"
      }
    }
  }
  post {
    always {
      sh 'sudo chmod -R 0777 .'
      cleanWs()
      sh "docker stop ${DOCKER_CONTAINER_SERVER_NAME} || true"
    }
  }
}