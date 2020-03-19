# Deploy_Dask_Jupyter_K8S_AWS
In this project, I will deploy dask and jupyter into kubernetes (k8s) cluster in Amazon Web Services (AWS)

## Set up a kubernetes cluster in AWS
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
brew update && brew install kops kubectl
```

### Set up the k8s cluster
The first thing we need to do is create an S3 bucket for kops to use to store the state of the Kubernetes cluster and its configuration. We’ll use the bucket name cy235-kops-state-store
```
$ aws s3api create-bucket --bucket ramhiser-kops-state-store --region us-east-1
```
Before creating the cluster, let’s set two environment variables: KOPS_CLUSTER_NAME and KOPS_STATE_STORE. For safe keeping you should add the following to your ~/.bash_profile or ~/.bashrc configs (or whatever the equivalent is if you don’t use bash).
```
export KOPS_CLUSTER_NAME=cy235.k8s.local
export KOPS_STATE_STORE=s3://cy235-kops-state-store
```
Now, to generate the cluster configuration:
```
$ kops create cluster --node-count=2 --node-size=t2.medium --zones=us-east-1a
```
Note: this line doesn’t launch the AWS EC2 instances. It simply creates the configuration and writes to the s3://cy235-kops-state-store bucket we created above. In our example, we’re creating 2 t2.medium EC2 work nodes in addition to a c4.large master instance (default).
```
$ kops edit cluster
```
Now that we’ve generated a cluster configuration, we can edit its description before launching the instances. The config is loaded from s3://ramhiser-kops-state-store. You can change the editor used to edit the config by setting $EDITOR or $KUBE_EDITOR. For instance, in my ~/.bashrc, I have export KUBE_EDITOR=cy235.

Time to build the cluster. This takes a few minutes to boot the EC2 instances and download the Kubernetes components.
```
kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```
After waiting a bit (5~10 minutes), let’s validate the cluster to ensure the master + 2 nodes have launched.
```
$ kops validate cluster
Validating cluster cy235.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-us-east-1a	Master	m3.medium	1	1	us-east-1a
nodes			Node	t2.medium	2	2	us-east-1a

NODE STATUS
NAME				ROLE	READY
ip-172-20-40-5.ec2.internal	node	True
ip-172-20-40-70.ec2.internal	master	True
ip-172-20-55-98.ec2.internal	node	True

Your cluster cy235.k8s.local is ready
```
Note: If you ignore the message Cluster is starting. It should be ready in a few minutes. and validate too early, you’ll get an error. Wait a little longer for the nodes to launch, and the validate step will return without error.

Finally, you can see your Kubernetes nodes with kubectl:
```
 kubectl get nodes
NAME                           STATUS   ROLES    AGE    VERSION
ip-172-20-40-5.ec2.internal    Ready    node     177m   v1.16.7
ip-172-20-40-70.ec2.internal   Ready    master   178m   v1.16.7
ip-172-20-55-98.ec2.internal   Ready    node     177m   v1.16.7
Chens-MacBook-Pro:~ chenyi$ 
```
## Kubernetes dashboard
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
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
