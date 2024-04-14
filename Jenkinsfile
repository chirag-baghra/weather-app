pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker_cred')
        //EC2_CREDENTIALS = credentials('ec2_cred')
        privateKey = credentials('dev_server_cred')
        //GIT_CREDENTIALS = credentials('git_cred')
        EC2_PROD_Key = credentials('EC2_PROD_Key')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'python3 manage.py test'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("jobychacko/weather-app:${env.BUILD_ID}")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo 'Testing..'
                withCredentials([usernamePassword(credentialsId: 'docker_cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        
                        docker tag jobychacko/weather-app:${env.BUILD_ID} jobychacko/weather-app:latest
                        docker push jobychacko/weather-app:latest
                    """
                }
            }
        }
       
        stage('Deploy and Test on DEV_EC2') {
    steps {
        script {
            // Start the Docker container
            sh """
            ssh -v -o StrictHostKeyChecking=no -i ${privateKey} ubuntu@${env.EC2_IP} '
               
                if sudo docker ps --format '{{.Ports}}' | grep -q '0.0.0.0:8000->8000/tcp'; then
                    echo "Container is already running on port 8000. Stopping it..."
                    sudo docker stop \$(sudo docker ps -qf "port=8000")
                    echo "Removing the container..."
                     sudo docker rm \$(sudo docker ps -aqf "port=8000")
                fi
                
                # Pull the latest Docker image
                sudo docker pull jobychacko/weather-app:latest
                
                # Start the Docker container
                sudo docker run -d -p 8000:8000 jobychacko/weather-app:latest
            '
        """
            
            // Execute Selenium tests against the Docker container on the development server
            def testResult = sh (
                script: '''
                    ssh -o StrictHostKeyChecking=no -i ${privateKey} ubuntu@${env.EC2_IP} << 'EOF'
                        containerId=$(sudo docker ps -qf "ancestor=jobychacko/weather-app:latest")
                        sudo docker exec $containerId python3 /app/selenium_test.py
                    EOF
                ''',
                returnStatus: true
            )

            // Store the test result
            env.TEST_RESULT = testResult
        }
    }
}

stage('Check Test Result') {
    when {
        expression {
            // Check if the test result is not successful
            return env.TEST_RESULT != 0
        }
    }
    steps {
        error("Selenium tests failed on the Docker container.")
    }
}
         stage('Merge to Master') {
    when {
        // This stage is executed only if DEPLOYMENT_EXIT_CODE is 0
        expression { return env.TEST_RESULT.toInteger() == 0 }
    }
            steps {
                script {
                    withCredentials([string(credentialsId: 'git_cred', variable: 'GIT_TOKEN')]){
                        // Configure remote with access token for authentication
                        sh """
                            git remote set-url origin https://x-access-token:${GIT_TOKEN}@github.com/jobychacko2001/weather-app.git
                        """
        
                        // Merge main branch into master
                        sh """
                            git fetch --all
                            git checkout master
                            git pull origin master
                            git merge origin/main --no-ff -m "Merge main into master by Jenkins"
                        """
        
                        // Push the changes back to the master branch
                        sh """
                            git push origin master
                        """
                    }
                }
            }
        }

            stage('Deploy to PROD_EC2') {
                steps {
                    script {
                        // Execute the deployment command
                        sh(script: """
                            ssh -v -o StrictHostKeyChecking=no -i ${EC2_PROD_Key} ubuntu@${env.EC2_PROD_IP} '
                              sudo docker pull jobychacko/weather-app:latest
                              sudo docker run -d -p 8000:8000 jobychacko/weather-app:latest
                            '
                        """, returnStdout: true).trim()
                    }
                }
            }

         
    }
}
