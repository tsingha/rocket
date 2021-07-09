  Rocket Chat Application Run.

1.	Install AWS CLI command utility

# curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# unzip awscliv2.zip

# sudo ./aws/install

# aws –version

2.	Install EKSCTL command utility

# curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# sudo mv /tmp/eksctl /usr/local/bin

# eksctl version

3.	Install KUBECTL command utility

Kubernetes 1.20:

# curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/linux/amd64/kubectl

# chmod +x ./kubectl

# mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin

# echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

# kubectl version --short –client

4.	Install HELM command utility

# curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh

# chmod 700 get_helm.sh

# ./get_helm.sh

# helm version

5.	Configure AWS CLI

Install EKS cluster through AWS CLI using eksctl command, we need to create a IAM user using below policies.

Go to Amazon Console -> IAM -> Users -> Add User -> User Name: eksctl -> Access Type: Click on Programmatic access and AWS Management Console access -> Console password: Autogenerated password -> Click on Require password reset -> Click on Attach existing policies directly -> select the below policies

AmazonEC2FullAccess
IAMUserChangePassword
SystemAdministrator
AdministratorAccess
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AmazonEKSServicePolicy
AmazonEKS_CNI_Policy
AmazonEKSFargatePodExecutionRolePolicy
AmazonEKSVPCResourceController

No need to give tags its optional -> Next -> Create User.

After these steps, AWS will provide you a Secret Access Key and Access Key ID. Save them preciously because this will be the only time AWS gives it to you.

Run the below command

[root@docker ~]# aws configure
AWS Access Key ID [None]: <Provide the Access Key ID for your IAM user>
AWS Secret Access Key [None]: <Provide the Secret Access Key for your IAM user>
Default region name [None]:
Default output format [None]:
[root@docker ~]#

[root@docker ~]# cat .aws/credentials
[default]
aws_access_key_id = XXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXX
[root@docker ~]#

Show list of all the IAM user
[root@docker ~]# aws iam list-users

Returns details about the IAM user or role whose credentials are used to call the operation.
[root@docker ~]# aws sts get-caller-identity
{
    "UserId": "XXXXXXXXXX",
    "Account": "XXXXXXXXXX",
    "Arn": "arn:aws:iam::XXXXXXXXX:user/eksctl"
}
[root@docker ~]#

6.	Install EKS Cluster

[root@docker ~]# eksctl create cluster --name eks-poc-cluster-v1 --region us-east-2 --nodegroup-name eks-poc-node-group-v1 --node-type t3.small --nodes-min 2 --nodes-max 3 --managed
2021-07-08 12:35:20 [ℹ]  eksctl version 0.55.0
2021-07-08 12:35:20 [ℹ]  using region us-east-2
2021-07-08 12:35:22 [ℹ]  setting availability zones to [us-east-2a us-east-2c us-east-2b]
2021-07-08 12:35:22 [ℹ]  subnets for us-east-2a - public:192.168.0.0/19 private:192.168.96.0/19
2021-07-08 12:35:22 [ℹ]  subnets for us-east-2c - public:192.168.32.0/19 private:192.168.128.0/19
2021-07-08 12:35:22 [ℹ]  subnets for us-east-2b - public:192.168.64.0/19 private:192.168.160.0/19
2021-07-08 12:35:22 [ℹ]  nodegroup "eks-poc-node-group-v1" will use "" [AmazonLinux2/1.19]
2021-07-08 12:35:22 [ℹ]  using Kubernetes version 1.19.
--------------
--------------
--------------
2021-07-08 12:41:33 [ℹ]  waiting for CloudFormation stack "eksctl-eks-poc-cluster-v1-cluster"
--------------
--------------
--------------
2021-07-08 12:53:47 [ℹ]  waiting for CloudFormation stack "eksctl-eks-poc-cluster-v1-nodegroup-eks-poc-node-group-v1"
2021-07-08 12:53:49 [ℹ]  waiting for the control plane availability...
2021-07-08 12:53:49 [✔]  saved kubeconfig as "/root/.kube/config"
--------------
--------------
--------------
2021-07-08 12:53:51 [ℹ]  node "ip-192-168-13-43.us-east-2.compute.internal" is ready
2021-07-08 12:53:51 [ℹ]  node "ip-192-168-69-115.us-east-2.compute.internal" is ready
2021-07-08 12:56:00 [ℹ]  kubectl command should work with "/root/.kube/config", try 'kubectl get nodes'
2021-07-08 12:56:00 [✔]  EKS cluster "eks-poc-cluster-v1" in "us-east-2" region is ready
[root@docker ~]#

Login to AMAZON CONSOLE with IAM user using which you are created the EKS cluster and see the below outputs. 
 
 
 

