@Library('github-api-lib') _

import groovy.json.JsonSlurper

// ---------- PARAMETERS (Dynamic Choices from GitHub) ----------
properties([
  parameters([
    choice(
      name: 'API_NAME',
      choices: getFilesFromGitHub("apis"),
      description: 'Select API YAML from GitHub repo'
    ),
    choice(
      name: 'PRODUCT_NAME',
      choices: getFilesFromGitHub("products"),
      description: 'Select Product YAML from GitHub repo'
    )
  ])
])

// ---------- PIPELINE START ----------
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Saikiran9824-pronteff/hari.git'
            }
        }

        stage('Validate API') {
            steps {
                script {
                    echo "Validating API YAML: ${params.API_NAME}.yaml"
                    sh """
                      apic validate apis/${params.API_NAME}.yaml
                    """
                }
            }
        }

        stage('Validate Product') {
            steps {
                script {
                    echo "Validating Product YAML: ${params.PRODUCT_NAME}.yaml"
                    sh """
                      apic validate products/${params.PRODUCT_NAME}.yaml
                    """
                }
            }
        }
    }
}

// ---------- HELPER FUNCTION ----------
def getFilesFromGitHub(folder) {
    def repo = "Saikiran9824-pronteff/hari"
    def url = "https://api.github.com/repos/${repo}/contents/${folder}"

    def connection = new URL(url).openConnection()
    connection.setRequestProperty("Accept", "application/vnd.github.v3+json")

    def json = new JsonSlurper().parse(connection.inputStream)

    return json.findAll { it.name.endsWith(".yaml") }
               .collect { it.name.replace(".yaml", "") }
               .join("\n")   // Jenkins requires newline-separated list
}
