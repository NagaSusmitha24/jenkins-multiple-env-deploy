pipeline {
    agent any

    tools {
        jdk 'JDK-17'
    }

    environment {
        ANYPOINT_ORG_ID      = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'
        CONNECTED_APP_ID     = 'c09ce27b91784380bf8646a62ec65c9d'
        CONNECTED_APP_SECRET = '8b7B8D66306E47a29f4e3007547774f4'
        EXCHANGE_GROUP_ID    = '6c5ad96b-67a8-4bc6-8bb1-12443d5764e1'

        APP_NAME             = 'jenkins-app'
        CH2_REGION           = 'us-east-1'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.17'

        // Use the actual CloudHub 2.0 target from Runtime Manager
        TARGET               = 'Cloudhub-US-East-2'

        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'
    }

    parameters {

        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox,dev,uat,prod',
            description: 'Comma separated environments. Example: Sandbox,UAT,Production'
        )

        booleanParam(
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish artifact to Anypoint Exchange'
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
                echo "Building Mule application..."

                bat """
                    mvn clean test ^
                    -s %MAVEN_SETTINGS%
                """
            }

            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('Package') {
            steps {
                echo "Packaging Mule application..."

                bat """
                    mvn package ^
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

                echo "Publishing to Exchange..."

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

                    def environments = params.DEPLOY_ENVS
                            .split(',')
                            .collect { it.trim() }

                    def successEnvs = []
                    def failedEnvs  = []

                    echo "====================================="
                    echo "Environments Selected:"
                    echo "${environments}"
                    echo "====================================="

                    for (String envName : environments) {

                        def appName = "${APP_NAME}-${envName.toLowerCase()}"

                        echo ""
                        echo "====================================="
                        echo "Environment : ${envName}"
                        echo "Application : ${appName}"
                        echo "====================================="

                        def deployStatus = bat(
                            returnStatus: true,
                            script: """
                                mvn deploy ^
                                -DskipTests ^
                                -DmuleDeploy ^
                                -s %MAVEN_SETTINGS% ^
                                -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                                -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                                -Danypoint.environment=${envName} ^
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
                    echo "====================================="
                    echo "DEPLOYMENT SUMMARY"
                    echo "====================================="
                    echo "SUCCESS : ${successEnvs}"
                    echo "FAILED  : ${failedEnvs}"
                    echo "====================================="

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
