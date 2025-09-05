pipeline {
    parameters {
        // Checkbox Parameter, If checked Artifact will upload otherwise skip the stage
        booleanParam(
            defaultValue: false,
            description: 'Upload Artifact?',
            name: 'YES'
        )
    }

    agent {
        // Docker Image where Maven and Docker Installed already
        docker {
            image 'abhishekf5/maven-abhishek-docker-agent:v1'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Static Code Analysis') {
            environment {
                SONAR_URL = "http://3.81.230.133:9000/"
            }
            steps {
                withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh 'mvn sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
                }
            }
        }

        stage('Upload Code Artifacts') {
            when {
                expression { return params.YES } // Run only if checkbox is selected
            }
            agent {
                docker {
                    image 'releases-docker.jfrog.io/jfrog/jfrog-cli-v2:2.2.0'
                    reuseNode true
                }
            }
            environment {
                CI = true
                ARTIFACTORY_ACCESS_TOKEN = credentials('artifactory-access-token')
            }
            steps {
                sh 'jfrog rt upload --url http://54.166.41.28:8082/artifactory/ --access-token ${ARTIFACTORY_ACCESS_TOKEN} target/*.jar springboot-web-app/'
            }
        }

        stage('Build & Push Docker Image') {
            environment {
                DOCKER_IMAGE = "yogesh793/spring-docker:${BUILD_NUMBER}"
                REGISTRY_CREDENTIALS = credentials('docker-cred')
            }
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                    def dockerImage = docker.image("${DOCKER_IMAGE}")
                    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                        dockerImage.push()
                    }
                    sh 'docker rmi ${DOCKER_IMAGE}'
                }
            }
        }

        stage('Updating Deployment File') {
            environment {
                GIT_REPO_NAME = "Springboot-end-to-end"
                GIT_USER_NAME = "yogeshhhh"
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "sharma.yogesh715@gmail.com"
                        git config user.name "yogeshhhh"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        imageTag=$(grep -oP '(?<=spring-docker:)[^ ]+' deployment.yml)
                        sed -i "s/spring-docker:${imageTag}/spring-docker:${BUILD_NUMBER}/" deployment.yml
                        git add deployment.yml
                        git commit -m "Update deployment Image to version ${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:master
                    '''
                }
            }
        }
    }
}
