pipeline {
    agent any
    parameters {
        string(name: 'PROJECT_NAME', description: 'Name of the project')
        booleanParam(name: 'CREATE_GITHUB_REPO', defaultValue: false, description: 'Create GitHub repo')
        booleanParam(name: 'CREATE_JENKINS_JOB', defaultValue: false, description: 'Create Jenkins job')
        string(name: 'DEFAULT_PORT', defaultValue: '122XX', description: 'Default port for deployment (from 1025 to 65535)')
        booleanParam(name: 'CONFIGURE_DEV_DATABASE', defaultValue: false, description: 'Configure dev database')
        password(name: 'DATABASE_PASSWORD_DEV', description: 'Dev password')
        booleanParam(name: 'CONFIGURE_PROD_DATABASE', defaultValue: false, description: 'Configure prod database')
        password(name: 'DATABASE_PASSWORD_PROD', description: 'Prod password')
    }
    environment {
        GITHUB_TOKEN = credentials('github-token-2')
        SERVER_HOST = "<server ip>"
        USER = '<jenkins user>'
        JENKINS_USER_EMAIL = '<jenkins email>'
        POSTGRES_CONTAINER_NAME = '<postgres container name>'
        DB_ADMIN_USER = '<postgres admin username>'
        DOCKER_REGISTRY_PORT = '<docker registry port>'
        REPOSITORY_NAME = '<docker repository name>'
        VAULT_INNER_IP = '<vault docker network ip>'
        VAULT_PORT = '<vault port>'
    }
    stages {
        stage('Preparation') {
            steps {
                script {
                    env.APPLICATION_NAME = params.PROJECT_NAME.toLowerCase().replaceAll(' ', '-')
                    echo "Application name: ${env.APPLICATION_NAME}"
                }
            }
        }

        stage('Create GitHub Repo') {
            when {
                expression { params.CREATE_GITHUB_REPO }
            }
            steps {
                echo "Creating GitHub repository from template..."

                sh """
                    curl -X POST -H "Authorization: token ${GITHUB_TOKEN}" \
                    -d '{"name": "${APPLICATION_NAME}", "private": true}' \
                    https://api.github.com/user/repos

                    rm -rf springboot-template ${APPLICATION_NAME}

                    git clone -b master git@github.com:donBaton/springboot-template.git
                    mv springboot-template ${APPLICATION_NAME}
                    cd ${APPLICATION_NAME}

                    rm -rf .git
                    git init
                    git branch -m develop

                    git config user.name '${USER}'
                    git config user.email '${JENKINS_USER_EMAIL}'

                    sed -i 's|<artifactId>template</artifactId>|<artifactId>${APPLICATION_NAME}</artifactId>|g' pom.xml
                    sed -i 's|COPY target/template-|COPY target/${APPLICATION_NAME}-|g' Dockerfile
                    sed -i 's|name: template|name: ${APPLICATION_NAME}|g' src/main/resources/application.yml

                    git add .
                    git commit -m "init"

                    git remote add origin git@github.com:donBaton/${APPLICATION_NAME}.git
                    git push -u origin develop
                """
            }
        }

        stage('Create Jenkins Job') {
            when {
                expression { params.CREATE_JENKINS_JOB }
            }
            steps {
                script {
                    def port = params.DEFAULT_PORT
                    if (!port.isInteger()) {
                        error("PORT must contain only numbers. You entered: ${port}")
                    }
                    port = port.toInteger()
                    if (port < 1024 || port > 65535) {
                        error("PORT must be between 1024 and 65535. You entered: ${port}")
                    }

                    echo "PORT is valid: ${port}"

                    def jobScript = """
pipelineJob("${APPLICATION_NAME} - build and deploy") {
    parameters {
        stringParam('BRANCH', 'develop', 'Branch')
        choiceParam('BUILD_TYPE', ['SNAPSHOT', 'RELEASE', 'PATCH'], 'Select the build type')
        choiceParam('ENVIRONMENT', ['DEV', 'PROD', 'SKIP'], 'Select environment or skip deploy')
        stringParam('DEPLOY_PORT', '${params.DEFAULT_PORT}', 'Port to deploy')
    }

    definition {
        cps {
            script('''
            pipeline {
                agent any

                parameters {
                    string(name: 'BRANCH', defaultValue: 'develop', description: 'Branch')
                    choice(name: 'BUILD_TYPE', choices: ['SNAPSHOT', 'RELEASE', 'PATCH'], description: 'Select the build type')
                    choice(name: 'ENVIRONMENT', choices: ['DEV', 'PROD', 'SKIP'], description: 'Select environment or skip deploy')
                    string(name: 'DEPLOY_PORT', defaultValue: '${params.DEFAULT_PORT}', description: 'Port for the application')

                }

                tools {
                    jdk 'jdk17'
                    maven 'maven'
                }

                environment {
                    APPLICATION_NAME = '${APPLICATION_NAME}'
                    DOCKER_REGISTRY_PORT = '${DOCKER_REGISTRY_PORT}'
                    REPOSITORY_NAME = '${REPOSITORY_NAME}'
                    SERVER_HOST = '${SERVER_HOST}'
                    USER = '${USER}'
                }

                stages {

                    stage('Checkout') {
                        steps {
                            git branch: '\$BRANCH', url: 'git@github.com:donBaton/${APPLICATION_NAME}.git', credentialsId: 'jenkins-server'
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
                                    def usernameKey = "username.\${params.ENVIRONMENT.toLowerCase()}"
                                    def passwordKey = "password.\${params.ENVIRONMENT.toLowerCase()}"

                                    withVault([vaultSecrets: [[path: "kv/projects/\${APPLICATION_NAME}/db", secretValues: [
                                        [envVar: 'DB_USERNAME', vaultKey: usernameKey],
                                        [envVar: 'DB_PASSWORD', vaultKey: passwordKey]
                                    ]]]]) {
                                        script {
                                            echo "Checking for existing Docker image..."
                                            def oldImageName = sh(
                                                script: \"\"\"
                                                ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                                    echo \\\\\$(docker inspect --format '{{.Config.Image}}' "\${APPLICATION_NAME}" 2>/dev/null)\\\\"\\\\"
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

                                            echo "Running new Docker container with port \${params.DEPLOY_PORT}..."
                                            sh \"\"\"
                                            ssh -o StrictHostKeyChecking=no \${USER}@\${SERVER_HOST} '
                                                docker run -d \\\\
                                                    --name \${APPLICATION_NAME} \\\\
                                                    -e SPRING_PROFILES_ACTIVE=\${params.ENVIRONMENT.toLowerCase()} \\\\
                                                    -e DB_USERNAME="\${DB_USERNAME}" \\\\
                                                    -e DB_PASSWORD="\${DB_PASSWORD}" \\\\
                                                    -p \${params.DEPLOY_PORT}:8080  \\\\
                                                    \${REPOSITORY_PATH}:\${NEW_VERSION}
                                                '
                                            \"\"\"
                                        }
                                    }

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
                                git config user.name "${USER}"
                                git config user.email "${JENKINS_USER_EMAIL}"
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

        stage('Configure databases') {
            when {
                expression { params.CONFIGURE_DEV_DATABASE || params.CONFIGURE_PROD_DATABASE }
            }
            steps {
                sshagent(['nas']) {
                    script {
                        if (params.CONFIGURE_DEV_DATABASE) {
                            echo "Configuring dev database..."
                            env.DATABASE_USER_DEV = "${APPLICATION_NAME.replace('-', '_')}_owner_dev"
                            setupDatabase('dev', "${DATABASE_USER_DEV}", "${params.DATABASE_PASSWORD_DEV}")
                        }

                        if (params.CONFIGURE_PROD_DATABASE) {
                            echo "Configuring prod database..."
                            env.DATABASE_USER_PROD = "${APPLICATION_NAME.replace('-', '_')}_owner_prod"
                            setupDatabase('prod', "${DATABASE_USER_PROD}", "${params.DATABASE_PASSWORD_PROD}")
                        }
                    }
                }
            }
        }

        stage('Save secrets to vault') {
            when {
                expression { params.CONFIGURE_DEV_DATABASE || params.CONFIGURE_PROD_DATABASE }
            }
            steps {
                withCredentials([string(credentialsId: 'vault-token', variable: 'VAULT_TOKEN')]) {
                    script {
                        def secretsData = [:]

                        if (params.CONFIGURE_DEV_DATABASE) {
                            echo "Adding dev database secrets to data..."
                            secretsData["username.dev"] = DATABASE_USER_DEV
                            secretsData["password.dev"] = env.DATABASE_PASSWORD_DEV
                        }

                        if (params.CONFIGURE_PROD_DATABASE) {
                            echo "Adding prod database secrets to data..."
                            secretsData["username.prod"] = DATABASE_USER_PROD
                            secretsData["password.prod"] = env.DATABASE_PASSWORD_PROD
                        }

                        def secretsJson = new groovy.json.JsonBuilder([data: secretsData]).toString()

                        echo "Saving secrets to Vault..."
                        sh """
                            curl --header "X-Vault-Token: ${VAULT_TOKEN}" \
                                 --request POST \
                                 --data '${secretsJson}' \
                                 http://${VAULT_INNER_IP}:${VAULT_PORT}/v1/kv/data/projects/${APPLICATION_NAME}/db
                        """
                    }
                }
            }
        }

    }
}

def setupDatabase(dbName, dbUserName, dbUserPassword) {
    def schemaName = env.APPLICATION_NAME.replaceAll('-', '_')
    sh """
    ssh -o StrictHostKeyChecking=no ${env.USER}@${env.SERVER_HOST} "docker exec ${env.POSTGRES_CONTAINER_NAME} psql -U ${env.DB_ADMIN_USER} -d ${dbName} -c \\"
    CREATE USER ${dbUserName} WITH PASSWORD '${dbUserPassword}'; \\
    CREATE SCHEMA IF NOT EXISTS ${schemaName}; \\
    ALTER SCHEMA ${schemaName} OWNER TO ${dbUserName}; \\
    GRANT ALL PRIVILEGES ON SCHEMA ${schemaName} TO ${dbUserName}; \\
    GRANT CONNECT ON DATABASE ${dbName} TO ${dbUserName}; \\
    GRANT USAGE ON SCHEMA ${schemaName} TO ${dbUserName}; \\
    GRANT CREATE ON SCHEMA ${schemaName} TO ${dbUserName}; \\
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA ${schemaName} TO ${dbUserName}; \\
    ALTER DEFAULT PRIVILEGES IN SCHEMA ${schemaName} GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ${dbUserName}; \\
    ALTER ROLE ${dbUserName} SET search_path TO ${schemaName};\\""
    """
}