[root@docker ~]# kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-13-43.us-east-2.compute.internal    Ready    <none>   21m   v1.19.6-eks-49a6c0
ip-192-168-69-115.us-east-2.compute.internal   Ready    <none>   21m   v1.19.6-eks-49a6c0
[root@docker ~]#

7.	Deploy Mongo DB application and service for mongodb

# kubectl create -f rocket-chat-mongo-db-svc.yaml

# kubectl create -f rocket-chat-mongo-db.yaml

[root@docker rocket]# kubectl get svc rocket-chat-mongo-db-svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
rocket-chat-mongo-db-svc   ClusterIP   10.100.125.254   <none>        27017/TCP   19m
[root@docker rocket]#

[root@docker rocket]# kubectl get po
NAME                                   READY   STATUS    RESTARTS   AGE
rocket-chat-mongo-db-7d6f8b758-6bqmv   1/1     Running   0          2m30s
[root@docker rocket]#

To login to the mongo db pod, run the below command.

# kubectl exec -it rocket-chat-mongo-db-7d6f8b758-6bqmv -- /bin/bash

Now run the command to login to mongo shell: # mongo

Rocket.Chat uses the MongoDB replica set to improve performance via Meteor Oplog tailing. To init replica set, run this function in the mongo shell
> rs.initiate()

It will show the below output
{
        "info2" : "no configuration specified. Using a default configuration for the set",
        "me" : "rocket-chat-mongo-db-7d6f8b758-6bqmv:27017",
        "ok" : 1
}
rs0:SECONDARY>

Hit enter, you should see your prompt turn into rs0:PRIMARY>, this indicates the replica set is being used. Type exit to get back to your regular shell:
rs0:PRIMARY>exit
bye

Now login with below url. Here we are using service rocket-chat-mongo-db-svc to connect mongodb database.

# mongo mongodb://rocket-chat-mongo-db-svc:27017

Above command will help you to login monogdb shell and run the below command to check the databases.
rs0:PRIMARY> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
rs0:PRIMARY>

8.	Deploy Rocket chat application service

# kubectl create -f rocket-svc.yaml

[root@docker rocket]# kubectl get svc rocket-chat-server
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
rocket-chat-server   ClusterIP   10.100.16.250   <none>        3000/TCP   31s
[root@docker rocket]#

9.	Deploy Network Load Balancer with the NGINX Ingress Controller

# kubectl apply -f deploy.yaml

[root@docker rocket]# kubectl get all -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-pt227        0/1     Completed   0          15m
pod/ingress-nginx-admission-patch-b57sj         0/1     Completed   0          15m
pod/ingress-nginx-controller-57cb5bf694-plqn7   1/1     Running     0          15m

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.100.170.241   a595a4579b2eb43d4b4cfd3c3ed815ed-0cc9070e29794210.elb.us-east-2.amazonaws.com   80:30351/TCP,443:30050/TCP   15m
service/ingress-nginx-controller-admission   ClusterIP      10.100.176.93    <none>                                                                          443/TCP                      15m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           15m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-57cb5bf694   1         1         1       15m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           3s         15m
job.batch/ingress-nginx-admission-patch    1/1           3s         15m
[root@docker rocket]#

10.	Create Ingress Resources

Run the below command and get the External IP

[root@docker rocket]# kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.170.241   a595a4579b2eb43d4b4cfd3c3ed815ed-0cc9070e29794210.elb.us-east-2.amazonaws.com   80:30351/TCP,443:30050/TCP   21m
ingress-nginx-controller-admission   ClusterIP      10.100.176.93    <none>                                                                          443/TCP                      21m
[root@docker rocket]#

External IP like : a595a4579b2eb43d4b4cfd3c3ed815ed-0cc9070e29794210.elb.us-east-2.amazonaws.com

If you’ve purchased and configured a custom domain name for your server, you can use that certificate, otherwise you can still use self-signed certificate for development and testing.

Here for create TLS secret, we are creating below self-signed certificate using below command. (Don’t run this certificate creation command because certificate already available with this project, if not the create the certificate)

# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt

Then create the secret in the cluster with below command:

# kubectl create secret tls tls-secret --key tls.key --cert tls.crt

To get the secret details run the below command:

# kubectl get secrets tls-secret
Now edit the ingress.yaml with the secret name tls-secret and host name with External IP, and then run the below command
# kubectl create -f ingress.yaml
# kubectl get ingress rocket-chat-ingress
11.	Setup Continuous Deployment

Install Jenkins
# wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
# rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
# yum upgrade
# yum install jenkins java-11-openjdk-devel
# systemctl daemon-reload
Jenkins URL: http://<FQDN/IP>:8080/

Once deployment done through Jenkins, run the below command to cross check the deployment

# kubectl logs -f <POD Name>

 

