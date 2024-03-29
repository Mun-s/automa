

pre-requisites (manual steps): 

install high avaliabilty cluster through ansible by following :https://github.com/kairen/kubeadm-ansible

install  jenkins in the container using command : 
      
      docker container run -d -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock  -v /Users/manishthakur/jenkins_home:/var/jenkins_home dokcer-jenkins

      DockerFile:
       from jenkinsci/jenkins:lts
  
        USER root
        RUN apt-get update -qq \
            && apt-get install -qqy apt-transport-https ca-certificates curl gnupg2 software-properties-common
        RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
        RUN add-apt-repository \
        "deb [arch=amd64] https://download.docker.com/linux/debian \
        $(lsb_release -cs) \
        stable"
        RUN apt-get update  -qq \
            && apt-get install docker-ce=17.12.1~ce-0~debian -y
        RUN usermod -aG docker jenkins

get inside the jenkins container and install kubectl  and helm binaries:

    Install the kubectl binary under /usr/local/bin/ of master jenkins node
            curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/linux/amd64/kubectl
            chmod +x ./kubectl
            sudo mv ./kubectl /usr/local/bin/kubectl
    Iinstall helm binary under under /usr/local/bin of master jenkins node
            curl https://get.helm.sh/helm-v2.14.2-linux-amd64.tar.gz
            tar -zxvf helm-v2.0.0-linux-amd64.tgz
            mv linux-amd64/helm /usr/local/bin/helm 
            
    mkdir /root/.kube, touch /root/.kube/config and copy the k8s kubeconf(get it from the k8s cluster) in config file. 


Automation: Create  a jenkins pipeline by using the jenkinsfile from the repo.

The Jenkins pipefile:
1) configures helm and sets permission for tiller so that it can access resources of the k8s cluster.  
2) Deployes guestbook application through helm.
3) Deployes Prometheus helm chart in monitoring namespace . With its  kubernetes_sd_config, kubernetes service discovery      
configurtion, which is set in hel chart, it gets metrices from kubernetes-pods, kuberetes-servies, kube-kubelet,  etc. This  metrices can  checked by  curl <node-ip>:<prometheus-port>/metrics. 
4) to display the metrics in grafana, get  the prometheus server service IP and publish  it in datasource of grafana. ( create  a datasource in grafana and configure it with prometehus server service ip). Grafana will  pickup  all the  metrics from the  prometheus  instance(Node/Container/API).
5) sets up ELK stack with  helm deployment.
6) deploys a blue- green deploymnt.
   -  deploys tarefik ingress controller
   -  configures RBAC policies for it
   -  creates a ingress object which defines path for blue and green deploymnt
   -  creates a blue and green deploymnt and svc ( image used customized nginx, which displays green and blue pages on defined path ) 
   -  Curl to the http://<External IP of the Node/cluster> :<node-port-of traefik-ing>/green and see the green page
   -  Curl to the http://<External IP of the Node/cluster> :<node-port-of traefik-ing>/blue and see the blue page
7) deploys istio using helm chart
   -  deployes a hello-world application  to demonstrate canary deployment. It runs two versions of a simple helloworld service that return their version and instance (hostname) when called.
   -  dynamically injectes automatic sidecar injection  
   -  Uses CRD kind of Gateway and virtualservices 
   -  export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT (manual step)
   -  Enable autoscale on both services (set hpa)
   -  curl http://$GATEWAY_URL/hello
   -  generates the  load with loadgen.sh  script and check  the no. of replicas on versions of helloworld.


