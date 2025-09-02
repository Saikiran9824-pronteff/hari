pipeline {
    agent any

    parameters {
        string(name: 'API_NAME', defaultValue: 'loan-api', description: 'API Name')
        string(name: 'API_VERSION', defaultValue: '1.0.0', description: 'API Version')
        string(name: 'PRODUCT_NAME', defaultValue: 'loan-product', description: 'Product Name')
        string(name: 'CATALOG_NAME', defaultValue: 'sandbox', description: 'Catalog Name')
    }

    environment {
        APIC_SERVER = "https://small-1-mgmt-api-manager-cp4i.apps.ocp.prontefflabs.com"
        APIC_USER   = "umesh"
        APIC_PASS   = "!n0r1t5@C"
        APIC_ORG    = "indusapi-np"
    }

    stages {
        stage('Login to API Connect') {
            steps {
                sh '''
                  echo "🔐 Logging in to API Connect..."
                  apic login --server $APIC_SERVER --username $APIC_USER --password $APIC_PASS --realm provider/default-idp-2
                '''
            }
        }

        stage('Validate API YAML') {
            steps {
                sh '''
                  echo "✅ Validating API YAML for ${API_NAME}_${API_VERSION}"
                  apic validate apis/${API_NAME}_${API_VERSION}.yaml
                '''
            }
        }

        stage('Process API (Create/Update)') {
            steps {
                script {
                    def apiCheck = sh(
                        script: "apic draft-apis:get ${API_NAME}:${API_VERSION} --server $APIC_SERVER --org $APIC_ORG || true",
                        returnStdout: true
                    ).trim()

                    if (apiCheck.contains("${API_NAME}:${API_VERSION}")) {
                        sh """
                          echo "♻️ Updating existing draft API ${API_NAME}:${API_VERSION}"
                          apic draft-apis:update --server $APIC_SERVER --org $APIC_ORG ${API_NAME}:${API_VERSION} apis/${API_NAME}_${API_VERSION}.yaml
                        """
                    } else {
                        sh """
                          echo "🆕 Creating new draft API ${API_NAME}:${API_VERSION}"
                          apic draft-apis:create apis/${API_NAME}_${API_VERSION}.yaml --server $APIC_SERVER --org $APIC_ORG
                        """
                    }
                }
            }
        }

        stage('Fix Product References') {
            steps {
                sh '''
                  echo "🔧 Fixing product references..."
                  /var/lib/jenkins/scripts/fix-product-refs.sh products apis
                '''
            }
        }

        stage('Process Product (Create/Update)') {
            steps {
                script {
                    def productCheck = sh(
                        script: "apic draft-products:get ${PRODUCT_NAME}:${API_VERSION} --server $APIC_SERVER --org $APIC_ORG || true",
                        returnStdout: true
                    ).trim()

                    if (productCheck.contains("${PRODUCT_NAME}:${API_VERSION}")) {
                        sh """
                          echo "♻️ Updating existing draft Product ${PRODUCT_NAME}:${API_VERSION}"
                          apic draft-products:update --server $APIC_SERVER --org $APIC_ORG ${PRODUCT_NAME}:${API_VERSION} products/${PRODUCT_NAME}_${API_VERSION}.yaml
                        """
                    } else {
                        sh """
                          echo "🆕 Creating new draft Product ${PRODUCT_NAME}:${API_VERSION}"
                          apic draft-products:create products/${PRODUCT_NAME}_${API_VERSION}.yaml --server $APIC_SERVER --org $APIC_ORG
                        """
                    }
                }
            }
        }

        stage('Publish Product') {
            steps {
                sh '''
                  echo "🚀 Publishing Product ${PRODUCT_NAME}_${API_VERSION}.yaml to catalog ${CATALOG_NAME}"
                  apic products:publish products/${PRODUCT_NAME}_${API_VERSION}.yaml --scope catalog --catalog ${CATALOG_NAME} --server $APIC_SERVER --org $APIC_ORG
                '''
            }
        }

        stage('Backup to Git') {
            steps {
                sh '''
                  echo "📦 Backing up API and Product YAML to Git..."
                  mkdir -p apis/${API_NAME} products/${PRODUCT_NAME}
                  cp apis/${API_NAME}_${API_VERSION}.yaml apis/${API_NAME}/
                  cp products/${PRODUCT_NAME}_${API_VERSION}.yaml products/${PRODUCT_NAME}/
                  
                  git add .
                  git commit -m "Backup ${API_NAME}:${API_VERSION} & ${PRODUCT_NAME}:${API_VERSION} at $(date +%Y-%m-%d_%H-%M-%S)" || true
                  git push origin main
                '''
            }
        }
    }

    post {
        success {
            echo "🎉 CI/CD pipeline completed successfully with backup!"
        }
        failure {
            echo "❌ Pipeline failed. Please check logs."
        }
    }
}
