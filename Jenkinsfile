
pipeline {

    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        APP_NAME = 'mevi-html-site'
        IMAGE_NAME = 'mevi-html-site:1.0'
        NETWORK_NAME = 'mevi-network'
        JENKINS_CONTAINER = 'jenkins'
        HOST_PORT = '8082'
        CONTAINER_PORT = '80'
    }

    stages {

        // STAGE 1: CHECKOUT GITHUB
        stage('1. Checkout GitHub') {
            steps {
                echo 'Checking out source code from GitHub'

                checkout scm

                sh '''
                    set -eu

                    echo "Project files:"
                    ls -la

                    test -f Dockerfile
                    test -f index.html
                    test -f Jenkinsfile

                    echo "All required files found."
                '''
            }
        }

        // STAGE 2: CHECK DOCKER
        stage('2. Check Docker') {
            steps {
                echo 'Checking Docker installation'

                sh '''
                    set -eu
                    docker --version
                    docker info
                '''
            }
        }

        // STAGE 3: CONFIGURE NETWORK
        stage('3. Configure Docker Network') {
            steps {
                echo 'Configuring Docker network'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" >/dev/null 2>&1; then
                        docker network create "$NETWORK_NAME"
                    fi

                    echo "Docker network is ready."
                    docker network ls
                '''
            }
        }

        // STAGE 4: BUILD WEBSITE IMAGE
        stage('4. Build Website Image') {
            steps {
                echo 'Building Docker image'

                sh '''
                    set -eu

                    docker build -t "$IMAGE_NAME" .

                    echo "Docker image built successfully."

                    docker image ls "$IMAGE_NAME"
                '''
            }
        }

        // STAGE 5: REMOVE OLD CONTAINER
        stage('5. Remove Old Website') {
            steps {
                echo 'Removing previous website container'

                sh '''
                    docker rm -f "$APP_NAME" 2>/dev/null || true
                '''
            }
        }

        // STAGE 6: DEPLOY WEBSITE
        stage('6. Deploy Website') {
            steps {
                echo 'Deploying website container'

                sh '''
                    set -eu

                    docker run -d \
                        --name "$APP_NAME" \
                        --network "$NETWORK_NAME" \
                        --restart unless-stopped \
                        -p "$HOST_PORT:$CONTAINER_PORT" \
                        "$IMAGE_NAME"

                    echo "Website deployed successfully."

                    docker ps -a
                '''
            }
        }

        // STAGE 7: VERIFY CONTAINER
        stage('7. Verify Container') {
            steps {
                echo 'Checking container status'

                sh '''
                    set -eu

                    RUNNING=$(docker inspect \
                        --format='{{.State.Running}}' "$APP_NAME")

                    if [ "$RUNNING" != "true" ]; then
                        echo "ERROR: Container is not running."
                        docker logs "$APP_NAME"
                        exit 1
                    fi

                    echo "SUCCESS: Container is running."
                '''
            }
        }

        // STAGE 8: VERIFY DOCKER NETWORK
        stage('8. Verify Docker Network') {
            steps {
                echo 'Verifying network connectivity'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' | grep -q "$APP_NAME"; then

                        echo "ERROR: Website is not connected to network."
                        exit 1
                    fi

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' | grep -q "$JENKINS_CONTAINER"; then

                        echo "Connecting Jenkins to Docker network..."

                        docker network connect \
                            "$NETWORK_NAME" "$JENKINS_CONTAINER"
                    fi

                    echo "Jenkins and website network verified."
                '''
            }
        }

        // STAGE 9: TEST WEBSITE
        stage('9. Test Website') {
            steps {
                echo 'Testing website HTTP response and content'

                sh '''
                    set -eu

                    SUCCESS=0

                    for i in $(seq 1 15); do

                        echo "HTTP test attempt $i of 15"

                        if curl -fsS \
                            --connect-timeout 3 \
                            --max-time 5 \
                            "http://$APP_NAME/" \
                            -o /tmp/mevi-site.html; then

                            SUCCESS=1
                            break
                        fi

                        sleep 2
                    done

                    if [ "$SUCCESS" -ne 1 ]; then
                        echo "ERROR: Website did not respond."
                        exit 1
                    fi

                    echo "SUCCESS: Website returned HTTP response."

                    if grep -qi "ShopEase - Online Shopping" /tmp/mevi-site.html; then

                        echo "SUCCESS: ShopEase website title verified."

                    else

                        echo "ERROR: Website title not found."

                        grep -i "<title>" /tmp/mevi-site.html || true

                        exit 1
                    fi
                '''
            }
        }

        // STAGE 10: VERIFY PUBLISHED PORT
        stage('10. Verify Published Port') {
            steps {
                echo 'Verifying published website port'

                sh '''
                    set -eu

                    echo "Checking published port..."

                    PORT_MAPPING=$(docker port "$APP_NAME" "$CONTAINER_PORT")

                    echo "$PORT_MAPPING"

                    echo "$PORT_MAPPING" | grep -q "$HOST_PORT"

                    echo "SUCCESS: Port mapping verified."

                    echo "Checking website inside container..."

                    docker exec "$APP_NAME" \
                        wget -q -O - http://127.0.0.1/ \
                        > /tmp/mevi-container-page.html

                    if grep -qi "ShopEase - Online Shopping" /tmp/mevi-container-page.html; then

                        echo "SUCCESS: Website content verified."

                    else

                        echo "ERROR: Website content verification failed."
                        exit 1
                    fi

                    echo "Website URL: http://localhost:8082/"
                '''
            }
        }

        // STAGE 11: DEPLOYMENT SUMMARY
        stage('11. Deployment Summary') {
            steps {
                echo 'Displaying deployment summary'

                sh '''
                    echo "=================================="
                    echo "       DEPLOYMENT SUMMARY"
                    echo "=================================="

                    echo "Docker Image:"
                    docker image ls "$IMAGE_NAME"

                    echo "Website Container:"
                    docker ps --filter "name=^/${APP_NAME}$"

                    echo "Docker Network:"
                    docker network ls

                    echo "Published Port:"
                    docker port "$APP_NAME"

                    echo "Website URL:"
                    echo "http://localhost:8082/"

                    echo "=================================="
                    echo "DEPLOYMENT SUCCESSFUL!"
                    echo "=================================="
                '''
            }
        }
    }

    post {

        success {
            echo '''
            ==================================
                PIPELINE SUCCESSFUL!
            ==================================

            GitHub Checkout: SUCCESS
            Docker Check: SUCCESS
            Image Build: SUCCESS
            Network Configuration: SUCCESS
            Website Deployment: SUCCESS
            Container Verification: SUCCESS
            HTTP Verification: SUCCESS
            Content Verification: SUCCESS

            Website URL:
            http://localhost:8082/

            ==================================
            '''
        }

        failure {
            echo '''
            ==================================
                PIPELINE FAILED
            ==================================

            Check the Console Output
            for the first failed stage.

            ==================================
            '''

            sh '''
                echo "Docker containers:"
                docker ps -a || true

                echo "Website logs:"
                docker logs "$APP_NAME" 2>&1 || true

                echo "Docker network:"
                docker network inspect "$NETWORK_NAME" 2>&1 || true
            '''
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
