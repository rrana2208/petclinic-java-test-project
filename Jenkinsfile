pipeline {
    agent any

    parameters {
        string(name: 'ECR_REGISTRY_ID', defaultValue: '521092179643', description: 'AWS ECR Registry ID')
        string(name: 'ECR_REPOSITORY_NAME', defaultValue: 'petclinic', description: 'ECR Repository Name')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS Region')
    }

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rrana2208/petclinic-java-test-project.git'
            }
        }

        stage('Build JAR') {
            steps {
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh './mvnw spring-boot:build-image -DskipTests'
            }
        }

        stage('Login to ECR & Tag Image') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN') // optional
                ]) {
                    script {
                        env.IMAGE_NAME = "${params.ECR_REGISTRY_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com/${params.ECR_REPOSITORY_NAME}"

                        sh """
                            echo "🔐 Logging into ECR..."
                            export AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}
                            ${env.AWS_SESSION_TOKEN ? "export AWS_SESSION_TOKEN=${env.AWS_SESSION_TOKEN}" : ""}
                            aws ecr get-login-password --region ${params.AWS_REGION} | docker login --username AWS --password-stdin ${env.IMAGE_NAME}

                            echo "🔁 Tagging local image for ECR..."
                            docker tag docker.io/library/spring-petclinic:3.4.0-SNAPSHOT ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                            docker tag docker.io/library/spring-petclinic:3.4.0-SNAPSHOT ${env.IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                    docker push ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                    docker push ${env.IMAGE_NAME}:latest
                """
            }
        }

        stage('Update Deployment Manifest') {
            steps {
                sh """
                    sed -i 's|image: .*|image: ${env.IMAGE_NAME}:${env.IMAGE_TAG}|' k8s/*.yml
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig-credential-id', variable: 'KUBECONFIG'),
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'aws-session-token', variable: 'AWS_SESSION_TOKEN')
                ]) {
                    script {
                        def namespace = 'petclinic'

                        sh """
                            echo "🔐 Creating or updating imagePullSecret..."
                            export AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}
                            ${env.AWS_SESSION_TOKEN ? "export AWS_SESSION_TOKEN=${env.AWS_SESSION_TOKEN}" : ""}
                            SERVER=${params.ECR_REGISTRY_ID}.dkr.ecr.${params.AWS_REGION}.amazonaws.com
                            PASSWORD=\$(aws ecr get-login-password --region ${params.AWS_REGION})
                            kubectl create secret docker-registry ecr-creds \\
                              --docker-server=\$SERVER \\
                              --docker-username=AWS \\
                              --docker-password="\$PASSWORD" \\
                              --docker-email=you@example.com \\
                              -n ${namespace} --dry-run=client -o yaml | kubectl apply -f - --kubeconfig=\$KUBECONFIG

                            echo "🚀 Deploying Kubernetes resources from k8s/..."
                            kubectl apply -n ${namespace} -f k8s --kubeconfig=\$KUBECONFIG
                        """
                    }
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                emailext(
                    to: 'always@foo.com',
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                    subject: "Build ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p><strong>Build Status:</strong> ${buildStatus}</p>
                        <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    mimeType: 'text/html'
                )
            }
        }
        success {
            echo "✅ Successfully deployed: ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed."
        }
    }
}
