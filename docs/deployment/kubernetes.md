---
layout: default
deployDoc: true
---

## Sock Shop on Kubernetes + Weave 

This demo demonstrates running the Sock Shop on a Kubernetes cluster using 
Weave Net and Weave Scope.

### Pre-requisites
* *Optional* [Terraform](https://www.terraform.io/downloads.html)
* *Optional* [AWS Account](https://aws.amazon.com/)
* *Optional* [awscli](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

<!-- deploy-test require-env AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION TRAVIS_BRANCH TRAVIS_COMMIT -->
<!-- deploy-test-start pre-install -->

    apt-get install -yq jq python-pip curl unzip git
    pip install awscli
    curl -o /tmp/terraform.zip https://releases.hashicorp.com/terraform/0.7.11/terraform_0.7.11_linux_amd64.zip 
    unzip /tmp/terraform.zip -d /usr/bin
    git clone https://github.com/microservices-demo/microservices-demo 

<!-- deploy-test-end -->

<!-- deploy-test-hidden pre-install 
    git clone --depth=50 --branch=$TRAVIS_BRANCH https://github.com/microservices-demo/microservices-demo 
    cd microservices-demo
    git checkout -qf $TRAVIS_COMMIT

    cat > /root/healthcheck.sh <<-EOF
#!/usr/bin/env bash
kubectl run -\-namespace=sock-shop healthcheck -\-image=andrius/alpine-ruby sleep 10000
sleep 120
kube_id=\$(kubectl get pods -\-namespace=sock-shop | grep healthcheck | awk '{print \$1}')
kubectl exec -\-namespace=sock-shop \$kube_id -\- sh -c "curl -o healthcheck.rb \"https://raw.githubusercontent.com/microservices-demo/microservices-demo/master/deploy/healthcheck.rb\"; chmod +x ./healthcheck.rb; ./healthcheck.rb -s user,catalogue,queue-master,cart,shipping,payment,orders"

EOF

    mkdir -p ~/.ssh/
    aws ec2 describe-key-pairs -\-key-name deploy-docs-k8s &>/dev/null
    if [ $? -eq 0 ]; then aws ec2 delete-key-pair -\-key-name deploy-docs-k8s; fi
-->
### Setup Kubernetes

<small>If you already have running k8s cluster skip to [here](#weavenet)</small>

We'll create a private key for use during this demo.    
<small>If you already have a private key on AWS you can skip this step.</small>

Begin by setting the appropriate AWS environment variables.
```
export AWS_ACCESS_KEY_ID=[YOURACCESSKEYID]
export AWS_SECRET_ACCESS_KEY=[YOURSECRETACCESSKEY]
export AWS_DEFAULT_REGION=[YOURDEFAULTREGION]
```
<!-- deploy-test-start create-infrastructure -->

    aws ec2 create-key-pair --key-name deploy-docs-k8s --query 'KeyMaterial' --output text > ~/.ssh/deploy-docs-k8s.pem
    chmod 600 ~/.ssh/deploy-docs-k8s.pem

<!-- deploy-test-end -->

Next we'll set some terraform variables such as our key name and where it's located, along with the instance sizes for the master and node instances.  
Finally run terraform.

<!-- deploy-test-start create-infrastructure -->

    cd microservices-demo/deploy/kubernetes/terraform
    terraform apply

<!-- deploy-test-end -->


Run `terraform output` to get the public endpoints of all the instances and load balancers.
```
Outputs:

master_address = ec2-52-49-12-162.eu-west-1.compute.amazonaws.com
node_addresses = [
    ec2-52-212-222-97.eu-west-1.compute.amazonaws.com,
    ec2-52-51-218-57.eu-west-1.compute.amazonaws.com,
    ec2-52-18-214-169.eu-west-1.compute.amazonaws.com
]
scope_address = elb-scope-927708071.eu-west-1.elb.amazonaws.com
sock_shop_address = elb-sock-shop-363710645.eu-west-1.elb.amazonaws.com
```

Our master node makes use of some of the files in this repo so lets securely copy those over.
```
scp -i <path-to-private-key> -rp microservices-demo/deploy/kubernetes ubuntu@<master-ip>:/tmp/
```
<!-- deploy-test-hidden create-infrastructure

    cd microservices-demo/deploy/kubernetes/terraform
    export PRIVATE_KEY_PATH="~/.ssh/deploy-docs-k8s.pem"

    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    node_addresses=$(terraform output -json | jq -r '.node_addresses.value|@sh' | sed -e "s/'//g" | tr ' ' '\n') 
    cd ..

    echo "Setting up kubernetes..."
    ssh -i $PRIVATE_KEY_PATH -o StrictHostKeyChecking=no ubuntu@$master_ip sudo kubeadm init > /root/k8s-init.log
    grep -e -\-token /root/k8s-init.log > /root/join.cmd

    echo "Setting up Weave Net.."
    ssh -i $PRIVATE_KEY_PATH ubuntu@$master_ip kubectl apply -f https://git.io/weave-kube
    echo "Setting up Weave Scope.."
    scp -i $PRIVATE_KEY_PATH -rp /repo/deploy/kubernetes/ ubuntu@$master_ip:/home/ubuntu
    ssh -i $PRIVATE_KEY_PATH ubuntu@$master_ip kubectl apply -f /home/ubuntu/kubernetes/weavescope.yaml -\-validate=false

    echo "Nodes joining kubernetes..."
    for node in $node_addresses; do
        ssh -i $PRIVATE_KEY_PATH -o StrictHostKeyChecking=no ubuntu@$node sudo `cat /root/join.cmd`
    done

    ssh -i $PRIVATE_KEY_PATH ubuntu@$master_ip kubectl apply -f /home/ubuntu/kubernetes/manifests/sock-shop-ns.yml -f /home/ubuntu/kubernetes/manifests

-->


### <a name="weavenet"></a>Setup Weave Net
* ssh into the master node
* Run the following commands:

```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<master-ip>
root@23590a8d99:/# sudo kubeadm init
<master/tokens> generated token: "f0c861.753c505740ecde4c"
... more output ...
You can connect any number of nodes by running:

kubeadm join --token <token> <master-ip>

root@23590a8d99:/# kubectl apply -f https://git.io/weave-kube
```

* make note of the ```kubeadm join --token <token> <master-ip>``` command after the ```kubeadm init``` command runs. You'll need it later to join the nodes to the master. 

### Time for the nodes to join the master 
* SSH into each node_addresses listed when you ran the ```terraform output``` command and run the ```kubeadm join --token <token> <master-ip>``` command from before.

```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<node-ip>
ubuntu@<node-ip>:/# sudo kubeadm join --token <token> <master-ip>
```

### Setup Weave Scope
* SSH into the master node
* Start weave scope on the cluster

```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<node-ip>
ubuntu@<master-ip>:/# kubectl apply -f /tmp/weavescope.yaml -\-validate=false
```

### Deploy Sock Shop
* SSH into the master node
* Deploy the sock shop

```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<master-ip>
ubuntu@<master-ip>:/# kubectl apply -f /tmp/manifests/sock-shop-ns.yml -f /tmp/manifests
```

### View results
Each app is behind a load balancer which delivers the app from one of 3+ nodes

Run `terraform output` again to see the load balancer URLs
```
Outputs:

master_address = ec2-52-49-12-162.eu-west-1.compute.amazonaws.com
node_addresses = [
    ec2-52-212-222-97.eu-west-1.compute.amazonaws.com,
    ec2-52-51-218-57.eu-west-1.compute.amazonaws.com,
    ec2-52-18-214-169.eu-west-1.compute.amazonaws.com
]
scope_address = elb-scope-927708071.eu-west-1.elb.amazonaws.com
sock_shop_address = elb-sock-shop-363710645.eu-west-1.elb.amazonaws.com
```

Open any of the links listed in scope_address and sock_shop_address to see the apps in action.   
It may take a few moments for the apps to get running.

### Run tests

There is a seperate load-test available to simulate user traffic to the application. For more information see [Load Test](#loadtest).  
This will send some traffic to the application, which will form the connection graph that you can view in Scope or Weave Cloud. 

<!-- deploy-test-start run-tests -->

    cd microservices-demo/deploy/kubernetes/terraform
    elb_url = $(terraform output -json | jq -r '.sock_shop_address.value') 
    docker run --rm weaveworksdemos/load-test -d 120 -h $elb_url -c 3 -r 10

<!-- deploy-test-end -->

<!-- deploy-test-hidden run-tests

    cd microservices-demo/deploy/kubernetes/terraform
    export PRIVATE_KEY_PATH="~/.ssh/deploy-docs-k8s.pem"
    master_ip=$(terraform output -json | jq -r '.master_address.value') 
    scp -i $PRIVATE_KEY_PATH -rp /root/healthcheck.sh ubuntu@$master_ip:/home/ubuntu
    ssh -i $PRIVATE_KEY_PATH ubuntu@$master_ip "chmod +x /home/ubuntu/healthcheck.sh; ./healthcheck.sh"

    if [ $? -ne 0 ]; then
        exit 1;
    fi

-->

### Uninstall App

Remove all deployments (will also remove pods)
```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<master-ip>
ubuntu@<master-ip>:/# kubectl delete deployments --all
```
Remove all services, except kubernetes
```
root@23590a8d99:/# ssh -i <path-to-private-key> ubuntu@<master-ip>
ubuntu@<master-ip>:/# kubectl delete service $(kubectl get services | cut -d" " -f1 | grep -v NAME | grep -v kubernetes)
```

Destroying the entire infrastructure
<!-- deploy-test-start destroy-infrastructure -->

    cd /repo/deploy/kubernetes/terraform
    terraform destroy -force
    aws ec2 delete-key-pair -\-key-name deploy-docs-k8s

<!-- deploy-test-end -->
