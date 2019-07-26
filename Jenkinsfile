/*
    Create the kubernetes namespace
 */
def createNamespace (namespace) {
    echo "Creating namespace ${namespace} if needed"

    sh "[ ! -z \"\$(kubectl get ns ${namespace} -o name 2>/dev/null)\" ] || kubectl create ns ${namespace}"
}

/*
    Helm install
 */
def helmInstallGB (namespace, release) {
    echo "Installing ${release} in ${namespace}"

    script {
        sh "helm repo add my-repo https://ibm.github.io/helm101/"
        sh "helm repo update"
        sh """
         helm install --name=${release}  --namespace=${namespace} my-repo/guestbook --set namespace=${namespace}          
        """
        sh "kubectl set image deploy redis-slave --namespace=${namespace} redis-slave=k8s.gcr.io/redis-slave:v2"
        sh "sleep 5"
    }
}

def helmInstall (namespace, release, repo) {
    echo "Installing ${release} in ${namespace}"

    script {
       
        sh """
         helm install --name=${release}  --namespace=${namespace} ${repo} --set namespace=${namespace}          
        """
        sh "sleep 5"
    }
}

/*
    Helm delete (if exists)
 */
def helmDelete (namespace, release) {
    echo "Deleting ${release} in ${namespace} if deployed"

    script {
        sh "[ -z \"\$(helm ls --short ${release} 2>/dev/null)\" ] || helm delete --purge ${release}"
    }
}


/*
    Test with a simple curl and check we get 200 back
 */
def curlTest (namespace, out) {
    echo "Running tests in ${namespace}"

    script {
        if (out.equals('')) {
            out = 'http_code'
        }

        // Get deployment's service IP
        def svc_ip = sh (
                returnStdout: true,
                script: "kubectl get svc -n ${namespace} | awk '{print \$3}'"
        )

        if (svc_ip.equals('')) {
            echo "ERROR: Getting service IP failed"
            sh 'exit 1'
        }

        echo "svc_ip is ${svc_ip}"
        url = 'http://' + svc_ip

        
    }
}


/*
    This is the main pipeline section with the stages of the CI/CD
 */
