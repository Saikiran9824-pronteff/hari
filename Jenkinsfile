pipeline {
    agent any

    parameters {
        string(name: 'API_NAME', defaultValue: '', description: 'API Name')
        string(name: 'API_VERSION', defaultValue: '', description: 'API Version')
        string(name: 'PRODUCT_NAME', defaultValue: '', description: 'Product Name')
        string(name: 'CATALOG_NAME', defaultValue: '', description: 'Catalog Name')
    }

    environment {
        APIC_SERVER = "https://small-1-mgmt-api-manager-cp4i.apps.ocp.prontefflabs.com"
        APIC_ORG    = "indusapi-np"
        APIC_USER   = "umesh"
        APIC_PASS   = "!n0r1t5@C"
        APIC_REALM  = "provider/default-idp-2"
    }

    stages {
        stage('Login to API Connect') {
            steps {
                sh """
                apic login --server $APIC_SERVER \
                           --username $APIC_USER \
                           --password $APIC_PASS \
                           --realm $APIC_REALM
                """
            }
        }

        stage('Validate API') {
            steps {
                sh """
                echo "✅ Validating API YAML for ${API_NAME}_${API_VERSION}"
                apic validate apis/${API_NAME}_${API_VERSION}.yaml
                """
            }
        }

        stage('Check API') {
            steps {
                script {
                    def apiExists = sh(
                        script: "apic draft-apis:get ${API_NAME}:${API_VERSION} --server $APIC_SERVER --org $APIC_ORG || true",
                        returnStdout: true
                    ).trim()
                    env.API_EXISTS = apiExists.contains("${API_NAME}:${API_VERSION}") ? "true" : "false"
                    echo "API exists? ${env.API_EXISTS}"
                }
            }
        }

        stage('Process API') {
            steps {
                script {
                    if (env.API_EXISTS == "true") {
                        echo "♻️ Updating existing draft API ${API_NAME}:${API_VERSION}"
                        sh """
                        apic draft-apis:update --server $APIC_SERVER --org $APIC_ORG \
                                               ${API_NAME}:${API_VERSION} \
                                               apis/${API_NAME}_${API_VERSION}.yaml
                        """
                    } else {
                        echo "🆕 Creating new draft API ${API_NAME}:${API_VERSION}"
                        sh """
                        apic draft-apis:create apis/${API_NAME}_${API_VERSION}.yaml \
                                               --server $APIC_SERVER --org $APIC_ORG
                        """
                    }
                }
            }
        }

        stage('Fix Product References') {
            steps {
                sh """
                echo "🔧 Fixing product references with script..."
                /var/lib/jenkins/scripts/fix-product-refs.sh \\
                    ${WORKSPACE}/products ${WORKSPACE}/apis
                """
            }
        }

        stage('Check Product') {
            steps {
                script {
                    def productExists = sh(
                        script: "apic draft-products:get ${PRODUCT_NAME}:${API_VERSION} --server $APIC_SERVER --org $APIC_ORG || true",
                        returnStdout: true
                    ).trim()
                    env.PRODUCT_EXISTS = productExists.contains("${PRODUCT_NAME}:${API_VERSION}") ? "true" : "false"
                    echo "Product exists? ${env.PRODUCT_EXISTS}"
                }
            }
        }

        stage('Process Product') {
            steps {
                script {
                    if (env.PRODUCT_EXISTS == "true") {
                        echo "♻️ Updating existing draft Product ${PRODUCT_NAME}:${API_VERSION}"
                        sh """
                        apic draft-products:update --server $APIC_SERVER --org $APIC_ORG \
                                                   ${PRODUCT_NAME}:${API_VERSION} \
                                                   products/${PRODUCT_NAME}_${API_VERSION}.yaml
                        """
                    } else {
                        echo "🆕 Creating new draft Product ${PRODUCT_NAME}:${API_VERSION}"
                        sh """
                        apic draft-products:create products/${PRODUCT_NAME}_${API_VERSION}.yaml \
                                                   --server $APIC_SERVER --org $APIC_ORG
                        """
                    }
                }
            }
        }

        stage('Publish Product') {
            steps {
                sh """
                echo "🚀 Publishing Product ${PRODUCT_NAME}_${API_VERSION}.yaml to catalog ${CATALOG_NAME}"
                apic products:publish products/${PRODUCT_NAME}_${API_VERSION}.yaml \
                                      --scope catalog \
                                      --catalog ${CATALOG_NAME} \
                                      --server $APIC_SERVER \
                                      --org $APIC_ORG
                """
            }
        }

        stage('Backup & Git Commit') {
            steps {
                script {
                    def timestamp = sh(
                        script: "date +%Y-%m-%d_%H-%M-%S",
                        returnStdout: true
                    ).trim()

                    sh """
                    mkdir -p apis/${API_NAME} products/${PRODUCT_NAME}
                    cp apis/${API_NAME}_${API_VERSION}.yaml apis/${API_NAME}/
                    cp products/${PRODUCT_NAME}_${API_VERSION}.yaml products/${PRODUCT_NAME}/

                    git add .
                    git commit -m "Backup ${API_NAME}:${API_VERSION} & ${PRODUCT_NAME}:${API_VERSION} at ${timestamp}" || echo "No changes to commit"
                    git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 CI/CD pipeline completed successfully with backup!"
        }
        failure {
            echo "❌ CI/CD pipeline failed!"
        }
    }
}
