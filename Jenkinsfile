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
        APP_NAME             = 'jenkins-multiple-environs'
        CH2_REGION           = 'us-east-2'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.0'
        TARGET               = 'cloudhub-2.0'
        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'

        // ── Define ALL your environments here as a variable ───────────────
        // Change this one line to control which environments get deployed
        // Single  : ALL_ENVS = 'dev'
        // Multiple: ALL_ENVS = 'dev,uat,prod'
        ALL_ENVS             = 'dev,Sandbox,uat,prod'
    }

    parameters {
        // ── Override ALL_ENVS at runtime if needed ─────────────────────────
        // Leave blank → uses ALL_ENVS variable above automatically
        // Type value  → overrides ALL_ENVS for this specific build only
        string(
            name: 'DEPLOY_ENVS',
            defaultValue: '',
            description: '''Optional: Override environments for this build only.
Leave BLANK to use default ALL_ENVS = dev,uat,prod (defined in pipeline).
Or type specific envs → Example: dev   or   dev,uat'''
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

        stage('Show Deploy Plan') {
            steps {
                script {
                    // ── If DEPLOY_ENVS param is blank → use ALL_ENVS variable
                    // ── If DEPLOY_ENVS param has value → use that instead
                    def targetEnvs = params.DEPLOY_ENVS?.trim()
                                        ? params.DEPLOY_ENVS.trim()
                                        : env.ALL_ENVS

                    // Store resolved value for use in later stages
                    env.RESOLVED_ENVS = targetEnvs

                    def envList = targetEnvs.split(',').collect { it.trim() }

                    echo "=========================================="
                    echo "DEPLOY PLAN"
                    echo "Source : ${params.DEPLOY_ENVS?.trim() ? 'Parameter (manual override)' : 'ALL_ENVS variable (default)'}"
                    echo "Environments: ${targetEnvs}"
                    echo "=========================================="
                    envList.eachWithIndex { envItem, i ->
                        echo "  ${i + 1}. Environment : ${envItem}"
                        echo "     App Name   : ${APP_NAME}-${envItem.toLowerCase()}"
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

        stage('Deploy to CloudHub 2.0') {
            when {
                expression { return params.DEPLOY_TO_CLOUDHUB }
            }
            steps {
                script {
                    // Use RESOLVED_ENVS set in Show Deploy Plan stage
                    def selectedEnvs = env.RESOLVED_ENVS.split(',').collect { it.trim() }
                    def successEnvs  = []
                    def failedEnvs   = []

                    echo ">>> Deploying to ${selectedEnvs.size()} environment(s): ${selectedEnvs}"

                    for (def envTrimmed in selectedEnvs) {
                        def appSuffix   = envTrimmed.toLowerCase()
                        def fullAppName = "${APP_NAME}-${appSuffix}"

                        echo "=========================================="
                        echo ">>> [${selectedEnvs.indexOf(envTrimmed)+1}/${selectedEnvs.size()}]"
                        echo ">>> Environment : ${envTrimmed}"
                        echo ">>> App Name    : ${fullAppName}"
                        echo "=========================================="

                        def exitCode = bat(
                            script: """
                                mvn deploy -DskipTests ^
                                -s %MAVEN_SETTINGS% ^
                                -DmuleDeploy ^
                                -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                                -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                                -Danypoint.environment=${envTrimmed} ^
                                -Dcloudhub2.applicationName=${fullAppName} ^
                                -Dcloudhub2.target=%TARGET% ^
                                -Dcloudhub2.region=%CH2_REGION% ^
                                -Dcloudhub2.replicas=%CH2_REPLICAS% ^
                                -Dcloudhub2.vCores=%CH2_VCORES% ^
                                -Dcloudhub2.muleVersion=%RUNTIME_VERSION% ^
                                -Danypoint.businessGroup=%ANYPOINT_ORG_ID%
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

                    echo "=========================================="
                    echo "DEPLOYMENT SUMMARY"
                    echo "=========================================="
                    echo "SUCCESS (${successEnvs.size()}): ${successEnvs}"
                    echo "FAILED  (${failedEnvs.size()}): ${failedEnvs}"
                    echo "=========================================="

                    if (failedEnvs.size() > 0) {
                        error "Deployment failed for environments: ${failedEnvs}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "ALL DEPLOYMENTS SUCCESSFUL - Environments: ${env.RESOLVED_ENVS}"
        }
        failure {
            echo "PIPELINE FAILED - Check logs above for details"
        }
        always {
            cleanWs()
        }
    }
}
