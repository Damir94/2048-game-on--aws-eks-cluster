![Screenshot](https://github.com/user-attachments/assets/eba724e3-66b4-42d0-bd2a-622bb3a684cc)


Today we shall have a step-by-step end-to-end deployment of the 2048 game on the AWS EKS platform.

Prerequisites
 - Familiarity with K8S and AWS
 - kubectl — install kubectl command line tool for working with K8S clusters on your PC
 - eksctl — install this command line tool for working with EKS clusters and is very useful in automating individual tasks
 - AWS CLI — command line tool for working with AWS services.

Installation of these tools are beyond the scope of this article. Depending on your the platform you use, you can download and install the tools from the links above.

Setting up your AWS IAM users

 - Go to the IAM (Identity and Access Management) service in the AWS Management Console.
 - Click on “Users” in the left-hand navigation pane and then click on “Add user.”
 - Enter a username for the new IAM user and select the access type (Programmatic access, AWS Management Console access, or both).
 - Choose the permissions for the IAM user by adding them to one or more IAM groups or attaching policies directly. You can use admin permissions for lab purposes but I do emphasize on the least privilege assignment.
 - If you selected “Programmatic access” during user creation, you will receive access keys (Access Key ID and Secret Access Key).
 - Store these access keys securely, as they will be used to authenticate API requests made to AWS services.

Configuring AWS CLI and Kubectl

After downloading and installing AWS CLI as per the prerequisites above, configure the AWS CLI credentials:

 - aws configure

- Enter the access key ID and secret access key of the IAM user you created earlier.
- Choose a default region and output format for AWS CLI commands e.g. us-west-2 and JSON respectively.

<img width="702" height="176" alt="Screenshot" src="https://github.com/user-attachments/assets/b78bcac9-452f-4099-949c-2a0099e17104" />

Installing EKS Cluster using Fargate

 - eksctl create cluster --name cluster-2048-game --region us-east-1 --fargate

Fargate is a serverless compute engine that lets you focus on building applications without managing servers. While we can also leverage EC2 instances for installing our EKS cluster, it requires us to manually configure and manage all the compute and memory resources, build your container image, isolate applications in separate VMs etc.

<img width="1149" height="430" alt="Screenshot 2025-08-28 at 12 16 42 PM" src="https://github.com/user-attachments/assets/bd419721-86dc-4bf0-a81c-d4b7c89c2393" />

Depending on your internet connection, the cluster takes between 10 to 20 minutes to get created. Going back to the AWS console, you can see the cluster has been created.

<img width="1444" height="236" alt="Screenshot 2025-08-28 at 12 26 23 PM" src="https://github.com/user-attachments/assets/9cc60b43-aba8-403f-b3e7-9c1a5281bae8" />

Configuring kubectl for EKS
 - Once kubectl is installed, you need to configure it to work with your EKS cluster.
 - In the AWS Management Console, go to the EKS service and select your cluster.
 - Click on the “Config” button and follow the instructions to update your kubeconfig file. Alternatively, you can use the AWS CLI to update the kubeconfig file:

 - aws eks update-kubeconfig --name your-cluster-name --region your-region

<img width="1150" height="68" alt="Screenshot 2025-08-28 at 12 35 16 PM" src="https://github.com/user-attachments/assets/c3e8438e-2df0-4199-88b7-ce66ed9a74be" />

Create a Fargate Profile

By default, the “fp-default” fargate profile is created. The “fp-default” has only two namespaces — default and kube-system. We shall create our own Fargate profile to enable us deploy pods on our own created namespace:

eksctl create fargateprofile \
    --cluster cluster-2048-game \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048

<img width="977" height="187" alt="Screenshot 2025-08-28 at 12 38 12 PM" src="https://github.com/user-attachments/assets/52fc1057-7e91-4f01-82dc-b6b32c95e320" />

<img width="1384" height="222" alt="Screenshot 2025-08-28 at 12 38 34 PM" src="https://github.com/user-attachments/assets/6f90c35b-12b5-4651-849f-f3a58a39ad6b" />


Once the fargate profile is created, we proceed to create our deployment using kubectl. The deployment file is linked here. Use the below command to directly create the deployment.

 - kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml


<img width="1172" height="138" alt="Screenshot 2025-08-28 at 12 39 39 PM" src="https://github.com/user-attachments/assets/dbade500-bd56-4794-b6f9-37c5b269756e" />

Verify that the pods/service/ingress are getting created with the command

 - kubectl get pods -n game-2048
 - kubectl get svc -n game-2048
 - kubectl get ingress -n game-2048

<img width="793" height="375" alt="Screenshot 2025-08-28 at 12 41 33 PM" src="https://github.com/user-attachments/assets/11cc7feb-a9e8-4ba9-aad2-2a9dab974a0b" />

For the service, note that we do not have an external IP. Users outside the cluster cannot access our application. We therefore need to create an ingress from which our users can access our 2048 game.

Analyzing our ingress resource “ingress-2048”, we see that we have a wildcard (*) for the hosts implying that anyone can access. We have a port as well but no address. This is because our ingress resource requires an ingress controller to be deployed, which then reads the ingress resource creates and configures the load balancer before an IP address is assigned.

One prerequisite before creating an ingress controller is configuring IAM OIDC provider. The ingress controller (ALB controller) needs to access the Application Load Balancer. Note that a controller is just a K8S pod which needs to talk to AWS resources. This can only be achieved by providing the necessary IAM permissions hence the use of a connector (OIDC). Run the below command to associate the identity provider with our cluster:

 - eksctl utils associate-iam-oidc-provider --cluster cluster_name --approve --region region_name

<img width="964" height="145" alt="Screenshot 2025-08-28 at 12 44 40 PM" src="https://github.com/user-attachments/assets/5dcb917b-74e3-4c70-b69a-41435a8cdd34" />

Our ALB controller pod should have access to AWS services such as ALB in order to create the service. Let us grant permissions by running the iam_policy.json file from the official ALB controller documentation.

 - curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

<img width="961" height="90" alt="Screenshot 2025-08-28 at 12 45 29 PM" src="https://github.com/user-attachments/assets/8e6e66c2-2d9e-4fc5-9570-4c906559dd41" />

Create an IAM policy:

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

<img width="962" height="131" alt="Screenshot 2025-08-28 at 12 46 40 PM" src="https://github.com/user-attachments/assets/34ba1838-d86c-4a3b-99f2-cc4fe7a21762" />

Create IAM Role. Remember to modify the AWS account ID and cluster name.

eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=us-east-1

<img width="962" height="424" alt="Screenshot 2025-08-28 at 12 48 42 PM" src="https://github.com/user-attachments/assets/95b95e0d-8c3c-46e8-8c52-b4fe6a15b0e8" />

Finally we can deploy the ALB Controller. We shall use a helm chart to create the controller and use the service account (iamserviceaccount) we created earlier for running the pod.

Let us add helm to our repo and update our helm repo

 - helm repo add eks https://aws.github.io/eks-charts
 - helm repo update eks

<img width="802" height="136" alt="Screenshot 2025-08-28 at 12 50 04 PM" src="https://github.com/user-attachments/assets/97631546-5cf8-444d-965c-0cdb8feef481" />

Now we can install the controller. Again, remember to fill in the variables as per your account and cluster details. VPCId is located in the Networking tab of your EKS cluster.

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>

<img width="962" height="291" alt="Screenshot 2025-08-28 at 12 53 05 PM" src="https://github.com/user-attachments/assets/ae6d1dac-041d-4f1f-89b8-ccaf7a0b3433" />

Verifying the load balancer has been created:

 - kubectl get deployment -n kube-system aws-load-balancer-controller

<img width="957" height="81" alt="Screenshot 2025-08-28 at 12 53 34 PM" src="https://github.com/user-attachments/assets/b7a6dec9-e5db-49bb-b09a-7130df46e855" />

We can see below that two replicas of the ALB have been created:

<img width="814" height="180" alt="Screenshot 2025-08-28 at 12 54 31 PM" src="https://github.com/user-attachments/assets/2ccd4fa8-612b-41d2-8f22-8f801fba3ec2" />

From the Dashboard, we can verify that the LB has been created.

<img width="1615" height="236" alt="Screenshot 2025-08-28 at 12 56 58 PM" src="https://github.com/user-attachments/assets/89ec1c20-b556-4e2a-aedc-09dd8b26ce09" />

Now, if we check the ingress resource, we should see the LB in the Address section:

<img width="1052" height="111" alt="Screenshot 2025-08-28 at 12 57 49 PM" src="https://github.com/user-attachments/assets/fa45281a-c229-4582-a658-2d83644f932b" />

We can use this address in a browser and we should finally have our game!

<img width="757" height="990" alt="Screenshot 2025-08-28 at 12 58 51 PM" src="https://github.com/user-attachments/assets/0be655d2-5412-4b54-a6da-1fe00e6a26de" />

We have successfully deployed our 2048 game publicly. Remember to delete your EKS cluster after playing around with the game.

 - eksctl delete cluster --name demo-cluster --region region-name

<img width="946" height="117" alt="Screenshot 2025-08-28 at 1 01 04 PM" src="https://github.com/user-attachments/assets/db915b26-fe7d-41b2-a98e-a24881be12f5" />


