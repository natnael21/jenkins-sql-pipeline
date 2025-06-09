
pipeline {
    agent any

    // Environment variables
    environment {
        // Vault path for database credentials
        VAULT_PATH = 'kv/data/postgres/rw-user'
        // Will store credentials fetched from vault
        DB_USERNAME = ''
        DB_PASSWORD = ''
    }

    // Parameters for the build
    parameters {
        choice(
            name: 'CLIENT',
            choices: ['ddco', 'ddva', 'ddil', 'taic'],
            description: 'Select the client database to target'
        )
        string(
            name: 'DB_NAME',
            defaultValue: 'correspondence',
            description: 'Database name to connect to'
        )
        text(
            name: 'SQL_COMMAND',
            description: '''SQL command to execute. 
            Privileges are controlled by the rw_users role.
            Common operations: SELECT, INSERT, UPDATE, DELETE'''
        )
    }

    options {
        // Timeout after 30 minutes
        timeout(time: 30, unit: 'MINUTES')
        // Keep only last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Don't run concurrent builds
        disableConcurrentBuilds()
    }

    stages {
        stage('Fetch Credentials') {
            steps {
                script {
                    // Using withVault to securely fetch credentials
                    withVault(
                        configuration: [
                            timeout: 60,
                            vaultCredentialId: 'vault-approle',
                            vaultUrl: 'https://vault.corvesta.net'
                        ],
                        vaultSecrets: [
                            [
                                path: env.VAULT_PATH,
                                secretValues: [
                                    [vaultKey: 'username', envVar: 'DB_USERNAME'],
                                    [vaultKey: 'password', envVar: 'DB_PASSWORD']
                                ]
                            ]
                        ]
                    ) {
                        // Store credentials in environment variables
                        env.DB_USERNAME = env.DB_USERNAME
                        env.DB_PASSWORD = env.DB_PASSWORD
                    }

                    if (!env.DB_USERNAME?.trim() || !env.DB_PASSWORD?.trim()) {
                        error "Failed to fetch database credentials from Vault"
                    }
                }
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    // Ensure SQL command is not empty
                    if (!params.SQL_COMMAND?.trim()) {
                        error "SQL command cannot be empty"
                    }

                    // Note: We're not explicitly blocking DROP/ALTER/etc anymore since 
                    // privileges are controlled by the rw_users role
                    
                    // Log a warning for potentially dangerous operations
                    def warningCommands = ['DROP', 'ALTER', 'TRUNCATE']
                    def upperSQL = params.SQL_COMMAND.toUpperCase()
                    warningCommands.each { command ->
                        if (upperSQL.contains(command)) {
                            echo "WARNING: Command contains ${command}. Execution will depend on rw_users role privileges."
                        }
                    }

                    // Determine environment based on branch
                    env.DB_ENV = getBranchEnvironment()
                    env.DB_HOST = "${params.CLIENT}.db.${env.DB_ENV}.corvesta.net"

                    echo """
                    ===== Execution Parameters =====
                    Client: ${params.CLIENT}
                    Environment: ${env.DB_ENV}
                    Database: ${params.DB_NAME}
                    Target Host: ${env.DB_HOST}
                    Using Role: rw_users
                    SQL Preview: ${params.SQL_COMMAND.take(100)}...
                    ==============================
                    """
                }
            }
        }

        stage('Production Approval') {
            when {
                expression { env.DB_ENV == 'prod' }
            }
            steps {
                // Required approval for production deployments
                timeout(time: 24, unit: 'HOURS') {
                    input message: 'Approve production database changes?',
                          submitter: 'devops-admins'
                }
            }
        }

        stage('Prepare SQL') {
            steps {
                script {
                    // Create wrapped SQL query
                    def wrappedSQL = """\\set AUTOCOMMIT OFF
\\conninfo
BEGIN;
-- Set role to ensure proper privileges
SET ROLE rw_users;
-- Verify role
SELECT CURRENT_USER, CURRENT_ROLE;
-- Preview affected rows
SELECT COUNT(*) FROM (
${params.SQL_COMMAND}
) AS count_check;
${params.SQL_COMMAND};
COMMIT;
"""
                    // Write to file
                    writeFile file: 'query.sql', text: wrappedSQL
                }
            }
        }

        stage('Execute SQL') {
            steps {
                script {
                    // Execute the SQL using psql with credentials from vault
                    def result = sh(
                        script: """
                            PGPASSWORD="\${DB_PASSWORD}" psql \
                            -U "\${DB_USERNAME}" \
                            -h "${env.DB_HOST}" \
                            -d "${params.DB_NAME}" \
                            -f query.sql \
                            -v ON_ERROR_STOP=1
                        """,
                        returnStatus: true
                    )

                    if (result != 0) {
                        error "SQL execution failed with exit code ${result}"
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                echo """
                ===== Script Execution Successful =====
                Please remember to:
                1. Update the Jira ticket with execution results
                2. Log this change in the Confluence hotfix log
                3. Include the role (rw_users) used for execution in documentation
                =====================================
                """
                
                // Uncomment for Slack notifications
                /*
                slackSend(
                    color: 'good',
                    message: "Database script executed successfully on ${env.DB_HOST} using rw_users role"
                )
                */
            }
        }
        failure {
            script {
                echo """
                Script execution failed. Please check:
                1. SQL syntax
                2. Role permissions (using rw_users)
                3. Database connection details
                4. Vault credentials access
                See logs above for specific error messages.
                """
                
                // Uncomment for Slack notifications
                /*
                slackSend(
                    color: 'danger',
                    message: "Database script execution failed on ${env.DB_HOST}"
                )
                */
            }
        }
        always {
            // Clean up sensitive files
            sh 'rm -f query.sql'
        }
    }
}

// Helper function to determine environment from branch name
def getBranchEnvironment() {
    switch(env.BRANCH_NAME) {
        case 'main':
        case 'prod':
            return 'prod'
        case 'stage':
            return 'stage'
        case 'dev':
            return 'dev'
        default:
            error "Unsupported branch: ${env.BRANCH_NAME}"
    }
}
