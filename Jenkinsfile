pipeline {
    agent any

    environment {
        GCP_PROJECT = "fundamental-run-464208-v1"
        REGION = "us-central1"
        REPO = "votingapp"
        IMAGE_BASE = "us-central1-docker.pkg.dev/${GCP_PROJECT}/${REPO}"
        KUBECONFIG = credentials('kubeconfig')
        CLUSTER_NAME = "jenkins-tfgke-cluster"
        ZONE="us-central1-a"
          }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/TCRDINSEH/vote.git'
                credentialsId: "${GIT_CREDENTIALS}"
            }
        }

        stage('Authenticate with GCP') {
            steps {
                withCredentials([file(credentialsId: "${GCP_CREDENTIALS}", variable: 'GCP_KEYFILE')]) {
                    sh '''
                    echo "🔐 Authenticating to GCP..."
                    gcloud auth activate-service-account --key-file=$GCP_KEYFILE
                    gcloud auth configure-docker ${REGION}-docker.pkg.dev -q
                    '''
                }
            }
        }

        stage('Build and Push Images') {
            steps {
                script {
                    def services = ['vote', 'result', 'worker', 'seed-data']
                    for (svc in services) {
                        echo "🔨 Building and pushing image for ${svc}"
                        sh """
                        cd ${svc}
                        docker build -t ${IMAGE_BASE}/${svc}:v.${BUILD_NUMBER} .
                        docker push ${IMAGE_BASE}/${svc}:v.${BUILD_NUMBER}
                        cd ..
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    // Write kubeconfig from Jenkins secret
                    writeFile file: 'config', text: KUBECONFIG
                    sh '''
                    gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $GCP_PROJECT
                    export KUBECONFIG=config

                    echo "🔁 Updating Kubernetes manifests..."
                    cd k8s-specifications

                    # Update images dynamically in YAML before applying (optional)
                    sed -i "s|image: .*/vote:.*|image: ${IMAGE_BASE}/vote:v${BUILD_NUMBER}|" vote-deployment.yaml
                    sed -i "s|image: .*/result:.*|image: ${IMAGE_BASE}/result:v${BUILD_NUMBER}|" result-deployment.yaml
                    sed -i "s|image: .*/worker:.*|image: ${IMAGE_BASE}/worker:v${BUILD_NUMBER}|" worker-deployment.yaml
                    sed -i "s|image: .*/seed-data:.*|image: ${IMAGE_BASE}/seed-data:v${BUILD_NUMBER}|" seed-data-job.yaml

                    echo "🚀 Applying manifests to cluster..."
                    kubectl apply -f .
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment successful to GKE!"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
