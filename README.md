# Epinio Project

This project is divided into three parts.
1. Create the EKS Cluster and install Epinio
2. Deploying a simple application that uses postgreSQL as a backend
3. Deleting the Lab

## Prerequisites

- kubectl
- helm 
- aws cli
- eksctl

# 1 - Create the EKS Cluster and install Epinio

### Step-01: Create EKS Cluster using eksctl

```
# Create Cluster
eksctl create cluster --name=ekslab1 \
                      --version=1.24 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 

# Get List of clusters
eksctl get cluster                  
```

### Step-02: Create & Associate IAM OIDC Provider for our EKS Cluster
- To enable and use AWS IAM roles for Kubernetes service accounts on our EKS cluster, we must create &  associate OIDC identity provider.

```                   
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster ekslab1 \
    --approve
```

### Step-03: Create EKS Node Group in Private Subnets

```
create nodegroup --cluster=ekslab1 \
                        --region=us-east-1 \
                        --name=ekslab1-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=MacKey \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking 

# Get List of nodes
kubectl get nodes                         
```
### Step-04: Create EBS CSI Drive

- Since EKS v1.23 it is necessary to configure and install an out-of-tree AWS EBS CSI driver as an addon into your EKS cluster. Please refer to this EKS documentation for more details.

Create the policy to attach in our nodes:
```
aws iam create-policy --policy-name Amazon_EBS_CSI_Driver --policy-document file://Amazon_EBS_CSI_Driver.json
```

- From output check arn:
```
"Arn": "arn:aws:iam::410334805876:policy/Amazon_EBS_CSI_Driver"
```

- Get Worker node IAM Role ARN
```
kubectl -n kube-system describe configmap aws-auth
```

- From output check rolearn
```
rolearn: arn:aws:iam::410334805876:role/eksctl-eksepinio1-nodegroup-eksep-NodeInstanceRole-190V592LWZLCU

# In this case the name of role is:
eksctl-eksepinio1-nodegroup-eksep-NodeInstanceRole-190V592LWZLCU
```

- Associate the policy to that role

```
aws iam attach-role-policy --policy-arn arn:aws:iam::410334805876:policy/Amazon_EBS_CSI_Driver --role-name eksctl-eksepinio1-nodegroup-eksep-NodeInstanceRole-190V592LWZLCU
```
### Deploy Amazon EBS CSI Driver  

- Verify kubectl version, it should be 1.14 or later
```
kubectl version --client --short
```
- Deploy Amazon EBS CSI Driver

```
# Deploy EBS CSI Driver

kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"

# Verify ebs-csi pods running
kubectl get pods -n kube-system
```

### Step-05: Allow the pulling of Epinio's app container images from its internal HTTP registry

- Since EKS v1.24 it is necessary to explicitly allow the pulling of Epinio's app container images from its internal HTTP registry, due to the removal of dockershim CRI support and its replacement by containerd, which supports only trusted HTTPS registries by default.

```
kubectl apply -f 01-Epinio-Images/
```

### Step-06: Install Cert-Manager

```
helm repo add cert-manager https://charts.jetstack.io
helm repo update
helm install cert-manager --namespace cert-manager --create-namespace jetstack/cert-manager --set installCRDs=true --set extraArgs={--enable-certificate-owner-ref=true}
```

### Step-07: Install Nginx Ingress Controller

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install nginx ingress-nginx/ingress-nginx --namespace nginx --create-namespace --set controller.ingressClassResource.default=true
```

### Step-08: Create a CNAME DNS entry pointing to ELB endpoint

```
kubectl get svc -n nginx nginx-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Output Example:
```
a414dcc40c6b9420cb92xxxxxxxxxxxxx-1300xxxxxxx.us-east-1.elb.amazonaws.com
```

Use that ELB endpoint value when creating the CNAME record for your DNS zone (for eg. in AWS Route53 service):
```
Record name: *.example.com
Type: CNAME
Value: a414dcc40c6b9420cb92xxxxxxxxxxxxx-1300xxxxxxx.us-east-1.elb.amazonaws.com
```
### If you use Route53, the following commands can be used to create the CNAME record

List Zones:
```
aws route53 list-hosted-zones
```

