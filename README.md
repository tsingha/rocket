# README.md

Hello! Welcome to my Rocket Chat Automation. This task is automatically deploying a Rocket Chat Application on EKS environment. The EKS environment will be deployed using **EKSCTL**. Rocket Chat Application deployment is a HA deployment, but MongoDB is standalone deployment. Also, here we are deploying **Network Load Balancer with the NGINX Ingress Controller** and **Ingress** to control the load and traffic. Also, we are using self-signed certificate for https secure access. For continuous deployment we are using **Jenkins**.

**Note:** As we are using HA for Rocket Chat Application, so it is very easy to upgrade/update the Rocket Chat Application with minimal downtime.

Before start create a Jump Server or Client Machine using Centos 7 and do the below task.

1.	**Install AWS CLI command utility**
    ```
    # curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

    # unzip awscliv2.zip

    # sudo ./aws/install

    # aws –version
    ```
2.	**Install EKSCTL command utility**
    ```
    # curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

    # sudo mv /tmp/eksctl /usr/local/bin

    # eksctl version
    ```
3.	**Install KUBECTL command utility**

    **Kubernetes 1.20:**
    ```
    # curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/linux/amd64/kubectl

    # chmod +x ./kubectl

    # mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin

    # echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

    # kubectl version --short –client
    ```
4.	**Install git command utility**
    ```
    # yum install git

    # git --version
    ```
5.	**Clone the repo**
    ```
    # git clone https://github.com/tsingha/rocket.git

    # cd rocket/
    ```
6.	**Configure AWS CLI**

    Install EKS cluster through **AWS CLI** using eksctl command, we need to create a **IAM user** using below policies.

    Go to Amazon Console -> IAM -> Users -> Add User -> User Name: eksctl -> Access Type: Click on Programmatic access and AWS Management Console access -> Console password:       Autogenerated password -> Click on Require password reset -> Click on Attach existing policies directly -> select the below policies

    - **AmazonEC2FullAccess**
    - **IAMUserChangePassword**
    - **SystemAdministrator**
    - **AdministratorAccess**
    - **AmazonEKSClusterPolicy**
    - **AmazonEKSWorkerNodePolicy**
    - **AmazonEKSServicePolicy**
    - **AmazonEKS_CNI_Policy**
    - **AmazonEKSFargatePodExecutionRolePolicy**
    - **AmazonEKSVPCResourceController**

    No need to give tags its optional -> Next -> Create User.

    After these steps, AWS will provide you a **Secret Access Key and Access Key ID. Save them preciously** because this will be the only time AWS gives it to you.

    **Run the below command**
    ```
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
    ```
    **Show list of all the IAM user**
    ```
    [root@docker ~]# aws iam list-users
    ```
    **Returns details about the IAM user or role whose credentials are used to call the operation.**
    ```
    [root@docker ~]# aws sts get-caller-identity
    {
        "UserId": "XXXXXXXXXX",
        "Account": "XXXXXXXXXX",
        "Arn": "arn:aws:iam::XXXXXXXXX:user/eksctl"
    }
    [root@docker ~]#
    ```
7.	**Install EKS Cluster using below command**
    ```
    # eksctl create cluster --name eks-poc-cluster-v1 --region us-east-2 --nodegroup-name eks-poc-node-group-v1 --node-type t3.small --nodes-min 2 --nodes-max 3 --managed
    ```
    Login to **AMAZON CONSOLE** with **IAM user** using which you are created the EKS cluster and you can able to see the cluster. 
 
    You can also verify the worker nodes using below command.
    ```
    # kubectl get nodes
    ```
8.	**Deploy Mongo DB application and service for mongodb**
    
    Deploy the pods using below commands.
    ```
    # cd rocket
    
    # kubectl create -f rocket-chat-mongo-db-svc.yaml

    # kubectl create -f rocket-chat-mongo-db.yaml

    # kubectl get svc rocket-chat-mongo-db-svc
    
    # kubectl get po
    ```
    **To login to the mongo db pod, run the below command.**
    ```
    # kubectl exec -it <POD Name> -- /bin/bash
    ```
    **Now run the command to login to mongo shell**
    ```
    # mongo
    ```
    **Rocket.Chat uses the MongoDB replica set to improve performance via Meteor Oplog tailing. To init replica set, run this function in the mongo shell**
    
    > **rs.initiate()**

    It will show the below output
    ```
    {
            "info2" : "no configuration specified. Using a default configuration for the set",
            "me" : "rocket-chat-mongo-db-7d6f8b758-6bqmv:27017",
            "ok" : 1
    }
    rs0:SECONDARY>
    ```
    Hit enter, you should see your prompt turn into rs0:PRIMARY>, this indicates the replica set is being used. Type exit to get back to your regular shell:
    ```
    rs0:PRIMARY>exit
    bye
    ```
    Now login with below url to check the connectivity. Here we are using service rocket-chat-mongo-db-svc to connect mongodb database.
    ```
    # mongo mongodb://rocket-chat-mongo-db-svc:27017
    ```
    Above command will help you to login monogdb shell and run the below command to check the databases.
    ```
    rs0:PRIMARY> show dbs
    admin   0.000GB
    config  0.000GB
    local   0.000GB
    rs0:PRIMARY>
    ```
