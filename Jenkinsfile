pipeline {
    agent any

    environment {
        APIC_SERVER = "https://small-1-mgmt-api-manager-cp4i.apps.ocp.prontefflabs.com"
        APIC_ORG    = "indusapi-np"
        APIC_USER   = "umesh"
        APIC_PASS   = "!n0r1t5@C"
        APIC_REALM  = "provider/default-idp-2"
        GIT_BRANCH  = "main"
        GITHUB_REPO = "your-org/your-repo"   // 🔹 change this
        GITHUB_TOKEN = credentials('github-token') // Jenkins secret
    }

    parameters {
        string(name: 'API_NAME',    defaultValue: 'loan-api', description: 'API Name')
        string(name: 'API_VERSION', defaultValue: '1.0.0',    description: 'API Version')
        string(name: 'PRODUCT_NAME',defaultValue: 'loan-product', description: 'Product Name')
        string(name: 'CATALOG_NAME',defaultValue: 'sb', description: 'Catalog Name')
    }

    stages {

        stage('GitHub API Query') {
            steps {
                script {
                    echo "🔎 Fetching latest commit from GitHub..."
                    def response = sh(
                        script: """curl -s -H "Authorization: token ${GITHUB_TOKEN}" \
                        https://api.github.com/repos/${GITHUB_REPO}/commits/${GIT_BRANCH}""",
                        returnStdout: true
                    ).trim()
                    echo "✅ GitHub Commit Info: ${response}"
                }
            }
        }

        stage('APIC Login') {
            steps {
                sh """
                    apic login --server ${APIC_SERVER} \
                        --username ${APIC_USER} \
                        --password ${APIC_PASS} \
                        --realm ${APIC_REALM}
                """
            }
        }

        stage('Validate API YAML') {
            steps {
                sh "apic validate apis/${params.API_NAME}_${params.API_VERSION}.yaml"
            }
        }

        stage('Check API Existence') {
            steps {
                script {
                    def apiExists = sh(
                        script: """apic draft-apis:get ${params.API_NAME}:${params.API_VERSION} \
                        --server ${APIC_SERVER} --org ${APIC_ORG} || true""",
                        returnStdout: true
                    ).trim()
                    env.API_EXISTS = apiExists.contains("${params.API_NAME}:${params.API_VERSION}") ? "true" : "false"
                    echo "API exists? ${env.API_EXISTS}"
                }
            }
        }

        stage('Process API') {
            steps {
                script {
                    if (env.API_EXISTS == "true") {
                        sh """
                            apic draft-apis:update --server ${APIC_SERVER} \
                                --org ${APIC_ORG} ${params.API_NAME}:${params.API_VERSION} \
                                apis/${params.API_NAME}_${params.API_VERSION}.yaml
                        """
                    } else {
                        sh """
                            apic draft-apis:create apis/${params.API_NAME}_${params.API_VERSION}.yaml \
                                --server ${APIC_SERVER} --org ${APIC_ORG}
                        """
                    }
                }
            }
        }

        stage('Fix Product Refs') {
            steps {
                sh "/var/lib/jenkins/scripts/fix-product-refs.sh ${WORKSPACE}/products ${WORKSPACE}/apis"
            }
        }

        stage('Check Product Existence') {
            steps {
                script {
                    def productExists = sh(
                        script: """apic draft-products:get ${params.PRODUCT_NAME}:${params.API_VERSION} \
                        --server ${APIC_SERVER} --org ${APIC_ORG} || true""",
                        returnStdout: true
                    ).trim()
                    env.PRODUCT_EXISTS = productExists.contains("${params.PRODUCT_NAME}:${params.API_VERSION}") ? "true" : "false"
                    echo "Product exists? ${env.PRODUCT_EXISTS}"
                }
            }
        }

        stage('Process Product') {
            steps {
                script {
                    if (env.PRODUCT_EXISTS == "true") {
                        sh """
                            apic draft-products:update --server ${APIC_SERVER} \
                                --org ${APIC_ORG} ${params.PRODUCT_NAME}:${params.API_VERSION} \
                                products/${params.PRODUCT_NAME}_${params.API_VERSION}.yaml
                        """
                    } else {
                        sh """
                            apic draft-products:create products/${params.PRODUCT_NAME}_${params.API_VERSION}.yaml \
                                --server ${APIC_SERVER} --org ${APIC_ORG}
                        """
                    }
                }
            }
        }

        stage('Publish Product') {
            steps {
                sh """
                    apic products:publish products/${params.PRODUCT_NAME}_${params.API_VERSION}.yaml \
                        --scope catalog --catalog ${params.CATALOG_NAME} \
                        --server ${APIC_SERVER} --org ${APIC_ORG}
                """
            }
        }

        stage('Backup & Commit') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y-%m-%d_%H-%M-%S", returnStdout: true).trim()
                    sh """
                        mkdir -p apis/${params.API_NAME}
                        mkdir -p products/${params.PRODUCT_NAME}

                        cp apis/${params.API_NAME}_${params.API_VERSION}.yaml apis/${params.API_NAME}/
                        cp products/${params.PRODUCT_NAME}_${params.API_VERSION}.yaml products/${params.PRODUCT_NAME}/

                        git config user.email "jenkins@yourorg.com"
                        git config user.name "Jenkins CI"
                        git add .
                        git commit -m "Backup ${params.API_NAME}:${params.API_VERSION} & ${params.PRODUCT_NAME}:${params.API_VERSION} at ${timestamp}" || true
                        git push origin ${GIT_BRANCH}
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
            echo "❌ Pipeline failed!"
        }
    }
}
