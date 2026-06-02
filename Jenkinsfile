pipeline {

    agent any

    tools {
        jdk 'JDK-17'
    }

    environment {

        // Organization Details
        ANYPOINT_ORG_ID      = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'
        EXCHANGE_GROUP_ID    = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'

        // Connected App
        CONNECTED_APP_ID     = 'c09ce27b91784380bf8646a62ec65c9d'
        CONNECTED_APP_SECRET = '8b7B8D66306E47a29f4e3007547774f4'

        // Application Details
        APP_NAME             = 'jenkins-project-api'

        // CloudHub 2.0 Settings
        TARGET               = 'Cloudhub-US-East-2'
        CH2_REGION           = 'us-east-1'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.17'

        // Maven Settings
        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'

        // Environment IDs
        SANDBOX_ENV_ID       = '39886fb6-d89f-4a56-a400-15ce44a87bbc'
        DEV_ENV_ID           = '2ce5a5ae-ce2a-41a0-96fc-68a5e22df760'
        UAT_ENV_ID           = '864fd6b3-5b44-41c5-89e9-d76573a27639'
        PROD_ENV_ID          = 'ac61ddf1-1c0f-4c0f-9dfe-f305f90dbb8a'
    }

    parameters {

        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox,dev,uat,prod',
            description: 'Comma separated environment names'
        )

        booleanParam(
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish artifact to Exchange'
        )

        booleanParam(
            name: 'DEPLOY_TO_CLOUDHUB',
            defaultValue: true,
            description: 'Deploy to CloudHub 2.0'
        )
    }

    stages {

        stage('Checkout') {

            steps {

                echo "Checking out source code..."

                checkout scm
            }
        }

        stage('Build & Test') {

            steps {

                echo "Running tests..."

                bat """
                    mvn clean test ^
                    -s %MAVEN_SETTINGS%
                """
            }

            post {

                always {

                    junit allowEmptyResults: true,
                           testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {

            steps {

                echo "Packaging Mule Application..."

                bat """
                    mvn clean package ^
                    -DskipTests ^
                    -s %MAVEN_SETTINGS%
                """
            }
        }

        stage('Publish To Exchange') {

            when {
                expression { params.PUBLISH_TO_EXCHANGE }
            }

            steps {

                echo "Publishing artifact to Exchange..."

                bat """
                    mvn deploy ^
                    -DskipTests ^
                    -s %MAVEN_SETTINGS% ^
                    -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                    -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                    -DrepositoryId=anypoint-exchange-v3 ^
                    -Durl=https://maven.anypoint.mulesoft.com/api/v3/organizations/%EXCHANGE_GROUP_ID%/maven
                """
            }
        }

        stage('Deploy To Multiple Environments') {

            when {
                expression { params.DEPLOY_TO_CLOUDHUB }
            }

            steps {

                script {

                    def envMap = [
                        "Sandbox": env.SANDBOX_ENV_ID,
                        "dev"    : env.DEV_ENV_ID,
                        "uat"    : env.UAT_ENV_ID,
                        "prod"   : env.PROD_ENV_ID
                    ]

                    def environments = params.DEPLOY_ENVS
                        .split(',')
                        .collect { it.trim() }

                    def successEnvs = []
                    def failedEnvs  = []

                    echo "======================================="
                    echo "Selected Environments:"
                    echo "${environments}"
                    echo "======================================="

                    for (String envName : environments) {

                        if (!envMap.containsKey(envName)) {

                            echo "Environment not configured: ${envName}"
                            failedEnvs.add(envName)
                            continue
                        }

                        def envId = envMap[envName]

                        def appName = "${APP_NAME}-${envName.toLowerCase()}"

                        echo "======================================="
                        echo "Deploying Application"
                        echo "Environment : ${envName}"
                        echo "Environment ID : ${envId}"
                        echo "Application : ${appName}"
                        echo "======================================="

                        def deployStatus = bat(
                            returnStatus: true,
                            script: """
                                mvn deploy ^
                                -DmuleDeploy ^
                                -DskipTests ^
                                -s %MAVEN_SETTINGS% ^
                                -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                                -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                                -Danypoint.environment=${envName} ^
                                -Danypoint.environmentId=${envId} ^
                                -Dcloudhub2.applicationName=${appName} ^
                                -Dcloudhub2.target=%TARGET% ^
                                -Dcloudhub2.region=%CH2_REGION% ^
                                -Dcloudhub2.replicas=%CH2_REPLICAS% ^
                                -Dcloudhub2.vCores=%CH2_VCORES% ^
                                -Dcloudhub2.muleVersion=%RUNTIME_VERSION% ^
                                -Danypoint.businessGroup=%ANYPOINT_ORG_ID%
                            """
                        )

                        if (deployStatus == 0) {

                            successEnvs.add(envName)

                            echo "SUCCESS : ${envName}"

                        } else {

                            failedEnvs.add(envName)

                            echo "FAILED : ${envName}"
                        }
                    }

                    echo ""
                    echo "======================================="
                    echo "DEPLOYMENT SUMMARY"
                    echo "======================================="
                    echo "SUCCESS : ${successEnvs}"
                    echo "FAILED  : ${failedEnvs}"
                    echo "======================================="

                    if (failedEnvs.size() > 0) {

                        error("Deployment failed for environments: ${failedEnvs}")
                    }
                }
            }
        }
    }

    post {

        success {

            echo "Pipeline completed successfully."
        }

        failure {

            echo "Pipeline failed."
        }

        always {

            cleanWs()
        }
    }
}
