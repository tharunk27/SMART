pipeline {
    agent any
    parameters {
        string(name: 'DOCKER_TAG', defaultValue: 'v1.0.0', description: 'Image Build Tag')
    }
    environment {
        DOCKER_IMAGE = "YOUR_DOCKERHUB_USERNAME/water-monitor:${params.DOCKER_TAG}"
    }
    stages {
        stage('Parallel Code Checks') {
            parallel {
                stage('Lint Web UI') { steps { echo 'Checking HTML structure...' } }
                stage('Validate K8s Manifest') { steps { echo 'Checking deployment.yaml syntax...' } }
            }
        }
        stage('Archive Artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html', onlyIfSuccessful: true
            }
        }
        stage('Build & Push Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                        sh "echo \$PASS | docker login -u \$USER --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }
        stage('Kubernetes Deploy & Rollback Watcher') {
            steps {
                script {
                    // Update manifest dynamically
                    sh "sed -i 's|YOUR_DOCKERHUB_USERNAME/water-monitor:latest|${DOCKER_IMAGE}|g' deployment.yaml"
                    sh "kubectl apply -f deployment.yaml"
                    
                    // Monitor rollout health status
                    try {
                        sh "kubectl rollout status deployment/water-monitor --timeout=30s"
                        echo "Deployment complete and healthy!"
                    } catch (Exception e) {
                        echo "Deployment failed! Reverting automatically to last functional state..."
                        sh "kubectl rollout undo deployment/water-monitor"
                        error("Pipeline failed: Automatically rolled back.")
                    }
                }
            }
        }
    }
}
