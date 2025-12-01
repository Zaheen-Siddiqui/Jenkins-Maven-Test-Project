pipeline {
    agent any

    tools {
        // Keep the names that match your Jenkins Global Tool Configuration
        maven 'Maven'
        jdk 'Java11'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from repository...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building the project...'
                sh 'mvn -B clean compile'
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh 'mvn -B test'
            }
            post {
                always {
                    // Publish test results
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging the application...'
                sh 'mvn -B package'
            }
        }

        stage('Archive') {
            steps {
                echo 'Archiving artifacts...'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Make sure the SonarQube server in Jenkins (Manage Jenkins → Configure System)
                // is named exactly 'SonarQube-Server' and has a Secret Text token credential configured.
                withSonarQubeEnv('SonarQube-Server') {
                    // Use a here-doc style script to keep multi-line commands tidy.
                    sh '''
                      echo "=== Starting SonarQube analysis ==="
                      echo "Injected SONAR_HOST_URL: $SONAR_HOST_URL"
                      echo "Overriding host to host.docker.internal:9000 (reachable from Jenkins container)"
                      # Run sonar analysis (verbose). Use injected token for authentication.
                      mvn -B -DskipTests -Dsonar.host.url=http://host.docker.internal:9000 -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.verbose=true sonar:sonar || true

                      echo "=== Workspace listing (root) ==="
                      pwd
                      ls -la

                      # Show report-task.txt if present (this file contains the ceTaskId used by waitForQualityGate)
                      if [ -f report-task.txt ]; then
                        echo "---- report-task.txt ----"
                        cat report-task.txt
                      else
                        echo "report-task.txt not found in workspace root"
                      fi
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                // If your analyses are large or Sonar is slow, increase timeout as needed.
                timeout(time: 30, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }

    post {
        success {
            echo 'Build completed successfully!'
        }
        failure {
            echo 'Build failed!'
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
