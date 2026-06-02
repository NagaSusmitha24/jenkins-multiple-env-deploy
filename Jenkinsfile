pipeline {
    agent any

    tools {
        jdk 'JDK-17'  // Must match Jenkins Global Tool Configuration
    }

    environment {
        ANYPOINT_USERNAME = 'Durga-May'   // or use Jenkins credentials plugin
        ANYPOINT_PASSWORD = 'Durga@53'    // secure this in Jenkins credentials
        BUSINESS_GROUP_ID = '0c18259d-596e-4342-88f4-05dc50278018'
        CLOUDHUB_ENV      = 'Sandbox'
        CLOUDHUB_TARGET   = 'Cloudhub-US-East-2'
        MULE_VERSION      = '4.9.17'
        APP_NAME          = 'susmitha-api'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Deploy to CloudHub 2.0') {
            steps {
                bat """
                    mvn deploy -DmuleDeploy \
                        -Danypoint.username=${ANYPOINT_USERNAME} \
                        -Danypoint.password=${ANYPOINT_PASSWORD} \
                        -DbusinessGroup=${BUSINESS_GROUP_ID} \
                        -Denv=${CLOUDHUB_ENV} \
                        -Dtarget=${CLOUDHUB_TARGET} \
                        -DmuleVersion=${MULE_VERSION} \
                        -DapplicationName=${APP_NAME} \
                        -DskipTests \
                        -s C:\\Users\\Admin\\.m2\\settings.xml
                """
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed — check the logs!'
        }
        always {
            cleanWs()
        }
    }
}