pipeline {

    options {
        // Build auto timeout
        timeout(time: 60, unit: 'MINUTES')
    }

    

    parameters {
        string (name: 'GIT_BRANCH',           defaultValue: 'master',  description: 'Git branch to build')
        booleanParam (name: 'DEPLOY_TO_PROD', defaultValue: false,     description: 'If build and tests are good, proceed and deploy to production without manual approval')


      
    }

    // In this example, all is built and run from the master
    agent { node { label 'master' } }

    // Pipeline stages
    stages {

        ////////// Step 1 //////////
        stage('Git clone and setup') {
            steps {
                echo "Check out code"
                git branch: "master",
                        credentialsId: 'Mun-s',
                        url: 'https://github.com/Mun-s/jenkins-pipeline-kubernetes.git'

                // Validate kubectl
                sh "kubectl cluster-info"

                // Init helm client

                sh "[ -z \"\$(kubectl get deploy tiller-deploy -n kube-system 2>/dev/null)\" ] || kubectl  delete deploy tiller-deploy -n kube-system"
                sh "sleep 3"
                sh "helm init"
                sh "sleep 3"
                sh "[ -z \"\$(kubectl get serviceaccount tiller -n kube-system 2>/dev/null)\" ] || kubectl  delete serviceaccount tiller -n kube-system"
                sh "sleep 3"
                sh "kubectl  create serviceaccount --namespace kube-system tiller"
                sh "sleep 3"
                sh "[ -z \"\$(kubectl get clusterrolebinding tiller-cluster-rule 2>/dev/null)\" ] || kubectl  delete clusterrolebinding tiller-cluster-rule"
                sh "sleep 3"
                sh "kubectl create  clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller"
                sh "sleep 3"
                sh "kubectl  patch deploy -n kube-system tiller-deploy -p '{\"spec\":{\"template\":{\"spec\":{\"serviceaccount\":\"tiller\"}}}}'"
                sh "helm init --service-account tiller --upgrade"    
                sh "sleep 3"  
                
               
            }
        }


        stage('Helm guestbook  deployment') {
            steps {
                  script {
                        echo "adding  guestbook to the  repo and deploying it"
                        createNamespace ("development")
                        helmDelete ("development", "guestbook")
                        helmInstallGB("development", "guestbook")
                        echo "adding  guestbook to the  repo and deploying it"
                        sh "sleep 5"
                    }                 
            }
        }

        stage('Helm prometheus  deployment') {
            steps {
                  script {
                        echo "adding  prometheus to the  repo and deploying it"
                        createNamespace ("monitoring")
                        helmDelete ("monitoring", "prometheus")
                        helmInstall("monitoring", "prometheus", "stable/prometheus")
                        echo "adding  prometheus to the  repo and deploying it"
                        sh "sleep 5"
                    }                 
            }
        }


        stage('Helm grafana  deployment') {
            steps {
                  script {
                        echo "adding  grafana to the  repo and deploying it"
                        createNamespace ("monitoring")
                        helmDelete ("monitoring", "garfana")
                        helmInstall("monitoring", "grafana", "stable/grafana")
                        echo "adding  grafana to the  repo and deploying it"
                        sh "sleep 5"
                    }                 
            }
        }

        stage('Helm ELK  deployment') {
            steps {
                  script {
                        echo "adding  ELK to the  repo and deploying it"
                        createNamespace ("monitoring")
                        helmDelete ("monitoring", "elk")
                        helmInstall("monitoring", "elk", "stable/elastic-stack")
                        echo "adding  ELK to the  repo and deploying it"
                        sh "sleep 5"
                    }                 
            }
        }

         stage('blue-green deployment') {
            steps {
                script {
                    // Remove release if exists
                    sh "kubectl  apply -f traefik-dep.yaml"
                    sh "sleep 15"
                    sh "kubectl apply  -f blue-green.yaml"
                    
                }
            }
        }

        stage('Helm istio  deployment') {
            steps {
                  script {
                      sh "helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.7/charts/"

                      sh "helm install --name istio-init --namespace istio-system istio.io/istio-init"
                      sh "helm install --name istio --namespace istio-system --set grafana.enabled=true istio.io/istio"
                       

              }                 
            }
        }

       /* stage('Helm istio  deployment') {
            steps {
                  script {
                        echo "adding  istio to the  repo and deploying it"
                        createNamespace ("istio-system")
                        helmDelete ("istio-system", "istio-init")
                        sh "helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com"
                        sh "helm repo update"
                        helmInstall("istio-system","istio-init", "incubator/istio")
                        echo "adding  guestbook to the  repo and deploying it"
                        sh "sleep 5"
                    }                 
            }
        }
        */
       stage('canary deployment') {
            steps {
                  script {
                       createNamespace ("canary")
                       sh "kubectl create -f helloworld-gateway.yaml"
                       sh "kubectl create -f helloworld.yaml"
                       sh "kubectl label namespace canary istio-injection=enabled"
                       sh "kubectl delete pod -l app=helloworld -n canary"
                       sh "kubectl autoscale deployment helloworld-v1 -n canary --cpu-percent=50 --min=1 --max=10"
                       sh "kubectl autoscale deployment helloworld-v2 -n canary --cpu-percent=50 --min=1 --max=10"
                       //sh "loadgen.sh &"
                       //sh "loadgen.sh &"
                       sh "sleep 40"
                       sh "kubectl  get  hpa -n canary"

              }                 
            }
        }

       stage('Dev tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        curlTest ("development", 'http_code')
                    }
                }
                stage('Curl total_time') {
                    steps {
                        curlTest ("development", 'time_total')
                    }
                }
                stage('Curl size_download') {
                    steps {
                        curlTest ("development", 'size_download')
                    }
                }
            }
        }

        stage('Cleanup dev') {
            steps {
                script {
                    // Remove release if exists
                    helmDelete ("development", "guestbook")
                }
            }
        }
  
    
    }
}