9.	**Deploy Rocket chat application service**
    ```
    # kubectl create -f rocket-svc.yaml

    # kubectl get svc rocket-chat-server
    ```
10.	**Deploy Network Load Balancer with the NGINX Ingress Controller**
    In AWS we use a Network load balancer (NLB) to expose the NGINX Ingress controller behind a Service of Type=LoadBalancer.
    ```
    # kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml

    # kubectl get all -n ingress-nginx
    ```
11.	**Create Ingress Resources**

    Run the below command and get the External IP
    ```
    # kubectl get svc -n ingress-nginx
    ```
    **External IP like** : a595a45XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.elb.us-east-2.amazonaws.com

    **Note**: If you’ve purchased and configured a custom domain name for your server, you can use that certificate, otherwise you can still use self-signed certificate for                  development and testing.

    Here for create **TLS secret**, we are creating below **self-signed certificate** using below command. **(Don’t run this certificate creation command because certificate already available with this project, if not then create the certificate)**
    ```
    # openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt
    ```
    **Create the secret in the cluster with below command.**
    ```
    # kubectl create secret tls tls-secret --key cert/tls.key --cert cert/tls.crt
    
    # kubectl get secrets tls-secret
    ```
    Now edit the **ingress.yaml** with the **secret name tls-secret** and **host name with External IP**, and then run the below command
    ```
    # kubectl create -f ingress.yaml
    
    # kubectl get ingress rocket-chat-ingress
    ```
12.	**Setup Continuous Deployment (Only Deploying Rocket Chat Server)**

    **Install Jenkins:**
    ```
    # wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    # rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
    # yum upgrade
    # yum install jenkins java-11-openjdk-devel
    # systemctl daemon-reload
    ```
    **Jenkins URL:**
    > **http://<FQDN/IP>:8080**
    
    Now do the below work,

    1.	**Allow the port 8080 in the EKS cluster security group to make the connectivity between jenkins server and EKS cluster nodes.**

    2.	**Enable the Git, Github, Kubernetes, AWS and Docker related plugins.**

        Go to Manage Jenkins -> Manage Plugins -> Click on Available -> Select the related plugins -> Click on Install without restart.

    3.	**Copy the AWS CLI credentials and kube config file in /var/lib/Jenkins, so jenkins user can able to use aws and kubectl command.**
        ```
        # cp .aws/ /var/lib/jenkins/
        # cp .kube/ /var/lib/jenkins/
        # chown -R jenkins:jenkins /var/lib/jenkins/
        # which aws
        # ln -s /usr/local/bin/aws /usr/bin/aws
        # which kubectl
        # ln -s /usr/local/bin/kubectl /usr/bin/kubectl
        # which eksctl
        # ln -s /usr/local/bin/eksctl /usr/bin/eksctl
        ```
    4.	**Now create a freestyle project for Rocket Chat Server Installation.**

        Go to new Item -> Select Freestyle Project -> Provide a name -> OK

        Now Open the project and Provide a description.
        > **Rocket Chat Server Installation**

        In **Source Code Management** provide the **git repository URL**.

        > **https://github.com/tsingha/rocket.git**

        In **Build** section choose the **Execute shell** and write down the command and click on **Save**.
        ```
        kubectl apply -f rocket.yaml
        ```
        
        Now go to the project and click on **“Build Now”** to deploy.

        Once deployment done through Jenkins, run the below command in CLI to cross check the deployment
        ```
        # kubectl get po
        
        # kubectl logs -f <rocket server POD Name>
        ```
    Now you can open a browser and type **https://< External IP or Public DNS>** to access the Rocket Chat Application.

13.	**Delete Environment**

    Do the following task to clean up all the Environment.

    1.	**Now create a freestyle project to delete all Rocket Chat related resources.**

        Go to new Item -> Select Freestyle Project -> Provide a name -> OK

        Now Open the project and Provide a description.
        > **Rocket Chat Resource Deletion**

        In **Source Code Management** provide the **git repository URL**.

        > **https://github.com/tsingha/rocket.git**

        In **Build** section choose the **Execute shell** and write down the command and click on **Save**.
        ```
        kubectl delete -f rocket.yaml
        kubectl delete -f rocket-svc.yaml
        kubectl delete -f ingress.yaml
        kubectl delete secret tls-secret
        kubectl delete -f rocket-chat-mongo-db-svc.yaml
        kubectl delete -f rocket-chat-mongo-db.yaml
        kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy.yaml
        ```
        Now go to the project and click on **“Build Now”** to deploy.

        Once the resources deletion done through Jenkins, you can check from CLI and you wont get any resources related to Rocket Chat.

    2.	Delete the port 8080 (for jenkins) from the EKS group security group. **(It is optional because in next step all the resources will be deleted)**

        Go to the Amazon console and then security group to do the task.

    3.	**Now delete the EKS cluster using the below command.**
        ```
        # eksctl delete cluster --name eks-poc-cluster-v1 --region us-east-2
        ```
        It will take some time to delete all the EKS resources. After some time if you go to the AWS console, EKS cluster won't showing. That means all the resources belongs to         EKS cluster along with EKS cluster has been deleted successfully.

    4.	Now you can delete your client server where you installed aws cli, eksctl, kubectl utility and setup Jenkins server.
