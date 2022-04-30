# eks-alb-ingress-aws-waf

- I want to create an Application Load Balancer (ALB) using the AWS Load Balancer Controller in Amazon Elastic Kubernetes Service (Amazon EKS). Then, I want to associate the ALB Ingress with AWS WAF.

**Short Description:**

The AWS Load Balancer Controller creates an Application Load Balancer when an Ingress object is created using the kubernetes.io/ingress.class: alb annotation. The Ingress resource configures the Application Load Balancer to route HTTP or HTTPS traffic to different pods within your Amazon EKS cluster. You can use AWS WAF to monitor the HTTP or HTTPS requests that are forwarded to the Application Load Balancer.

**Resolution**

Create an OIDC provider and IAM role for the AWS Load Balancer Controller

1. Create an AWS Identity and Access Management (IAM) OIDC provider and associate the OIDC provider with your cluster:

        eksctl utils associate-iam-oidc-provider \
         --region us-east-2 \
         --cluster eksdemo-qa \
         --approve

2. Download an IAM policy for the AWS Load Balancer Controller:

        curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json

3. Create an IAM policy using the policy that you downloaded from step 2:

        aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

        {
            "Policy": {
                "PolicyName": "AWSLoadBalancerControllerIAMPolicy", 
                "PermissionsBoundaryUsageCount": 0, 
                "CreateDate": "2022-04-30T03:35:07Z", 
                "AttachmentCount": 0, 
                "IsAttachable": true, 
                "PolicyId": "ANPAWXLRIGN25YQXTQZDX", 
                "DefaultVersionId": "v1", 
                "Path": "/", 
                "Arn": "arn:aws:iam::4627568568669:policy/AWSLoadBalancerControllerIAMPolicy", 
                "UpdateDate": "2022-04-30T03:35:07Z"
            }
        }


Note: Copy the name of the policy's Amazon Resource Name (ARN) that's returned in Step 3.

4. Create an IAM role for the AWS Load Balancer Controller and attach the role to the service account created in step 2:

        eksctl create iamserviceaccount --cluster=eksdemo-qa --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::4627568568669:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve --region us-east-2

Install the AWS Load Balancer Controller using Helm 3.0.0

1. Install the TargetGroupBinding custom resource definitions:

        kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
        customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
        customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created
   
2. Add the eks-charts repository:

        helm repo add eks https://aws.github.io/eks-charts
       
3. Install the AWS Load Balancer Controller using the command that corresponds to the AWS Region for your cluster:

        # helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller --set clusterName=eksdemo-qa --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller -n kube-system --set region=us-east-2


        Release "aws-load-balancer-controller" does not exist. Installing it now.
        NAME: aws-load-balancer-controller
        LAST DEPLOYED: Sat Apr 30 03:53:17 2022
        NAMESPACE: kube-system
        STATUS: deployed
        REVISION: 1
        TEST SUITE: None
        NOTES:
        AWS Load Balancer controller installed!
        
        [root@ip-172-31-25-208 ~]# helm ls -n kube-system
        NAME                        	NAMESPACE  	REVISION	UPDATED                               	STATUS  	CHART                             	APP VERSION
        aws-load-balancer-controller	kube-system	1       	2022-04-30 03:53:17.97800165 +0000 UTC	deployed	aws-load-balancer-controller-1.4.1	v2.4.1     
4. Verify that the AWS Load Balancer Controller is installed:

        # kubectl get deployment -n kube-system aws-load-balancer-controller
        NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
        aws-load-balancer-controller   2/2     2            2           47s

5. Now setup is done, whenever we create new ingress resource create it with below annotation:

        annotations:
            kubernetes.io/ingress.class: alb
         
6. Add either an internal or internet-facing annotation to specify where you want the Ingress to create your load balancer:

        alb.ingress.kubernetes.io/scheme: internal
        -or-

        alb.ingress.kubernetes.io/scheme: internet-facing

**Deploy Smaple Example:**

1. Deploy an example application to verify that the ALB Ingress Controller creates an Application Load Balancer as a result of the Ingress object:

         kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/examples/2048/2048_full.yaml
       
         namespace/game-2048 created
         deployment.apps/deployment-2048 created
         service/service-2048 created
         ingress.networking.k8s.io/ingress-2048 created
        
2. Verify that the Ingress resource is created and is associated with an Application Load Balancer:

        kubectl get ingress ingress-2048 -n game-2048

        NAME           CLASS    HOSTS   ADDRESS                                                                  PORTS   AGE
        ingress-2048   <none>   *       k8s-game2048-ingress2-6d68b24489-553167937.us-east-2.elb.amazonaws.com   80      30s
     
3. Now we can acess the application using the ingress address: 

        http://k8s-game2048-ingress2-6d68b24489-553167937.us-east-2.elb.amazonaws.com
        
**Create a web ACL**
When you create a web ACL, choose the same Region that you're using for your Amazon EKS cluster and do the following:

1. Associate your web ACL with the Application Load Balancer.

2. Choose an Application Load Balancer as your resource type.

3. Add the Amazon CloudWatch metric name for your web ACL.

4. Add any desired conditions and rules to your web ACL.

5. Copy the AWS WAF ID from the AWS WAF console, or download the AWS WAF web ACL JSON file.

        {
          "Name": "",
          "Id": "5c677ad3-xxx-xxxx-xxxx-xxxxxxxxxxx",
          "ARN": "",
          "DefaultAction": {
            "Allow": {}
          },
          "Description": "",
          "Rules": [],
          "VisibilityConfig": {
            "SampledRequestsEnabled": ,
            "CloudWatchMetricsEnabled": ,
            "MetricName": ""
          },
          "Capacity": ,
          "ManagedByFirewallManager": 
        }
        
Add the AWS WAF web ACL annotation to your ALB Ingress
 
1. Edit the ALB Ingress and add the alb.ingress.kubernetes.io/waf-acl-id: annotation with the AWS WAF ID that you copied earlier:

     kubectl edit ingress/ingress-2048 -n game-2048


      apiVersion: v1
      items:
      - apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          annotations:
            alb.ingress.kubernetes.io/scheme: internet-facing
            alb.ingress.kubernetes.io/target-type: instance
            alb.ingress.kubernetes.io/waf-acl-id: arn:aws:wafv2:us-east-2:462483370869:regional/webacl/albWAF/5c677ad3-2d3b-4fb3-b824-a642f7ea6aa4
          finalizers:
          - ingress.k8s.aws/resources
          generation: 1
          name: ingress-2048
          namespace: game-2048
          resourceVersion: "91357"
          uid: a8a2d5c8-da4f-470f-8d83-a4d1acc5c2e3
        spec:
          rules:
          - http:
              paths:
              - backend:
                  service:
                    name: service-2048
                    port:
                      number: 80
                path: /
                pathType: Prefix
        status:
          loadBalancer:
            ingress:
            - hostname: k8s-game2048-ingress2-6d68b24489-553167937.us-east-2.elb.amazonaws.com

