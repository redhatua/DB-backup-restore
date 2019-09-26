#!/usr/bin/env groovy

pipeline {
  agent any
  environment {
    DOCKER_CONTAINER_SOURCE_NAME = "DB-backup-restore-source-${BUILD_NUMBER}"
    DOCKER_CONTAINER_NEW_NAME = "DB-backup-restore-new-${BUILD_NUMBER}"
    DOCKER_CONTAINER_BACKUP_NAME = "DB-backup-restore-backup-${BUILD_NUMBER}"
    DOCKER_SOURCE_CMD = "docker exec -i ${DOCKER_CONTAINER_SOURCE_NAME} sh -c"
    DOCKER_NEW_CMD = "docker exec -i ${DOCKER_CONTAINER_NEW_NAME} sh -c"
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
        sh "docker run --name ${DOCKER_CONTAINER_SOURCE_NAME}  -d -e MYSQL_ROOT_PASSWORD=secret -v `pwd`/mysql_data:/var/lib/mysql:rw percona/percona-server:5.7"
        sh "${DOCKER_SOURCE_CMD} 'while ! MYSQL_PWD=secret mysqladmin ping -u root -h localhost -P 3306 --protocol=tcp ; do sleep 30; done'"
        sh 'echo MySQL server is up and running'
      }
    }
    stage('Prepare source server') {
      steps {
        sh "${DOCKER_SOURCE_CMD} 'echo \"CREATE DATABASE bench; USE bench; CREATE TABLE IF NOT EXISTS test (id INT AUTO_INCREMENT PRIMARY KEY, text VARCHAR(255), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP) ENGINE=INNODB;\" | MYSQL_PWD=secret mysql -uroot -h localhost -P 3306 --protocol=tcp'"
        sh "${DOCKER_SOURCE_CMD} 'for i in \$(for i in \$(seq -w 1 \$(shuf -i 100-200 -n 1)); do echo \$i; done) ; do MYSQL_PWD=secret mysql -uroot -h localhost -P 3306 --protocol=tcp -e \"INSERT INTO bench.test (text) VALUES (\$i)\" ; done'"
      }
    }
    stage('Backup source server') {
      steps {
        sh "docker run --name ${DOCKER_CONTAINER_BACKUP_NAME} --rm -i -v `pwd`/mysql_data:/var/lib/mysql -v `pwd`/mysql_out:/xtrabackup_backupfiles perconalab/percona-xtrabackup --backup --host=\$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${DOCKER_CONTAINER_SOURCE_NAME}) --user=root --password=secret"
      }
    }
    stage('Prepare new server') {
      steps {
        sh 'sudo chown -R 1001:1001 ./mysql_out && sudo chmod -R 0777 ./mysql_out'
        sh "docker run --name ${DOCKER_CONTAINER_NEW_NAME} -d -v `pwd`/mysql_out:/var/lib/mysql:rw percona/percona-server:5.7"
        sh "${DOCKER_NEW_CMD} 'while ! MYSQL_PWD=secret mysqladmin ping -u root -h localhost -P 3306 --protocol=tcp ; do sleep 30; done'"
        sh 'echo MySQL server is up and running'
      }
    }
    stage('Validate') {
      steps {
        script {
          env.SOURCE_MAX_ID = sh(returnStdout: true, script: "${DOCKER_SOURCE_CMD} 'MYSQL_PWD=secret mysql -uroot -h localhost -P 3306 --protocol=tcp -s -r -N -e \"SELECT MAX(id) FROM bench.test\"'").trim()
          env.NEW_MAX_ID = sh(returnStdout: true, script: "${DOCKER_NEW_CMD} 'MYSQL_PWD=secret mysql -uroot -h localhost -P 3306 --protocol=tcp -s -r -N -e \"SELECT MAX(id) FROM bench.test\"'").trim()
          if (env.SOURCE_MAX_ID == env.NEW_MAX_ID) {
            echo 'Backup valid'
          } else {
            echo 'Results not equal'
            sh 'exit 1'
          }
        }
      }
    }
  }
  post {
    always {
      sh 'sudo chmod -R 0777 .'
      cleanWs()
      sh "docker stop ${DOCKER_CONTAINER_SOURCE_NAME} || true"
      sh "docker stop ${DOCKER_CONTAINER_NEW_NAME} || true"
    }
  }
}