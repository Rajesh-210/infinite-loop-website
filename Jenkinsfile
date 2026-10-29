pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        IMAGE_NAME     = 'infinite-loop-website'
        CONTAINER_NAME = 'infinite-loop-website'
        HOST_PORT      = '8081'
        WEBSITE_IP     = '15.206.149.132'
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '463771485942'
        ECR_REPOSITORY = 'infinite-loop-website'
        ECR_REGISTRY   = '463771485942.dkr.ecr.ap-south-1.amazonaws.com'
        ECR_IMAGE      = '463771485942.dkr.ecr.ap-south-1.amazonaws.com/infinite-loop-website'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Gitleaks Secret Scan') {
            steps {
                sh '''
                    set -eu
                    mkdir -p reports

                    gitleaks detect \
                        --source . \
                        --redact \
                        --report-format json \
                        --report-path reports/gitleaks-report.json
                '''
            }
        }

        stage('Validate Project') {
            steps {
                sh '''
                    set -eu

                    test -f index.html
                    test -f Dockerfile
                    test -d css
                    test -d js
                    test -d img

                    echo "Project structure validation successful."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        withEnv(["SCANNER_HOME=${scannerHome}"]) {
                            sh '''
                                set -eu

                                if [ -z "${SONAR_AUTH_TOKEN:-}" ]; then
                                    echo "ERROR: SONAR_AUTH_TOKEN is unavailable."
                                    exit 1
                                fi

                                "$SCANNER_HOME/bin/sonar-scanner" \
                                    -Dsonar.token="$SONAR_AUTH_TOKEN"
                            '''
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} .

                    docker tag \
                        ${IMAGE_NAME}:${BUILD_NUMBER} \
                        ${IMAGE_NAME}:latest

                    echo "Docker image built successfully."
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    set -eu

                    mkdir -p reports

                    trivy --version

                    # Generate a complete vulnerability report.
                    trivy image \
                        --scanners vuln \
                        --format table \
                        --output reports/trivy-report.txt \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    # Fail the pipeline for HIGH or CRITICAL vulnerabilities.
                    trivy image \
                        --scanners vuln \
                        --severity HIGH,CRITICAL \
                        --exit-code 1 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "Trivy security scan passed."
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    set -eu

                    TEST_CONTAINER="infinite-loop-test-${BUILD_NUMBER}"

                    cleanup() {
                        docker rm -f "$TEST_CONTAINER" \
                            >/dev/null 2>&1 || true
                    }

                    trap cleanup EXIT

                    docker run -d \
                        --name "$TEST_CONTAINER" \
                        -p 127.0.0.1::80 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    TEST_PORT=$(docker port "$TEST_CONTAINER" 80/tcp |
                        sed 's/.*://')

                    for i in $(seq 1 20); do
                        if curl -fsS \
                            "http://127.0.0.1:${TEST_PORT}/" \
                            -o /dev/null; then

                            echo "Temporary Docker image test passed."
                            exit 0
                        fi

                        sleep 2
                    done

                    echo "ERROR: Temporary container health check failed."
                    exit 1
                '''
            }
        }

        stage('Push Image to Amazon ECR') {
            steps {
                sh '''
                    set -eu

                    echo "Checking AWS identity available to Jenkins..."
                    aws sts get-caller-identity

                    echo "Logging in to Amazon ECR..."
                    aws ecr get-login-password --region "$AWS_REGION" | \
                        docker login --username AWS --password-stdin "$ECR_REGISTRY"

                    echo "Tagging versioned image for ECR..."
                    docker tag "${IMAGE_NAME}:${BUILD_NUMBER}" "${ECR_IMAGE}:${BUILD_NUMBER}"
                    docker tag "${IMAGE_NAME}:${BUILD_NUMBER}" "${ECR_IMAGE}:latest"

                    echo "Pushing versioned image..."
                    docker push "${ECR_IMAGE}:${BUILD_NUMBER}"

                    echo "Pushing latest tag..."
                    docker push "${ECR_IMAGE}:latest"

                    echo "Image pushed successfully to Amazon ECR."
                '''
            }
        }

        // UPDATED STAGE: BLUE-GREEN DEPLOYMENT AND VERIFICATION
        stage('Deploy and Verify') {
            steps {
                sh '''
                    set -eu

                    NEW_IMAGE="${ECR_IMAGE}:${BUILD_NUMBER}"
                    BLUE_CONTAINER="infinite-loop-website"
                    GREEN_CONTAINER="infinite-loop-green"
                    NGINX_SITE="/etc/nginx/sites-available/default"

                    echo "New image: ${NEW_IMAGE}"

                    # Detect the currently active Nginx upstream.
                    ACTIVE_PORT=$(grep -oE \
                        'proxy_pass http://127[.]0[.]0[.]1:(8081|8082);' \
                        "$NGINX_SITE" |
                        head -n 1 |
                        grep -oE '8081|8082' || true)

                    if [ "$ACTIVE_PORT" != "8081" ] && [ "$ACTIVE_PORT" != "8082" ]; then
                        echo "ERROR: Could not determine active Nginx upstream."
                        exit 1
                    fi

                    # Select the inactive deployment slot.
                    if [ "$ACTIVE_PORT" = "8081" ]; then
                        TARGET_PORT="8082"
                        TARGET_CONTAINER="$GREEN_CONTAINER"
                        ACTIVE_CONTAINER="$BLUE_CONTAINER"
                    else
                        TARGET_PORT="8081"
                        TARGET_CONTAINER="$BLUE_CONTAINER"
                        ACTIVE_CONTAINER="$GREEN_CONTAINER"
                    fi

                    echo "Active production port: ${ACTIVE_PORT}"
                    echo "Candidate port: ${TARGET_PORT}"
                    echo "Candidate container: ${TARGET_CONTAINER}"

                    # Save the image used by the active production container.
                    PREVIOUS_IMAGE=$(docker inspect \
                        --format='{{.Config.Image}}' \
                        "$ACTIVE_CONTAINER" 2>/dev/null || true)

                    echo "Previous production container: ${ACTIVE_CONTAINER}"
                    echo "Previous production image: ${PREVIOUS_IMAGE:-unknown}"

                    # Remove only the inactive candidate slot.
                    # The current production container remains running.
                    docker rm -f "$TARGET_CONTAINER" \
                        >/dev/null 2>&1 || true

                    # Start the candidate container.
                    if ! docker run -d \
                        --restart unless-stopped \
                        --name "$TARGET_CONTAINER" \
                        -p "127.0.0.1:${TARGET_PORT}:80" \
                        "$NEW_IMAGE"; then

                        echo "ERROR: Candidate container failed to start."
                        exit 1
                    fi

                    # Verify candidate HTTP health before switching traffic.
                    CANDIDATE_HEALTHY=false

                    for i in $(seq 1 20); do
                        if curl --connect-timeout 3 --max-time 5 -fsS \
                            "http://127.0.0.1:${TARGET_PORT}/" \
                            -o /dev/null; then

                            echo "Candidate HTTP health check passed."
                            CANDIDATE_HEALTHY=true
                            break
                        fi

                        echo "Waiting for candidate... attempt ${i}/20"
                        sleep 3
                    done

                    if [ "$CANDIDATE_HEALTHY" != "true" ]; then
                        echo "ERROR: Candidate failed health checks."

                        docker logs "$TARGET_CONTAINER" || true

                        docker rm -f "$TARGET_CONTAINER" \
                            >/dev/null 2>&1 || true

                        exit 1
                    fi

                    # Switch Nginx only after candidate health succeeds.
                    if ! sudo -n /usr/local/sbin/switch-website-upstream \
                        "$TARGET_PORT"; then

                        echo "ERROR: Nginx switch failed."
                        echo "Attempting to restore the previous production upstream."

                        if ! sudo -n /usr/local/sbin/switch-website-upstream \
                            "$ACTIVE_PORT"; then

                            echo "CRITICAL: Nginx rollback failed."
                            echo "Candidate retained for troubleshooting: ${TARGET_CONTAINER}"
                            exit 1
                        fi

                        docker rm -f "$TARGET_CONTAINER" \
                            >/dev/null 2>&1 || true

                        exit 1
                    fi

                    # Perform an additional HTTPS verification through Nginx.
                    if ! curl --connect-timeout 5 --max-time 10 -kfsS \
                        --resolve "${WEBSITE_IP}:443:127.0.0.1" \
                        "https://${WEBSITE_IP}/" \
                        -o /dev/null; then

                        echo "ERROR: Post-switch HTTPS verification failed."
                        echo "Switching back to the previous production port."

                        if ! sudo -n /usr/local/sbin/switch-website-upstream \
                            "$ACTIVE_PORT"; then

                            echo "CRITICAL: Nginx rollback failed."
                            echo "Candidate retained for troubleshooting: ${TARGET_CONTAINER}"
                            exit 1
                        fi

                        docker rm -f "$TARGET_CONTAINER" \
                            >/dev/null 2>&1 || true

                        exit 1
                    fi

                    echo "SUCCESS: Blue-Green deployment completed."
                    echo "Active production port: ${TARGET_PORT}"
                    echo "Previous production port: ${ACTIVE_PORT}"
                    echo "Previous production container retained: ${ACTIVE_CONTAINER}"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'reports/trivy-report.txt,reports/gitleaks-report.json',
                allowEmptyArchive: true
            )

            echo 'Security report archiving completed.'
        }

        success {
            echo 'SUCCESS: Pipeline and deployment verification passed.'
        }

        failure {
            echo 'FAILURE: Pipeline failed. Review the console output.'
        }
    }
}
