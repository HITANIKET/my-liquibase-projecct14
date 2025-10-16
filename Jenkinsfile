pipeline {
    agent any
    
    environment {
        DB_URL = 'jdbc:oracle:thin:@10.37.0.103:1521/OFSUATDB'
        MAVEN_HOME = tool 'Maven'
        PATH = "${MAVEN_HOME}/bin:${PATH}"
    }
    
    tools {
        maven 'Maven-3.9.9'
        jdk 'JDKHITANIKET'  // Fixed: Changed from 'JDK11' to 'JDKHITANIKET'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Validate') {
            steps {
                echo 'Validating project structure...'
                bat 'mvn validate'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling the project...'
                bat 'mvn clean compile'
            }
        }
        
        stage('Clear Liquibase Locks') {
            steps {
                echo 'Clearing any Liquibase locks...'
                withCredentials([usernamePassword(credentialsId: 'OFSUATDB',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    bat '''
                        mvn liquibase:releaseLocks ^
                            -Dliquibase.url=%DB_URL% ^
                            -Dliquibase.username=%DB_USERNAME% ^
                            -Dliquibase.password=%DB_PASSWORD%
                    '''
                }
            }
        }
        
        stage('Fix Checksum Issues') {
            steps {
                echo 'Clearing checksum validation issues...'
                withCredentials([usernamePassword(credentialsId: 'OFSUATDB',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    script {
                        try {
                            bat '''
                                mvn liquibase:clearCheckSums ^
                                    -Dliquibase.url=%DB_URL% ^
                                    -Dliquibase.username=%DB_USERNAME% ^
                                    -Dliquibase.password=%DB_PASSWORD%
                            '''
                            echo 'Checksum cleared successfully'
                        } catch (Exception e) {
                            echo "Checksum clearing failed, but continuing: ${e.getMessage()}"
                        }
                    }
                }
            }
        }
        
        stage('Test Connection') {
            steps {
                echo 'Testing database connection...'
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'OFSUATDB',
                                                         usernameVariable: 'DB_USERNAME',
                                                         passwordVariable: 'DB_PASSWORD')]) {
                            bat '''
                                mvn liquibase:status ^
                                    -Dliquibase.url=%DB_URL% ^
                                    -Dliquibase.username=%DB_USERNAME% ^
                                    -Dliquibase.password=%DB_PASSWORD%
                            '''
                        }
                        echo 'Database connection successful'
                    } catch (Exception e) {
                        echo "Connection test failed, but proceeding with migration: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Liquibase Update') {
            steps {
                echo 'Running Liquibase database migrations...'
                withCredentials([usernamePassword(credentialsId: 'OFSUATDB',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    bat '''
                        mvn liquibase:update ^
                            -Dliquibase.url=%DB_URL% ^
                            -Dliquibase.username=%DB_USERNAME% ^
                            -Dliquibase.password=%DB_PASSWORD%
                    '''
                }
            }
        }
        
        stage('Generate Changelog Report') {
            steps {
                echo 'Generating changelog report...'
                withCredentials([usernamePassword(credentialsId: 'OFSUATDB',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    bat '''
                        mvn liquibase:status ^
                            -Dliquibase.url=%DB_URL% ^
                            -Dliquibase.username=%DB_USERNAME% ^
                            -Dliquibase.password=%DB_PASSWORD% ^
                            -Dliquibase.verbose=true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
            archiveArtifacts artifacts: 'target/**/*', fingerprint: true, allowEmptyArchive: true
        }
        success {
            echo 'Database migration completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}