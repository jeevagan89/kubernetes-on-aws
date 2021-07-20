# Deploy Consul and Vault on Amazon Elastic Kubernetes Service (EKS)

Guide to deploy a Consul datacenter and Vault to Elastic Kubernetes Services (EKS) on AWS with high 
availability and data persistence. We will use HashiCorp’s official Helm charts to deploy these
services.

1. Setup AWS Load Balancer Controller

We will use AWS Load Balancer Controller as the ingress controller. Please follow the below steps 
to deploy it to EKS cluster.

  1. Create an IAM policy using the policy `lb-controller-iam-policy.json`.
      ```  
      aws iam create-policy \
          --policy-name AWSLoadBalancerControllerIAMPolicy \
          --policy-document file://lb-controller-iam-policy.json
      ```
  1. Create an IAM role and annotate the Kubernetes service account named 
`aws-load-balancer-controller` in the `kube-system` namespace for the AWS Load Balancer Controller.
      ```
      eksctl create iamserviceaccount \
        --cluster=<EKS_Cluster_Name> \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --attach-policy-arn=arn:aws:iam::<AWS account ID>:policy/AWSLoadBalancerControllerIAMPolicy \
        --override-existing-serviceaccounts \
        --approve  
      ```

  1. Install the AWS Load Balancer Controller using Helm V3

    1. Create a service account for AWS Load Balancer Controller. Replace the IAM role ARN in the
    deployment file.
      ``` 
      kubectl apply -f lb-controller-service-account.yaml
      ```

    1. Install the TargetGroupBinding custom resource definitions.
      ```
      kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"
      ```

    1. Install the AWS Load Balancer Controller
      ```
      helm repo add eks https://aws.github.io/eks-charts
      helm repo update
      helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
        -f lb-controller-values.yaml \
        -n kube-system
    ```

1. Setup External DNS

  1. Create an IAM policy using the policy `external-dns-iam-policy.json`.
      ```  
      aws iam create-policy \
          --policy-name ExternalDNSIAMPolicy \
          --policy-document file://external-dns-iam-policy.json
      ```
  1. Create an IAM role and annotate the Kubernetes service account named  `external-dns` in the
   `kube-system` namespace for the External DNS.
      ```
      eksctl create iamserviceaccount \
        --cluster=<EKS_Cluster_Name> \
        --namespace=kube-system \
        --name=external-dns \
        --attach-policy-arn=arn:aws:iam::<AWS account ID>:policy/ExternalDNSIAMPolicy \
        --override-existing-serviceaccounts \
        --approve  
      ```

  1. Replace the IAM role ARN Private hosted zone name in th deployment file and run,
    ```
    kubectl apply -f external-dns.yaml
    ```

1. Setup Amazon EFS CSI driver

We will EFS as the persistence storage to store Consul data. An EFS file system can be accessed 
from multiple availability zones and it is valuable for multi-AZ cluster.

  1. Create an IAM policy using the policy `efs-iam-policy.json`.
      ```  
      aws iam create-policy \ 
        --policy-name EFSCSIControllerIAMPolicy \ 
        --policy-document file://efs-iam-policy.json 
      ```

  1. Create an IAM role and annotate the Kubernetes service account named `efs-csi-controller-sa` 
  in the `kube-system` namespace.
      ```
      eksctl create iamserviceaccount \ 
        --cluster=<EKS_Cluster_Name> \ 
        --region <AWS_Region> \ 
        --namespace=kube-system \ 
        --name=efs-csi-controller-sa \ 
        --override-existing-serviceaccounts \ 
        --attach-policy-arn=arn:aws:iam::<AWS account ID>:policy/EFSCSIControllerIAMPolicy \ 
        --approve
      ```

  1. Install the Amazon EFS CSI driver using the Helm chart
    ```
    helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver
    helm repo update
    helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver -f efs-values.yaml \
    -n kube-system
    ```

1. Create an EFS share and setup a storage class

  1. Create an EFS file system in your AWS account and update the file system id in 
  `efs-storageclass.yaml`. Refer the [link](https://docs.aws.amazon.com/efs/latest/ug/gs-step-two-create-efs-resources.html) 
  for more instructions.

  1. Create a storage class for Consul.
    ```
    kubectl apply -f efs-storage-class.yaml
    ```

1. Setup Consul

  1. Create a new namespace for Consul
    ```
    kubectl apply -f consul-namespace.yaml
    ```

  1. Create docker hub secret in consul namespace
    ```
    export DOCKERHUB_USERNAME=<USERNAME>
    export DOCKERHUB_PASSWORD=<PASSWORD>

    kubectl create secret docker-registry dockerhub-credentials \
      --docker-username=$DOCKERHUB_USERNAME \
      --docker-password=$DOCKERHUB_PASSWORD \
      --docker-server="https://index.docker.io/v1/" \
      -n consul
    ```

  1. Create a SSL certificate in AWS Certificate Manager for the Consul UI domain and update the
   `consul-values.yaml` with it's ARN.

  1. Create Consul gossip encryption secret
    ```
    kubectl -n consul create secret generic consul-gossip-encryption-key --from-literal=key=$(consul keygen)
    ```

  1. Setup Consul using Helm
    ```
    helm repo add hashicorp https://helm.releases.hashicorp.com
    helm upgrade -i consul hashicorp/consul --version 0.32.1 -f consul-values.yaml -n consul
    ```

1. Setup Vault using Helm
    ```
    helm upgrade -i vault hashicorp/vault --version 0.13.0 -f vault-values.yaml -n vault
    ```
