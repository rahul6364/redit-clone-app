pipeline {
    agent any

    tools{
        jdk 'jdk'
        nodejs 'nodejs'
    }

    environment{
        SCANNER_HOME= tool 'sonarqube'
        APP_NAME= "reddit-clone-app"
        RELEASE= "1.0.0"
        DOCKER_USER= 'rahul6364'
        
        IMAGE_NAME= "${DOCKER_USER}"+ "/" + "${APP_NAME}"
        IMAGE_TAG= "${RELEASE}-${BUILD_NUMBER}"
        SONAR_TOKEN=credentials('sonar-token')
        JENKINS_API_TOKEN=credentials('JENKINS_API_TOKEN')
    }

    stages{
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('clonning the code'){
            when {
                expression { !fileExists('package.json') } // only clone if repo not already checked out
            }
            steps{
                git branch: 'main', url: 'https://github.com/rahul6364/redit-clone-app.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Reddit-clone-CI \
                        -Dsonar.projectName=Reddit-clone-CI \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 2, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        
        stage('install dependencies'){
            steps{
                sh 'npm install'
                sh 'npm audit fix || true'
            }
        }
        stage('trivy fs scan'){
            steps{
               sh 'docker run --rm -v $PWD:/scan aquasec/trivy fs /scan'
            }
        }
        // stage('build and push Docker image'){
        //     steps{
        //         script{ 
        //             docker.withRegistry('',DOCKER_PASS)
        //             {
        //                 docker_image=docker.build ${IMAGE_NAME}
        //             }
        //             docker.withRegistry('',DOCKER_PASS)
        //             {
        //                 docker_image.push("${IMAGE_TAG}")
        //                 docker_image.push('latest')
        //             }
                    
        //         }
               
        //     }
        // }
        stage('docker login'){
            steps{
                 withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                 }
            }
        }
        stage('docker build'){
            steps{
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }
        stage('docker push'){
            steps{
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                }
                sh " docker push ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }
        stage('trivy image scan'){
            steps{
                sh '''
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
        stage('clean the artifacts'){
            steps{
                sh 'docker rmi ${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh """
                        curl -v -k --user rahul:${JENKINS_API_TOKEN} \
                        -X POST \
                        -H 'cache-control: no-cache' \
                        -H 'content-type: application/x-www-form-urlencoded' \
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \
                        https://78525a609e0c.ngrok-free.app/job/Reddit-Clone-CD/buildWithParameters?token=gitOps-token
                    """
                }
            }
        }
  
    }
}


