node {
    stage('SSH transfer') {
             script {
                  sshPublisher(
                   continueOnError: false, failOnError: true,
                   publishers: [
                    sshPublisherDesc(
                     configName: "${env.SSH_CONFIG_NAME}",
                     verbose: true,
                        transfers: [
                          sshTransfer(
                               sourceFiles: "word_press/docker-compose.yml",
                               execCommand: "cd /home/mms/word_press/ && docker-compose up -d"
                            )
                        ])
                   ])
                }
    }
    stage ('Code analyse') {
        sh 'echo "hello"'
    }
    stage ('Unit test') {
        sh 'echo "Tests will back"'
    }
    stage ('Build') {
        sh 'echo "Building"'
    }
}