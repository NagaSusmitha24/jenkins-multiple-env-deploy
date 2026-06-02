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
        APP_NAME             = 'jenkins-multiple-env'
        CH2_REGION           = 'us-east-1'
        CH2_REPLICAS         = '1'
        CH2_VCORES           = '0.1'
        RUNTIME_VERSION      = '4.9.0'
        TARGET               = 'cloudhub-2.0'
        MAVEN_SETTINGS       = 'C:\\Users\\Admin\\.m2\\settings.xml'
    }
 
    parameters {
        extendedChoice(
            name: 'DEPLOY_ENVS',
            type: 'PT_CHECKBOX',
            value: 'Sandbox,Dev,QA,UAT,Prod',
            defaultValue: 'Sandbox',
            description: 'Select one or more environments to deploy to'
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
 
        stage('Build and Test') {
            steps {
                echo ">>> Building and running MUnit tests..."
                bat "mvn clean test -s %MAVEN_SETTINGS%"
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
                echo ">>> Packaging MuleSoft application..."
                bat "mvn package -DskipTests -s %MAVEN_SETTINGS%"
            }
        }
 
        stage('Publish to Exchange') {
            when {
                expression { return params.PUBLISH_TO_EXCHANGE }
            }
            steps {
                echo ">>> Publishing artifact to Anypoint Exchange..."
                bat """
                    mvn deploy -DskipTests ^
                    -s %MAVEN_SETTINGS% ^
                    -Danypoint.platform.client_id=%CONNECTED_APP_ID% ^
                    -Danypoint.platform.client_secret=%CONNECTED_APP_SECRET% ^
                    -DrepositoryId=anypoint-exchange-v3 ^
                    -Durl=https://maven.anypoint.mulesoft.com/api/v3/organizations/%EXCHANGE_GROUP_ID%/maven
                """
            }
        }
 
        stage('Deploy to CloudHub 2.0') {
            when {
                expression { return params.DEPLOY_TO_CLOUDHUB }
            }
            steps {
                script {
                    def selectedEnvs = params.DEPLOY_ENVS.split(',')
 
                    for (def envName in selectedEnvs) {
                        def envTrimmed  = envName.trim()           // e.g. "Sandbox"
                        def appSuffix   = envTrimmed.toLowerCase() // e.g. "sandbox"
 
                        echo "=========================================="
                        echo ">>> Deploying to environment: ${envTrimmed}"
                        echo "=========================================="
 
                        // ── KEY FIX ────────────────────────────────────────────────────
                        // Use ${groovyVar} for Groovy variables (envTrimmed, appSuffix)
                        // Use %WIN_VAR%    for Jenkins environment variables (MAVEN_SETTINGS etc.)
                        // Both work together inside the same bat """ block
                        // ──────────────────────────────────────────────────────────────
                        bat """
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
                            -Danypoint.environment=%envTrimmed%
                        """
 
                        echo ">>> Successfully deployed to: %envTrimmed%"
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
