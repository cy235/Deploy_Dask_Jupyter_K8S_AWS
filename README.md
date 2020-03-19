# Deploy_Dask_Jupyter_K8S_AWS
In this project, I will deploy dask and jupyter into kubernetes (k8s) cluster in Amazon Web Services (AWS)

## Set up a kubernetes cluster in AWS
### Prerequisites
Before setting up the Kubernetes cluster, you’ll need an AWS account and an installation of the AWS Command Line Interface.

Make sure to configure the AWS CLI to use your access key ID and secret access key:
```
$ aws configure
AWS Access Key ID [None]: your_key_ID
AWS Secret Access Key [None]: your_secret_key
Default region name [None]: us-east-1
Default output format [None]: json
```
