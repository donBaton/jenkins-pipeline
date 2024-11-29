pipeline {
    agent any
    parameters {
        booleanParam(name: 'CREATE_GITHUB_REPO', defaultValue: false, description: 'Create GitHub Repo')
        string(name: 'PROJECT_NAME', description: 'Name of the project')
        string(name: 'DEFAULT_PORT', defaultValue: '12200', description: 'Default port for deployment')
    }
    environment {
        GITHUB_TOKEN = credentials('<github-credentials>')
        SERVER_HOST = <server-host>
        USER = '<user>'
    }
    stages {
        stage('Check input parameters') {
            steps {
                script {
                    env.APPLICATION_NAME = params.PROJECT_NAME.toLowerCase().replaceAll(' ', '-')
                    echo "Application name: ${env.APPLICATION_NAME}"

                    def port = params.DEFAULT_PORT
                    if (!port.isInteger()) {
                        error("PORT must contain only numbers. You entered: ${port}")
                    }
                    port = port.toInteger()
                    if (port < 1024 || port > 65535) {
                        error("PORT must be between 1024 and 65535. You entered: ${port}")
                    }

                    echo "PORT is valid: ${port}"
                }
            }
        }

        stage('Create GitHub Repo') {
            when {
                expression { params.CREATE_GITHUB_REPO }
            }
            steps {
                sh """
                curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                     -d '{"name": "${APPLICATION_NAME}", "private": true}' \
                     https://api.github.com/user/repos
                """
            }
        }

        stage('Create Jenkins Job') {
            steps {
                script {
                    def jobScript = """
pipelineJob("${APPLICATION_NAME} - build and deploy") {
    parameters {
        stringParam('BRANCH', 'master', 'Branch')
        choiceParam('BUILD_TYPE', ['SNAPSHOT', 'RELEASE', 'PATCH'], 'Select the build type')
        choiceParam('ENVIRONMENT', ['DEV-1', 'PROD', 'SKIP'], 'Select environment or skip deploy')
        stringParam('DEPLOY_PORT', '${params.DEFAULT_PORT}', 'Port to deploy')
    }

    definition {
        cps {
            script('''
            pipeline {
                agent any

                parameters {
                    string(name: 'BRANCH', defaultValue: 'master', description: 'Branch')
                    choice(name: 'BUILD_TYPE', choices: ['SNAPSHOT', 'RELEASE', 'PATCH'], description: 'Select the build type')
                    choice(name: 'ENVIRONMENT', choices: ['DEV-1', 'PROD', 'SKIP'], description: 'Select environment or skip deploy')
                    string(name: 'DEPLOY_PORT', defaultValue: '${params.DEFAULT_PORT}', description: 'Port for the application')

                }

                tools {
                    jdk 'jdk17'
                    maven 'maven'
                }

                environment {
                    APPLICATION_NAME = '${APPLICATION_NAME}'
                    DOCKER_REGISTRY_PORT = '<docker-repository-port>'
                    REPOSITORY_NAME = '<repository-name>'
                    SERVER_HOST = '${SERVER_HOST}'
                    USER = '${USER}'
                }

                stages {

                    stage('Checkout') {
                        steps {
                            git branch: '\$BRANCH', url: 'git@github.com:<gitHubName>/${APPLICATION_NAME}.git', credentialsId: '<jenkins-controller-credentials>'
                        }
                    }

                    stage('Update Version') {
                        steps {
                            script {
                                // Read the current version from pom.xml
                                def pom = readMavenPom file: 'pom.xml'
                                echo "Current Version: \${pom.version}"

                                // Increment the patch version
                                def versionParts = pom.version.tokenize('.')
                                def newVersion
                                if (params.BUILD_TYPE == 'RELEASE') {
                                    newVersion = "\${versionParts[0]}.\${(versionParts[1].toInteger() + 1)}.\${(versionParts[2])}"
                                } else if (params.BUILD_TYPE == 'SNAPSHOT') {
                                    newVersion = "\${versionParts[0]}.\${versionParts[1]}.\${(versionParts[2])}-b\${BUILD_NUMBER}"
                                } else {
                                    newVersion = "\${versionParts[0]}.\${versionParts[1]}.\${(versionParts[2].toInteger() + 1)}"
                                }
                                echo "New Version: \${newVersion}"

                                // Update pom.xml with the new version
                                sh "mvn versions:set -DnewVersion=\${newVersion}"
                                sh "mvn versions:commit"

                                // Store the new version in the environment for later stages
                                env.NEW_VERSION = newVersion
                            }
                        }
                    }

                    stage('Build') {
                        steps {
                            sh 'export PATH=\$PATH:/opt/maven/bin'
                            sh 'mvn clean package'
                        }
                    }

                    stage('Docker Build & Push') {
                        steps {
                            script {
                                withCredentials([usernamePassword(credentialsId: 'nexus-creds', passwordVariable: 'DOCKER_REGISTRY_PASSWORD', usernameVariable: 'DOCKER_REGISTRY_USERNAME')]) {
                                    echo "Building Docker image with version: \${env.NEW_VERSION}"
                                    def REPOSITORY_PATH = \"\"\"\${SERVER_HOST}:\${DOCKER_REGISTRY_PORT}/\${REPOSITORY_NAME}/\${APPLICATION_NAME}\"\"\"
                                    if (params.BUILD_TYPE == 'SNAPSHOT') {
                                        REPOSITORY_PATH = REPOSITORY_PATH + "/snapshot"
                                    } else {
                                        REPOSITORY_PATH = REPOSITORY_PATH + "/release"
                                    }
                                    env.REPOSITORY_PATH = REPOSITORY_PATH
                                    sh \"\"\"
                                    #!/bin/bash
                                    docker build --build-arg APP_VERSION=\${NEW_VERSION} -t \${REPOSITORY_PATH}:\${NEW_VERSION} .
                                    echo \${DOCKER_REGISTRY_PASSWORD} | docker login \${SERVER_HOST}:\${DOCKER_REGISTRY_PORT} --username \${DOCKER_REGISTRY_USERNAME} --password-stdin
                                    docker push \${REPOSITORY_PATH}:\${NEW_VERSION}
                                    \"\"\"
                                }
                            }
                        }
                    }

                    stage('Deploy') {
                        when {
                            expression { params.ENVIRONMENT != 'SKIP' }
                        }

                        steps {
                            sshagent(['nas']) {
                                script {
                                    echo "Checking for existing Docker image..."
                                    def oldImageName = sh(
                                        script: \"\"\"
                                        ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                            echo \\\\\$(docker inspect --format '{{.Config.Image}}' "\${APPLICATION_NAME}" 2>/dev/null):\\\\"\\\\"
                                        '
                                        \"\"\",
                                        returnStdout: true
                                    ).trim()
                                    echo "Old Docker image found: \${oldImageName}"

                                    echo "Stopping existing Docker container..."
                                    sh \"\"\"
                                    ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                        docker stop \${APPLICATION_NAME} || true
                                        docker rm \${APPLICATION_NAME} || true
                                    '
                                    \"\"\"

                                    if (oldImageName) {
                                        echo "Deleting old Docker image..."
                                        sh \"\"\"
                                        ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                            docker rmi \${oldImageName} || true
                                        '
                                        \"\"\"
                                    } else {
                                        echo "No old Docker image found for \${APPLICATION_NAME}."
                                    }

                                    echo "Pulling new Docker image from Nexus..."
                                    sh \"\"\"
                                    ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                        docker pull \${REPOSITORY_PATH}:\${NEW_VERSION}
                                    '
                                    \"\"\"

                                    echo "Running new Docker container with port \${params.DEFAULT_PORT}..."
                                    sh \"\"\"
                                    ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                        docker run -d --name \${APPLICATION_NAME} -p \${params.DEPLOY_PORT}:8080 \
                                            \${REPOSITORY_PATH}:\${NEW_VERSION}
                                        '
                                    \"\"\"
                                }
                            }
                        }
                    }

                    stage('Update version in repo') {

                        when {
                            expression { params.BUILD_TYPE != 'SNAPSHOT' }
                        }

                        steps {
                            script {
                                // Commit and push the updated pom.xml to the repository
                                sh \"\"\"
                                #!/bin/bash
                                git config user.name "jenkins"
                                git config user.email "<e-mail>"
                                git add pom.xml
                                git commit -m "Increment version to \${NEW_VERSION}"
                                git push origin \${BRANCH}
                                \"\"\"
                            }
                        }
                    }

                }

                post {
                    always {
                        script {
                            currentBuild.description = "\${APPLICATION_NAME}-\${env.NEW_VERSION} : \${params.DEPLOY_PORT}"
                        }
                    }
                }

            }
            ''')
            sandbox(true)
        }
    }
}
"""
                    writeFile file: "${APPLICATION_NAME.replace('-', '_')}_job.groovy", text: jobScript
                    jobDsl targets: "${APPLICATION_NAME.replace('-', '_')}_job.groovy"
                }
            }
        }
    }
}