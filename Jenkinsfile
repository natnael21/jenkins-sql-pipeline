pipeline {
  agent any

  // 1. Parameters
  parameters {
    choice(name: 'CLIENT', choices: ['ddco', 'ddil', 'ddva', 'taic'], description: 'Client code (used in DB hostname)')
    string(name: 'DB_NAME', defaultValue: 'correspondence', description: 'Target Postgres database name')
    text(name: 'SQL_COMMAND', defaultValue: '', description: 'SQL to execute (SELECT, DELETE, UPDATE only)')
  }

  environment {
    // 2. Map branch to environment
    ENV_SUFFIX = """${env.BRANCH_NAME == 'dev' ? 'dev' : (env.BRANCH_NAME == 'stage' ? 'stage' : 'prod')}"""
    DB_HOST = "${params.CLIENT}.db.${ENV_SUFFIX}.corvesta.net"
    SQL_FILE = 'query.sql'
  }

  options {
    // Fail fast, keep logs, etc.
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timeout(time: 10, unit: 'MINUTES')
    ansiColor('xterm')
  }

  stages {
    stage('Validate SQL') {
      steps {
        script {
          // 3. Security: Only allow SELECT, DELETE, UPDATE; block DROP, ALTER, TRUNCATE, etc.
          def sql = params.SQL_COMMAND.trim()
          if (!sql) {
            error "SQL_COMMAND is empty!"
          }
          def forbidden = ~/(?i)\b(DROP|ALTER|TRUNCATE|CREATE|GRANT|REVOKE|INSERT|MERGE|REPLACE|COMMENT|SET|SHOW|VACUUM|ANALYZE|COPY|UNLOGGED|CLUSTER|DISCARD|EXPLAIN|LISTEN|NOTIFY|REFRESH|REINDEX|RESET|SECURITY|UNLISTEN|WITH)\b/
          if (sql =~ forbidden) {
            error "Forbidden SQL keyword detected! Only SELECT, DELETE, and UPDATE are allowed."
          }
          def allowed = ~/(?i)^\s*(SELECT|DELETE|UPDATE)\b/
          if (!(sql =~ allowed)) {
            error "SQL must start with SELECT, DELETE, or UPDATE."
          }
          // 4. Insert SELECT COUNT(*) safety check
          def countCheck = ""
          if (sql.toUpperCase().startsWith("DELETE") || sql.toUpperCase().startsWith("UPDATE")) {
            // Try to extract FROM ... or WHERE ... for count
            def fromIdx = sql.toUpperCase().indexOf("FROM")
            def whereIdx = sql.toUpperCase().indexOf("WHERE")
            if (fromIdx > -1) {
              def fromClause = sql.substring(fromIdx)
              countCheck = "SELECT COUNT(*) " + fromClause + ";"
            } else {
              error "Could not parse FROM clause for safety check."
            }
          }
          // 5. Write SQL to file with BEGIN/COMMIT and count check
          writeFile file: env.SQL_FILE, text: (
            "BEGIN;\n" +
            (countCheck ? "-- Safety check\n${countCheck}\n" : "") +
            "-- User SQL\n${sql}\n" +
            "COMMIT;\n"
          )
        }
      }
    }

    stage('Run SQL') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'pg-creds', usernameVariable: 'PGUSER', passwordVariable: 'PGPASSWORD')]) {
          sh '''
            export PGPASSWORD="$PGPASSWORD"
            echo "Connecting to: $DB_HOST, DB: $DB_NAME, User: $PGUSER"
            psql -v ON_ERROR_STOP=1 -U "$PGUSER" -h "$DB_HOST" -d "$DB_NAME" -f "$SQL_FILE"
          '''
        }
      }
    }
  }

  post {
    always {
      script {
        echo "==== Execution Summary ===="
        echo "Branch: ${env.BRANCH_NAME}"
        echo "Environment: ${env.ENV_SUFFIX}"
        echo "DB Host: ${env.DB_HOST}"
        echo "DB Name: ${params.DB_NAME}"
        echo "Executed SQL:\n${params.SQL_COMMAND}"
        echo "=========================="
      }
      archiveArtifacts artifacts: "${env.SQL_FILE}", onlyIfSuccessful: false
    }
    success {
      echo "✅ SQL executed successfully."
    }
    failure {
      echo "❌ SQL execution failed. See logs above."
    }
  }
} 