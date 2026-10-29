
pipeline {
    agent any

    triggers {
        // Check S3 for changes every minute
        cron('* * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        S3_BUCKET = 'YOUR_S3_BUCKET_NAME'
        S3_KEY = 'release/trigger.txt'

        IMAGE_NAME = 's3-k8s-demo'
        NEW_TRIGGER = 'false'
    }

    stages {

        stage('Check prerequisites') {
            steps {
                sh '''
                    set -eu

                    command -v aws
                    command -v docker
                    command -v kubectl
                    command -v ctr

                    aws --version
                    docker --version
                    kubectl version --client
                    sudo ctr version
                '''
            }
        }

        stage('Check S3 for changes') {
            steps {
                script {
                    def currentETag = sh(
                        script: """
                            aws s3api head-object \
                              --region '${env.AWS_REGION}' \
                              --bucket '${env.S3_BUCKET}' \
                              --key '${env.S3_KEY}' \
                              --query ETag \
                              --output text
                        """,
                        returnStdout: true
                    ).trim().replace('"', '')

                    def previousETag = fileExists('.last-etag') ?
                        readFile('.last-etag').trim() : ''

                    if (currentETag != previousETag) {
                        env.NEW_TRIGGER = 'true'
                        env.S3_ETAG = currentETag

                        echo 'S3 object changed. Deployment will start.'
                    } else {
                        echo 'No S3 change detected. Deployment will be skipped.'
                    }
                }
            }
        }

        stage('Download S3 trigger') {
            when {
                expression { env.NEW_TRIGGER == 'true' }
            }
            steps {
                sh '''
                    set -eu

                    aws s3 cp \
                      "s3://${S3_BUCKET}/${S3_KEY}" \
                      trigger.txt \
                      --region "${AWS_REGION}"

                    echo "S3 trigger downloaded."
                    cat trigger.txt
                '''
            }
        }

        stage('Test application') {
            when {
                expression { env.NEW_TRIGGER == 'true' }
            }
            steps {
                sh '''
                    set -eu

                    test -s index.html
                    grep -q "Hello from Jenkins" index.html

                    echo "Application test passed."
                '''
            }
        }

        stage('Build Docker image') {
            when {
                expression { env.NEW_TRIGGER == 'true' }
            }
            steps {
                sh '''
                    set -eu

                    IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"

                    docker build -t "$IMAGE" .

                    docker image inspect "$IMAGE" >/dev/null

                    echo "Docker image built: $IMAGE"
                '''
            }
        }

        stage('Import image into containerd') {
            when {
                expression { env.NEW_TRIGGER == 'true' }
            }
            steps {
                sh '''
                    set -eu

                    IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"

                    docker save "$IMAGE" |
                      sudo ctr --namespace k8s.io images import -

                    sudo ctr --namespace k8s.io images ls |
                      grep -F "$IMAGE"

                    echo "Image imported into containerd."
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { env.NEW_TRIGGER == 'true' }
            }
            steps {
                sh '''
                    set -eu

                    IMAGE="${IMAGE_NAME}:${BUILD_NUMBER}"

                    kubectl get nodes

                    sed \
                      "s|s3-k8s-demo:INITIAL|${IMAGE}|g" \
                      k8s/deployment.yaml \
                      > deployment-rendered.yaml

                    kubectl apply -f deployment-rendered.yaml
                    kubectl apply -f k8s/service.yaml

                    kubectl rollout status \
                      deployment/s3-k8s-demo \
                      --timeout=180s

                    kubectl get deployment s3-k8s-demo
                    kubectl get pods -l app=s3-k8s-demo
                    kubectl get service s3-k8s-demo

                    echo "Kubernetes deployment completed."
                '''
            }
        }
    }

    post {
        success {
            script {
                if (env.NEW_TRIGGER == 'true') {
                    writeFile(
                        file: '.last-etag',
                        text: env.S3_ETAG + '\n'
                    )

                    echo 'Successful deployment recorded.'
                }
            }
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }
    }
}
