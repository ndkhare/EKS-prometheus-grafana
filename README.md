# EKS-prometheus-grafana

The guide to install Prometheus and Grafana on pre-configured EKS cluster in AWS, and exposing the cluster using LoadBalancer.

## Getting started

Pre-requisites:
1. AWS Account
2. AWS CLI v2 on the local host configured with IAM credentials. Find the detailed instructions here: https://docs.aws.amazon.com/cli/v1/userguide/cli-authentication-user.html
3. Install eksctl. Find the detailed instructions here: https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eksctl.html
4. Install kubectl. Find the detailed instructions here: https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
5. Install helm. Find the detailed instructions here: https://docs.aws.amazon.com/eks/latest/userguide/helm.html



## Steps
### Create the EKS Cluster
Open the `cluster.yaml` and make changes to region, version and name as preferred.
1. Create a cluster in your AWS account using eksctl and the `cluster.yaml` file. If you've configured a profile for your AWS CLI, ensure to append the command with the `--profile <name>`
```eksctl create cluster -f cluster.yaml```
2. Open the AWS console and nvaigate to EKS Cluster. Open the cluster console page and allow the IAM access required as prompted on the top.
3. You can check the status of deployment on CloudFormation or the local host terminal.
4. Once the cluster is created, a message will read: `EKS cluster "name" in "region specified" region is ready`

Run the following commands to confirm cluster is created:
```kubectl get nodes``` --> You should see 3 nodes with the private DNS names and their status
```kubectl get deployment ebs-csi-controller -n kube-system``` --> You should see `ebs-csi-controller` in output
```kubectl get sa -A | egrep "cert-manager|efs|aws-load|external|cloudwatch-agent|cluster-autoscaler"``` --> You should see kube-system, cloudwatch in the output

__This confirms the cluster is created.__

The next step is to install Load Balancer controller (load-balancer-controller) to the EKS Cluster. For that we need to create some environment variables
1. ```kubectl config current-context`````` --> You should see something similar to `<profile>@prometheus.region.eksctl.io`. If you do not see this, please create a namespace using ```kubectl create namespace <name>```
2. ```export CLUSTER_NAME=$(aws eks describe-cluster --region <region> --name prometheus --profile <profile-name-optional> --query "cluster.name" --output text)```
3. ```export CLUSTER_REGION=$(aws eks describe-cluster --name ${CLUSTER_NAME} --profile <profile-name-optional> --query "cluster.arn" --output text | cut -d: -f4)```
4. ```export CLUSTER_VPC=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${CLUSTER_REGION} --profile <profile-name-optional> --query "cluster.resourcesVpcConfig.vpcId" --output text)```
5. ```export ACCOUNT_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${CLUSTER_REGION} --profile <profile-name-optional> --query "cluster.arn" --output text | cut -d':' -f5)```
6. Optional - Check the environment variables created using ```env```
7. ```kubectl get sa aws-load-balancer-controller -n kube-system -o yaml```
8. ```curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json``` --> Note the location of where the file is saved.
9. ```aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json --profile <profile-name-optional>``` (Use the exact file location)
10. ```helm repo add eks https://aws.github.io/eks-charts``` OR ```helm repo update eks``` based on whether you've already added EKS using helm
11. ```helm install aws-load-balancer-controller eks/aws-load-balancer-controller --namespace kube-system --set clusterName=${CLUSTER_NAME} --set serviceAccount.create=false --set region=${CLUSTER_REGION} --set vpcId=${CLUSTER_VPC} --set serviceAccount.name=aws-load-balancer-controller```
12. ```kubectl create namespace prometheus --save-config``` --> In case Step#2 was missed, you can create and save the config for the namespace

### Installing and Applying Prometheus to the cluster and editing to expose the LoadBalancer to public
Edit the `prometheus.yaml` as required.

1. ```kubectl apply -n prometheus -f prometheus.yaml```
2. ```helm install stable prometheus-community/kube-prometheus-stack -n prometheus```
3. ```kubectl get ingress -n prometheus --> You should see a LoadBalancer in the output```
#### Updating the ingress groups for the cluster and loadbalancer
Edit the `update-ingress-prometheus.yaml` as required.

4. ```kubectl apply -f updated-ingress-prometheus.yaml``` --> You should see `ingress.networking.k8s.io/ingress-prometheus configured` in the output
5. ```kubectl get ingress -n prometheus``` --> You should see the LoadBalancer in the output
#### Editing the svc to expose Prometheus and Grafana:
1. ```kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus``` --> Change the `type` from Cluter to LoadBalancer
2. ```kubectl edit svc stable-grafana -n prometheus``` --> Change the `type` from Cluter to LoadBalancer
3. ```kubectl get svc -n prometheus``` --> You should see LoadBalancer DNS names in Prometheus and Grafana rows

Use the Grafana default credentials to log into Grafana using the LoadBalancer url from output.

Use the LoadBalancer DNS name and open it in a browser.


*Username - admin*
*Password - prom-user*


# References:
https://community.aws/content/2dr1uEC9C5CEKgiLUFY6fKRL5yo/navigating-amazon-eks-eks-cluster-high-traffic?lang=en
https://community.aws/tutorials/navigating-amazon-eks/eks-cluster-load-balancer-ipv4
https://towardsaws.com/devops-hands-on-lab-how-to-provision-and-monitor-eks-cluster-using-prometheus-and-grafana-helm-740abfc6b805
