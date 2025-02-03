pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "https://sonarcloud.io"
        TRUFFLEHOG_PATH = "/usr/local/bin/trufflehog"
        JIRA_SITE = "https://derrickweil.atlassian.net"
        JIRA_PROJECT = "SEC" // Your Jira project key
    }

    stages {
        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'AWS_SECRET_ACCESS_KEY' 
                ]]) {
                    sh '''
                    echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/derrickSh43/autoScale'
            }
        }

        // Security Scans
        stage('Static Code Analysis (SAST)') {
            steps {
                script {
                    def scanStatus = sh(script: """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=derricksh43_autoScale \
                        -Dsonar.organization=derricksh43 \
                        -Dsonar.host.url=${SONARQUBE_URL} \
                        -Dsonar.login=${SONARQUBE_TOKEN}
                    """, returnStatus: true)
                    
                    if (scanStatus != 0) {
                        createJiraTicket("Static Code Analysis Failed", "SonarQube scan detected issues in your code.")
                        error("SonarQube found security vulnerabilities!")
                    }
                }
            }
        }



        stage('Dependency Scanning') {
            steps {
                script {
                    def scanStatus = sh(script: "snyk test --all-projects", returnStatus: true)
                    if (scanStatus != 0) {
                        createJiraTicket("Dependency Scan Failed", "Snyk found security vulnerabilities in dependencies.")
                        error("Snyk detected vulnerabilities!")
                    }
                }
            }
        }

        stage('Secret Scanning') {
            steps {
                script {
                    def scanStatus = sh(script: "${TRUFFLEHOG_PATH} --regex --entropy=True .", returnStatus: true)
                    if (scanStatus != 0) {
                        createJiraTicket("Secret Scan Failed", "TruffleHog detected sensitive information in the repository.")
                        error("TruffleHog found secrets in your code!")
                    }
                }
            }
        }

        stage('Terraform Security Scan (Checkov)') {
            steps {
                script {
                    def scanStatus = sh(script: "checkov -d .", returnStatus: true)
                    if (scanStatus != 0) {
                        createJiraTicket("Infrastructure Security Issue", "Checkov found security misconfigurations in Terraform.")
                        error("Checkov found security risks in Terraform code!")
                    }
                }
            }
        }

        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }

        stage('Post Deployment Security Scan (DAST)') {
            steps {
                script {
                    def scanStatus = sh(script: "zap-cli quick-scan --start --self-contained --api-key myapikey http://deployed-url.com", returnStatus: true)
                    if (scanStatus != 0) {
                        createJiraTicket("Dynamic Security Scan Failed", "OWASP ZAP found security vulnerabilities in the deployed environment.")
                        error("OWASP ZAP found security vulnerabilities!")
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Terraform deployment completed successfully!'
        }

        failure {
            echo 'Terraform deployment failed!'
        }
    }
}

// Function to Create a Jira Ticket
def createJiraTicket(String issueTitle, String issueDescription) {
    script {
        jiraNewIssue site: "${JIRA_SITE}",
                     projectKey: "${JIRA_PROJECT}",
                     issueType: "Bug",
                     summary: issueTitle,
                     description: issueDescription,
                     priority: "High"
    }
}