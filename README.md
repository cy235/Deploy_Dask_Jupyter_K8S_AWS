# Deploy Dask and Jupyter into kubernetes cluster in Amazon Web Services
In this project, I will deploy dask and jupyter into kubernetes (k8s) cluster in Amazon Web Services (AWS). Dask is a flexible library for parallel computing in Python. Dask is composed of two parts: ... “Big Data” collections like parallel arrays, dataframes, and lists that extend common interfaces like NumPy, Pandas, or Python iterators to larger-than-memory or distributed environments.

## Set Up a Kubernetes Cluster in AWS
### Prerequisites
Before setting up the Kubernetes cluster, you’ll need an AWS account and an installation of the AWS Command Line Interface.

Make sure to configure the AWS CLI to use your access key ID and secret access key:
```
$ aws configure
AWS Access Key ID [None]: AWS_key_ID
AWS Secret Access Key [None]: AWS_secret_key
Default region name [None]: us-east-1
Default output format [None]: json
```
### Install kops + kubectl
On Mac OS X, we’ll use brew to install.
```
$ brew update && brew install kops kubectl
```

### Set Up the Kubernetes Cluster
The first thing we need to do is create an S3 bucket for kops to use to store the state of the Kubernetes cluster and its configuration. We’ll use the bucket name cy235-kops-state-store
```
$ aws s3api create-bucket --bucket cy235-kops-state-store --region us-east-1
```
Before creating the cluster, let’s set two environment variables: `KOPS_CLUSTER_NAME` and `KOPS_STATE_STORE`. For safe keeping you should add the following to your `~/.bash_profile` or `~/.bashrc` configs (or whatever the equivalent is if you don’t use bash).
```
$ export KOPS_CLUSTER_NAME=cy235.k8s.local
$ export KOPS_STATE_STORE=s3://cy235-kops-state-store
```
Now, to generate the cluster configuration:
```
$ kops create cluster --node-count=2 --node-size=t2.medium --zones=us-east-1a
```
you may encounter:
```
$ kops create cluster --node-count=2 --node-size=t2.medium --zones=us-east-1a
I0318 22:59:06.019860   23964 create_cluster.go:562] Inferred --cloud=aws from zone "us-east-1a"
I0318 22:59:06.162553   23964 subnets.go:184] Assigned CIDR 172.20.32.0/19 to subnet us-east-1a
Previewing changes that will be made:


SSH public key must be specified when running with AWS (create with `kops create secret --name cy235.k8s.local sshpublickey admin -i ~/.ssh/id_rsa.pub`)
```
In order solve above problem, I execute the following commands:
```
$ ssh-keygen -t rsa -f ./cluster.fayzlab.com
$ kops create secret sshpublickey admin -i ~/.ssh/cluster.cy235.com.pub  --state s3://cy235-kops-state-store
```
Note: this line doesn’t launch the AWS EC2 instances. It simply creates the configuration and writes to the `s3://cy235-kops-state-store` bucket we created above. In our example, we’re creating 2 t2.medium EC2 work nodes in addition to a c4.large master instance (default).
```
$ kops edit cluster
```
Now that we’ve generated a cluster configuration, we can edit its description before launching the instances. The config is loaded from `s3://cy235-kops-state-store`. You can change the editor used to edit the config by setting `$EDITOR` or `$KUBE_EDITOR`. For instance, in my `~/.bashrc`, I have export `KUBE_EDITOR=cy235`.

Time to build the cluster. This takes a few minutes to boot the EC2 instances and download the Kubernetes components.
```
$ kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```
After waiting a bit (5~10 minutes), let’s validate the cluster to ensure the master + 2 nodes have launched.
```
$ kops validate cluster
Using cluster from kubectl context: cy235.k8s.local

Validating cluster cy235.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-east-1a	Master	m3.medium	1	1	us-east-1a
nodes			Node	t2.medium	2	2	us-east-1a

NODE STATUS
NAME				ROLE	READY
ip-172-20-33-220.ec2.internal	master	True
ip-172-20-49-54.ec2.internal	node	True
ip-172-20-56-3.ec2.internal	node	True

Your cluster cy235.k8s.local is ready
```
Note: If you ignore the message Cluster is starting. It should be ready in a few minutes. and validate too early, you’ll get an error. Wait a little longer for the nodes to launch, and the validate step will return without error.

Finally, you can see your Kubernetes nodes with kubectl:
```
$ kubectl get nodes
NAME                            STATUS   ROLES    AGE   VERSION
ip-172-20-33-220.ec2.internal   Ready    master   32m   v1.16.7
ip-172-20-49-54.ec2.internal    Ready    node     31m   v1.16.7
ip-172-20-56-3.ec2.internal     Ready    node     31m   v1.16.7
```

