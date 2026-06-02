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
        APP_NAME             = 'jenkins-multiple-envs'
        CH2_REGION           = 'us-east-1'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.0'
        TARGET               = 'cloudhub-2.0'
        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'
    }

    parameters {
        // ─────────────────────────────────────────────────────────────────
        // TYPE EXACTLY as your Anypoint Platform environment names
        // Single  → Sandbox
        // Multiple → Sandbox,Dev,QA,Prod
        // ─────────────────────────────────────────────────────────────────
        string(
            name: 'DEPLOY_ENVS',
            defaultValue: 'Sandbox',
            description: '''Enter Anypoint environment name(s).
Single:   Sandbox
Multiple: Sandbox,dev,uat,prod
NOTE: Must exactly match names in Anypoint Platform > Access Management > Environments'''
        )
        booleanParam(
            name: 'PUBLISH_TO_EXCHANGE',
            defaultValue: true,
            description: 'Publish artifact to Anypoint Exchange?'
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

        // ── Print exactly which environments will be deployed ──────────────
        stage('Show Deploy Plan') {
            steps {
                script {
                    if (!params.DEPLOY_ENVS?.trim()) {
                        error "DEPLOY_ENVS is empty! Please enter at least one environment."
                    }
                    def envList = params.DEPLOY_ENVS.split(',').collect { it.trim() }
                    echo "=========================================="
                    echo "DEPLOY PLAN"
                    echo "=========================================="
                    envList.eachWithIndex { env, i ->
                        echo "  ${i + 1}. ${env}  →  ${APP_NAME}-${env.toLowerCase()}"
                    }
                    echo "=========================================="
                }
            }
        }

        stage('Build and Test') {
            steps {
                script {
                    echo ">>> Building and running MUnit tests..."
                    def exitCode = bat(
                        script: "mvn clean test -s %MAVEN_SETTINGS%",
                        returnStatus: true
                    )
                    if (exitCode != 0) {
                        error "Build & Test FAILED! Exit code: ${exitCode}"
                    }
                    echo ">>> Build & Test PASSED!"
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
                    if (exitCode != 0) {
                        error "Packaging FAILED! Exit code: ${exitCode}"
                    }
                    echo ">>> Packaging SUCCESSFUL!"
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
                    if (exitCode != 0) {
                        error "Exchange Publish FAILED! Exit code: ${exitCode}"
                    }
                    echo ">>> Exchange Publish SUCCESSFUL!"
                }
            }
        }

        // ── Deploy to each environment one by one ──────────────────────────
        stage('Deploy to CloudHub 2.0') {
            when {
                expression { return params.DEPLOY_TO_CLOUDHUB }
            }
            steps {
                script {
                    def selectedEnvs = params.DEPLOY_ENVS.split(',').collect { it.trim() }
                    def successEnvs  = []
                    def failedEnvs   = []

                    echo ">>> Starting deployment to ${selectedEnvs.size()} environment(s)..."

                    for (def envTrimmed in selectedEnvs) {
                        def appSuffix = envTrimmed.toLowerCase()

                        echo "=========================================="
                        echo ">>> [${selectedEnvs.indexOf(envTrimmed)+1}/${selectedEnvs.size()}] Deploying to: ${envTrimmed}"
                        echo ">>> App Name : ${APP_NAME}-${appSuffix}"
                        echo ">>> Env Name : ${envTrimmed}"
                        echo "=========================================="

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
                            returnStatus: true
                        )

                        if (exitCode == 0) {
                            successEnvs.add(envTrimmed)
                            echo ">>> SUCCESS: ${envTrimmed}"
                        } else {
                            failedEnvs.add(envTrimmed)
                            echo ">>> FAILED : ${envTrimmed} (exit code: ${exitCode})"
                        }
                    }

                    // ── Final Summary ──────────────────────────────────────
                    echo "=========================================="
                    echo "DEPLOYMENT SUMMARY"
                    echo "=========================================="
                    echo "SUCCESS (${successEnvs.size()}): ${successEnvs}"
                    echo "FAILED  (${failedEnvs.size()}): ${failedEnvs}"
                    echo "=========================================="

                    if (failedEnvs.size() > 0) {
                        error "Deployment failed for: ${failedEnvs}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "ALL DEPLOYMENTS SUCCESSFUL - Environments: ${params.DEPLOY_ENVS}"
        }
        failure {
            echo "PIPELINE FAILED - Check logs above for details"
        }
        always {
            cleanWs()
        }
    }
}
