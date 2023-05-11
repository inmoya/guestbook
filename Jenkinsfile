import java.text.SimpleDateFormat

def TODAY = (new SimpleDateFormat("yyMMddHHmmss")).format(new Date())

pipeline {
    agent none

    environment {
        TODAY_SH = ''
        strDockerTag = "${TODAY}_${BUILD_ID}"
        strDockerImage ="inmoya/cicd_guestbook:${strDockerTag}"
    }

    stages {
        stage('Tag Name 확인') {
            agent any
            steps{
		echo "[Jenkinsfile]"
                echo "inmoya/cicd_guestbook:${TODAY}_${BUILD_ID}"
                script {
                    TODAY_SH = sh(returnStdout: true, script: 'date +%Y%m%d%H%M%S').trim()
                }
                // echo "inmoya/cicd_guestbook:${TODAY_SH}_${BUILD_ID}"
            }
        }
        stage('Checkout') {
            agent any
            steps {
                echo "${TODAY}"
                git branch: 'master', url:'https://github.com/inmoya/guestbook.git'
            }
        }
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.8.4-openjdk-11-slim'
                    args '-v $HOME/.m2:/root/.m2'
                } 
            }
            steps {
                sh "mvn -Dmaven.test.failure.ignore=true clean package"
            }
            post {
                success {
                    archiveArtifacts 'target/*.jar'
                }
            }
        }
        stage('Unit Test') {
            agent any
            steps {
                sh './mvnw test'
            }
            
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }
        stage('SonarQube Analysis') {
            agent any
            steps{
                echo 'SonarQube Analysis'
                /*
                withSonarQubeEnv('SonarQube-Server'){
                    sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=guestbook \
                        -Dsonar.host.url=http://192.168.56.143:9000 \
                        -Dsonar.login=3b57e5204575b3dfa30fe903ef535f6e2749dc0b
                    '''
                }
                */
            }
        }
        stage('SonarQube Quality Gate'){
            agent any
            steps{
                echo 'SonarQube Quality Gate'
                /*
                timeout(time: 2, unit: 'MINUTES') {
                    script{
                        def qg = waitForQualityGate()
                        if(qg.status != 'OK') {
                            echo "NOT OK Status: ${qg.status}"
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else{
                            echo "OK Status: ${qg.status}"
                        }
                    }
                }
                */
            }
        }
        stage('Docker Image Build') {
            agent any
            steps {
                echo 'Docker Image Build'
                script {
                    //oDockImage = docker.build(strDockerImage)
                    oDockImage = docker.build(strDockerImage, "--build-arg VERSION=${strDockerTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            agent any
            steps {
                echo 'Docker Image Push'
                script {
                    docker.withRegistry('', 'DockerHub_inmoya') {
                        oDockImage.push()
                    }
                }
            }
        }
        stage('Staging Deploy') {
            agent any
            steps {
                sshagent(credentials: ['Staging-PrivateKey']) {
                    sh "ssh -o StrictHostKeyChecking=no root@192.168.56.144 docker container rm -f guestbookapp"
                    sh "ssh -o StrictHostKeyChecking=no root@192.168.56.144 docker container run \
                                        -d \
                                        -p 38080:80 \
                                        --name=guestbookapp \
                                        -e MYSQL_IP=192.168.56.140 \
                                        -e MYSQL_PORT=3306 \
                                        -e MYSQL_DATABASE=guestbook \
                                        -e MYSQL_USER=root \
                                        -e MYSQL_PASSWORD=education \
                                        ${strDockerImage} "
                }
            }
        }
        stage ('JMeter LoadTest') {
            agent any
            steps { 
                sh '~/lab/sw/jmeter/bin/jmeter.sh -j jmeter.save.saveservice.output_format=xml -n -t src/main/jmx/guestbook_loadtest.jmx -l loadtest_result.jtl' 
                perfReport filterRegex: '', showTrendGraphs: true, sourceDataFiles: 'loadtest_result.jtl' 
            } 
        }
    }
    post {
        always { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        success { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 성공적으로 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        failure { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'danger'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 실패하였습니다. Details: (<${BUILD_URL} | here >)")
        }
    }
}