Output Example:
```
{
    "HostedZones": [
        {
            "Id": "/hostedzone/Z040811XXXXXXXXXX",
            "Name": "example.com.",
            "CallerReference": "RISWorkflow-RD:xxxxxxx-xxxxxx-xxxxxx-xxxxx",
            "Config": {
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 5
        }
    ]
}
```

Create CNAME command:
```
aws route53 change-resource-record-sets --hosted-zone-id <YOUR_ZONE_ID> --change-batch '{"Changes": [{"Action": "CREATE", "ResourceRecordSet": {"Name": "*.example.com", "Type": "CNAME", "TTL": 300, "ResourceRecords": [{"Value": "a414dcc40c6b9420cb92xxxxxxxxxxxxx-1300xxxxxxx.us-east-1.elb.amazonaws.com"}]}}]}'
```

### Step-09: Install Epinio on the Cluster
```
helm upgrade --install epinio epinio/epinio --namespace epinio --create-namespace --set global.domain=example.com --set global.tlsIssuer=letsencrypt-production --set global.tlsIssuerEmail=email@example.com
```


# 2 - Deploying a simple application that uses postgreSQL as a backend

I used the following application as an example:

https://github.com/epinio/example-rails

The first step of this documentation requires rails to be installed to create the master.key:
```
EDITOR=vi rails credentials:edit
```

### To overcome this step it is necessary to have ruby 3.2.2 and rails installed and configured in your operating system.

Here is a how to for CentOS 9:

- Enable EPEL Release:
```
sudo dnf config-manager --set-enabled crb
sudo dnf install epel-release epel-next-release
```

- Install Dependencies
```
sudo yum install gcc make git-core zlib zlib-devel gcc-c++ patch readline readline-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison curl sqlite-devel -y
```

- Install Node.js to make use of Ruby on Rails LTS
```
curl -sL https://rpm.nodesource.com/setup_14.x | sudo bash -
sudo dnf install -y nodejs
```

- Install Yarn Package required by Ruby on Rails source components.
```
curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
sudo dnf install -y yarn
```

- Install rbenv
```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL

git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL

Verify that rbenv is set up correctly.
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/main/bin/rbenv-doctor | bash
```

- Install ruby
```
sudo rbenv install 3.2.2
```

- Install bundles
```
sudo gem install bundler
```

- Install rails
```
sudo gem install rails
```

- Clone the project 

```
git clone https://github.com/epinio/example-rails.git
```

- Install the bundle
```
# Dependences:
sudo dnf install postgresql-devel
sudo gem install pg

cd example-rails

sudo bundle install
```

### Install Epinio CLI for CentOS 9
```
sudo wget https://github.com/epinio/epinio/releases/download/v1.8.1/epinio-linux-arm64
sudo mv epinio-linux-arm64 epinio
sudo chmod +x epinio
sudo mv ./epinio /usr/local/bin/epinio
epinio version

epinio login -u admin https://yourcluster.yourdomain.com

```
From this point you can follow the documentation described in https://github.com/epinio/example-rails

# 3 - Deleting the Lab

### Step-01: Uninstall the packages

```
helm uninstall -n epinio epinio
helm uninstall -n nginx nginx
helm uninstall -n cert-manager cert-manager
kubectl delete deployment ebs-csi-controller -n kube-system
```
### Step-02: Remove IAM policy the packages
```
aws iam detach-role-policy --policy-arn arn:aws:iam::410334805876:policy/Amazon_EBS_CSI_Driver --role-name 
eksctl-eksepinio1-nodegroup-eksde-NodeInstanceRole-8EBAJEZX667H

aws iam delete-policy --policy-arn arn:aws:iam::410334805876:policy/Amazon_EBS_CSI_Driver
```

## Step-03: Delete the Node Group

### List EKS Clusters
```
eksctl get clusters
```

### Capture Node Group name
```
eksctl get nodegroup --cluster=<clusterName>
eksctl get nodegroup --cluster=ekslab1
```

### Delete Node Group
```
eksctl delete nodegroup --cluster=<clusterName> --name=<nodegroupName>
eksctl delete nodegroup --cluster=ekslab1 --name=ekslab1-ng-private1
```

## Step-04: Delete the Cluster
### Delete Cluster
```
eksctl delete cluster <clusterName>
eksctl delete cluster ekslab11
```
