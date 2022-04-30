# eks-alb-ingress-aws-waf

Short Description:

The AWS Load Balancer Controller creates an Application Load Balancer when an Ingress object is created using the kubernetes.io/ingress.class: alb annotation. The Ingress resource configures the Application Load Balancer to route HTTP or HTTPS traffic to different pods within your Amazon EKS cluster. You can use AWS WAF to monitor the HTTP or HTTPS requests that are forwarded to the Application Load Balancer.


# curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json

# aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# eksctl create iamserviceaccount --cluster=eksdemo-qa --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::462483370869:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve --region us-east-2

# kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created
