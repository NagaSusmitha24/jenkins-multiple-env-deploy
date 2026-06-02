pipeline {

    agent any

    tools {
        jdk 'JDK-17'
    }

    environment {

        ANYPOINT_ORG_ID      = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'
        EXCHANGE_GROUP_ID    = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'

        CONNECTED_APP_ID     = 'c09ce27b91784380bf8646a62ec65c9d'
        CONNECTED_APP_SECRET = '8b7B8D66306E47a29f4e3007547774f4'

        TARGET               = 'Cloudhub-US-East-2'
        CH2_REGION           = 'us-east-1'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.17'

        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'

        SANDBOX_ENV_ID       = '39886fb6-d89f-4a56-a400-15ce44a87bbc'
        DEV_ENV_ID           = '2ce5a5ae-ce2a-41a0-96fc-68a5e22df760'
        UAT_ENV_ID           = '864fd6b3-5b44-41c5-89e9-d76573a27639'
        PROD_ENV_ID          = 'ac61ddf1-1c0f-4c0f-9dfe-f305f90dbb8a'
    }

    parameters {

        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox,dev,uat,prod',
            description: 'Comma separated environments'
        )

        booleanParam(
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish artifact to Exchange'
        )

        booleanParam(
            name: 'DEPLOY_TO_CLOUDHUB',
            defaultValue: true,
            description: 'Deploy artifact to CloudHub 2.0'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
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
                bat """
                mvn clean package ^
                -DskipTests ^
                -s %MAVEN_SETTINGS%
                """
            }
        }

        stage('Publish To Exchange') {

            when {
                expression { return params.PUBLISH_TO_EXCHANGE }
            }

            steps {

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

        stage('Deploy To CloudHub2') {

            when {
                expression { return params.DEPLOY_TO_CLOUDHUB }
            }

            steps {

                script {

                    def envConfig = [
                        "Sandbox": [
                            envId   : env.SANDBOX_ENV_ID,
                            appName : "world-project-api-jen-sandbox"
                        ],
                        "dev": [
                            envId   : env.DEV_ENV_ID,
                            appName : "world-project-api-jen-dev"
                        ],
                        "uat": [
                            envId   : env.UAT_ENV_ID,
                            appName : "world-project-api-jen-uat"
                        ],
                        "prod": [
                            envId   : env.PROD_ENV_ID,
                            appName : "world-project-api-jen-prod"
                        ]
                    ]

                    def successEnvs = []
                    def failedEnvs = []

                    def environments = params.DEPLOY_ENVS
                        .split(',')
                        .collect { it.trim() }

                    environments.each { envName ->

                        if (!envConfig.containsKey(envName)) {

                            echo "Invalid Environment: ${envName}"
                            failedEnvs.add(envName)
                            return
                        }

                        def envId = envConfig[envName].envId
                        def appName = envConfig[envName].appName

                        echo "================================="
                        echo "Environment : ${envName}"
                        echo "Environment ID : ${envId}"
                        echo "Application : ${appName}"
                        echo "================================="

                        def status = bat(
                            returnStatus: true,
                            script: """
                            mvn deploy ^
                            -DmuleDeploy ^
                            -DskipTests ^
                            -s %MAVEN_SETTINGS% ^
                            -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                            -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                            -Danypoint.platform.environment=${envName} ^
                            -Dcloudhub2.applicationName=${appName} ^
                            -Dcloudhub2.target=%TARGET% ^
                            -Dcloudhub2.region=%CH2_REGION% ^
                            -Dcloudhub2.replicas=%CH2_REPLICAS% ^
                            -Dcloudhub2.vCores=%CH2_VCORES% ^
                            -Dcloudhub2.muleVersion=%RUNTIME_VERSION% ^
                            -Danypoint.businessGroup=%ANYPOINT_ORG_ID%
                            """
                        )

                        if (status == 0) {
                            successEnvs.add(envName)
                        } else {
                            failedEnvs.add(envName)
                        }
                    }

                    echo "SUCCESS : ${successEnvs}"
                    echo "FAILED  : ${failedEnvs}"

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
