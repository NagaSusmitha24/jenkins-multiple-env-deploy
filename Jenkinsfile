pipeline {

    agent any

    tools {
        jdk 'JDK-17'
    }

    environment {

        ANYPOINT_ORG_ID      = credentials('anypoint-org-id')
        EXCHANGE_GROUP_ID    = credentials('anypoint-org-id')

        CONNECTED_APP_ID     = credentials('anypoint-client-id')
        CONNECTED_APP_SECRET = credentials('anypoint-client-secret')

        TARGET               = credentials('target')
        CH2_REGION           = credentials('region')
        CH2_REPLICAS         = credentials('replicas')
        CH2_VCORES           = credentials('vcore')
        RUNTIME_VERSION      = credentials('version')

        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'

        SANDBOX_ENV_ID       = credentials('SANDBOX_ENV_ID')
        DEV_ENV_ID           = credentials('DEV_ENV_ID')
        UAT_ENV_ID           = credentials('UAT_ENV_ID')
        PROD_ENV_ID          = credentials('PROD_ENV_ID')
    }

    parameters {

        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox,dev,uat,production',
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
                            appName : "multiple-environments-sandbox"
                        ],
                        "dev": [
                            envId   : env.DEV_ENV_ID,
                            appName : "multiple-environments-dev"
                        ],
                        "uat": [
                            envId   : env.UAT_ENV_ID,
                            appName : "multiple-environments-uat"
                        ],
                        "production": [
                            envId   : env.PROD_ENV_ID,
                            appName : "multiple-environments-prod"
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