When cluster is created, we can login the AWS to EC2s, Auto scaling groups and Load balancers in the following:

EC2s
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/EC2.jpg)

Auto scaling groups
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/auto_scaling_group.jpg)

Load balancers
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/load_balancer.jpg)

### Kubernetes Dashboard
Now, we have a working Kubernetes cluster deployed on AWS. At this point, we can deploy lots of applications, such as Dask and Jupyter. For demonstration, I will launch the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard). Think UI instead of command line for managing Kubernetes clusters and applications.

To deploy Dashboard, execute following command:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
```
To access Dashboard from your local workstation you must create a secure channel to your Kubernetes cluster. Run the following command:
```
$ kubectl proxy
```
Now access Dashboard at:

<http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/>

At this point, you’ll be prompted for a username and password. The username is admin. To get the password at the CLI, type:
```
$ kops get secrets kube --type secret -oplaintext
```
After you log in, you’ll see another prompt. 
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/k8s_dashboard_login.jpg)

Select Token. To get the Token, type:
```
$ kops get secrets admin --type secret -oplaintext
```
After typing in the token, you’ll see the Dashboard!
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/k8s_dashboard.jpg)

### Delete the Kubernetes Cluster
When you’re ready to tear down your Kubernetes cluster or if you messed up and need to start over, you can delete the cluster with a single command:
```
$ kops delete cluster --name ${KOPS_CLUSTER_NAME} --yes
```
The --yes argument is required to delete the cluster. Otherwise, Kubernetes will perform a dry run without deleting the cluster.

## Deploy Dask and Jupyter to a Kubernetes Cluster
### Install Helm
First, let’s install Helm, the Kubernetes package manager. On Mac OS X, we’ll use brew to install. If you’re on another platform, check out the Helm docs.

Go to <https://github.com/helm/helm/releases>, download Helm v2.x.x (DO NOT download v3.x.x because `helm init` doesn't work for v3.x.x ), for Mac OS X, put the helm file in the 
```
/usr/local/bin/ 
```
Then execute
```
$ helm init
```
Once you’ve initialized Helm, you should see this message: Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

### Install Dask
Next, we’re going to install a Dask chart. [Kubernetes Charts](https://github.com/helm/charts) are curated application definitions for Helm. To install the Dask chart, we’ll update the known Charts channels before installing the stable version of Dask.

```
$ kubectl create serviceaccount --namespace kube-system tiller
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 
```
Let’s install Dask.
```
$ helm init --service-account tiller --upgrade
$ helm repo update
$ helm install stable/dask
```

### Determine AWS DNS Entry
Before we’re able to work with our deployed Jupyter server, we need to determine the URL. To do this, let’s start by listing all services in the namespace:
```
$ kubectl get services
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                       AGE
kindled-skunk-dask-jupyter     LoadBalancer   100.65.190.185   ab607a07cd16241e2a27b8af5fae6fa2-814371222.us-east-1.elb.amazonaws.com    80:30715/TCP                  17m
kindled-skunk-dask-scheduler   LoadBalancer   100.65.198.224   a9730d96b3848471fafb73b38e2e1908-1350744166.us-east-1.elb.amazonaws.com   8786:32617/TCP,80:30193/TCP   17m
kubernetes                     ClusterIP      100.64.0.1       <none>                                                                    443/TCP                       36m
```
Notice that the EXTERNAL-IP displays hex values. These refer to AWS ELB (Elastic Load Balancer) entries you can find in your AWS console: EC2 -> Load Balancers. You can get the exposed DNS entry by matching the EXTERNAL-IP to the appropriate load balancer. For instance, the screenshot below shows that the DNS entry for the Jupyter node is 
```
http://ab607a07cd16241e2a27b8af5fae6fa2-814371222.us-east-1.elb.amazonaws.com/
```
### Jupyter Server
Now that we have the DNS entry, let’s go to the Jupyter server in the browser at: http://ab607a07cd16241e2a27b8af5fae6fa2-814371222.us-east-1.elb.amazonaws.com/. The first thing you’ll see is a Jupyter password prompt. 
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/dask_login.jpg)

Recall the default password is: dask. Now we login Dask:
![image](https://github.com/cy235/Deploy_Dask_Jupyter_K8S_AWS/blob/master/dask.jpg)

The notebooks include lots of useful information, such as:

Parallelizing Python code with Dask
Using Dask futures
Parallelizing Pandas operations with Dask dataframes

### Disable Jupyter Server
If you decide you’d rather run Dask only without Jupyter, that’s easy to do. Simply update the config YAML with:
```
jupyter:
  enabled: false
```
