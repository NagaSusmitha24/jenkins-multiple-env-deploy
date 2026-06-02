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
        RUNTIME_VERSION      = '4.9.0'
        TARGET               = 'cloudhub-2.0'
        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'
    }

    parameters {
        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox',
            description: 'Sandbox,dev,uat,prod'
        )
        booleanParam(
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish artifact to Anypoint Exchange first?'
        )
        booleanParam(
            name: 'DEPLOY_TO_CLOUDHUB',
            defaultValue: true,
            description: 'Deploy to CloudHub 2.0?'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                echo ">>> Checking out source code..."
                checkout scm
            }
        }

        stage('Validate Environments') {
            steps {
                script {
                    if (!params.DEPLOY_ENVS?.trim()) {
                        error "DEPLOY_ENVS is empty! Provide at least one environment."
                    }
                    echo ">>> Environments: ${params.DEPLOY_ENVS}"
                }
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    echo ">>> Building and running MUnit tests..."

                    // returnStatus: true → captures exit code instead of throwing exception
                    def exitCode = bat(
                        script: "mvn clean test -s %MAVEN_SETTINGS%",
                        returnStatus: true   // ← this is how you use $? equivalent in Jenkins
                    )

                    echo ">>> Build exit code: ${exitCode}"  // same as echo $? in bash

                    if (exitCode != 0) {
                        error "Build & Test FAILED! Exit code: ${exitCode}"
                    } else {
                        echo ">>> Build & Test PASSED!"
                    }
                }
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
                script {
                    echo ">>> Packaging MuleSoft application..."

                    def exitCode = bat(
                        script: "mvn package -DskipTests -s %MAVEN_SETTINGS%",
                        returnStatus: true
                    )

                    echo ">>> Package exit code: ${exitCode}"

                    if (exitCode != 0) {
                        error "Packaging FAILED! Exit code: ${exitCode}"
                    }
                }
            }
        }

        stage('Publish to Exchange') {
            when {
                expression { return params.PUBLISH_TO_EXCHANGE }
            }
            steps {
                script {
                    echo ">>> Publishing to Anypoint Exchange..."

                    def exitCode = bat(
                        script: """
                            mvn deploy -DskipTests ^
                            -s %MAVEN_SETTINGS% ^
                            -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                            -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                            -DrepositoryId=anypoint-exchange-v3 ^
                            -Durl=https://maven.anypoint.mulesoft.com/api/v3/organizations/%EXCHANGE_GROUP_ID%/maven
                        """,
                        returnStatus: true
                    )

                    echo ">>> Exchange publish exit code: ${exitCode}"

                    if (exitCode != 0) {
                        error "Exchange publish FAILED! Exit code: ${exitCode}"
                    }

                    echo ">>> Exchange publish SUCCESSFUL!"
                }
            }
        }

        stage('Deploy to CloudHub 2.0') {
            when {
                expression { return params.DEPLOY_TO_CLOUDHUB }
            }
            steps {
                script {
                    def selectedEnvs = params.DEPLOY_ENVS.split(',').collect { it.trim() }
                    def failedEnvs   = []   // track which envs failed
                    def successEnvs  = []   // track which envs succeeded

                    echo ">>> Total environments to deploy: ${selectedEnvs.size()}"

                    for (def envTrimmed in selectedEnvs) {
                        def appSuffix = envTrimmed.toLowerCase()

                        echo "=========================================="
                        echo ">>> Deploying to: ${envTrimmed}"
                        echo "=========================================="

                        // returnStatus captures the exit code ($? equivalent)
                        def exitCode = bat(
                            script: """
                                mvn deploy -DskipTests ^
                                -s %MAVEN_SETTINGS% ^
                                -DmuleDeploy ^
                                -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                                -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                                -Dcloudhub2.applicationName=%APP_NAME%-${appSuffix} ^
                                -Dcloudhub2.target=%TARGET% ^
                                -Dcloudhub2.region=%CH2_REGION% ^
                                -Dcloudhub2.replicas=%CH2_REPLICAS% ^
                                -Dcloudhub2.vCores=%CH2_VCORES% ^
                                -Dcloudhub2.muleVersion=%RUNTIME_VERSION% ^
                                -Danypoint.businessGroup=%ANYPOINT_ORG_ID% ^
                                -Danypoint.environment=${envTrimmed}
                            """,
                            returnStatus: true   // ← $? equivalent — captures 0 (success) or 1 (fail)
                        )

                        echo ">>> ${envTrimmed} deploy exit code: ${exitCode}"

                        // Check exit code and track result
                        if (exitCode == 0) {
                            successEnvs.add(envTrimmed)
                            echo ">>> SUCCESS: ${envTrimmed}"
                        } else {
                            failedEnvs.add(envTrimmed)
                            echo ">>> FAILED: ${envTrimmed} (exit code: ${exitCode})"
                            // Continue to next env instead of stopping entire pipeline
                        }
                    }

                    // Final summary
                    echo "=========================================="
                    echo ">>> DEPLOYMENT SUMMARY"
                    echo ">>> SUCCESS: ${successEnvs}"
                    echo ">>> FAILED : ${failedEnvs}"
                    echo "=========================================="

                    // Fail pipeline if any environment failed
                    if (failedEnvs.size() > 0) {
                        error "Deployment failed for environments: ${failedEnvs}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "SUCCESS - ${APP_NAME} deployed to: ${params.DEPLOY_ENVS}"
        }
        failure {
            echo "FAILED - Check console output above for errors"
        }
        always {
            cleanWs()
        }
    }
}
