pipeline {
    environment {
        DOCKER_ID = credentials('DOCKER_ID')
        DOCKER_IMAGE_CATALOGUE_DB = "catalogue-db"
        DOCKER_TAG = "${BUILD_ID}"
        BUILD_AGENT  = ""
        NAMESPACE = credentials("NAMESPACE")
    }
agent any
    stages {
        stage('Build') {
            steps { //create a loop somehow??
                sh 'docker build -t $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:$DOCKER_TAG .'
            }
        }
        stage('Run') {
            steps {
                sh 'docker network create $BUILD_TAG'
                sh 'docker run -d --name $DOCKER_IMAGE_CATALOGUE_DB --rm --network $BUILD_TAG $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:$DOCKER_TAG'
                sh 'docker stop $DOCKER_IMAGE_CATALOGUE_DB'
                sh 'docker network rm $BUILD_TAG'    
            }
        }
        stage('Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                sh 'docker image tag $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:$DOCKER_TAG $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:latest'
                sh 'docker login -u $DOCKER_ID -p $DOCKER_PASS'
                sh 'docker push $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:$DOCKER_TAG && docker push $DOCKER_ID/$DOCKER_IMAGE_CATALOGUE_DB:latest'
            }
        }    
        stage('Deploy EKS') {
            environment { // import Jenkin global variables 
                //KUBECONFIG = credentials("EKS_CONFIG")  
                AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
                AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
                AWSREGION = "eu-west-3"
                EKSCLUSTERNAME = credentials("EKS_CLUSTER")
            }
            steps {
                script {
                    dir('manifests') {
                        sh "aws eks update-kubeconfig --name sock-shop-eks --region $AWSREGION"
                        // Check if the namespace exists
                        def namespaceExists = sh(script: "kubectl get namespace $NAMESPACE", returnStatus: true)
                        if (namespaceExists == 0) {
                            echo "Namespace '$NAMESPACE' already exists."
                        } else {
                            // Create the namespace
                            sh "kubectl create namespace $NAMESPACE"
                            echo "Namespace '$NAMESPACE' created."
                        }
                        
                        sh 'kubectl apply -f ./statefulset.yaml -n $NAMESPACE'
                        sh 'kubectl apply -f ./service.yaml -n $NAMESPACE'
                        sh 'aws configure set output text'
                        sh 'aws eks list-clusters'
                        sh 'kubectl config view'
                        sh 'kubectl cluster-info --kubeconfig .kube/config'                
                    }
                }
            }
        }
    }
}
