
# eks =================================================================== #
this_cluster="tic-tac-toe"
this_region="us-east-1"
GH_TOKEN=""
GH_USERNAME="gesatessa"
# ----------------------------------------------------------------------- #
## cluster
kubectl config get-contexts -o name
aws eks list-clusters

eksctl create cluster -f ./k8s/eks-cluster-config.yaml

eksctl delete cluster --name $this_cluster --region $this_region

# ----------------------------------------------------------------------- #
## ghcr.io

echo $GH_TOKEN | docker login ghcr.io -u $GH_USERNAME --password-stdin
docker push ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1

### verify
docker run --rm -p 3000:80 ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1
curl http://localhost:3000

docker inspect ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1

# just get the config section (cmd, entrypoint, exposed port ...)
docker inspect --format='{{json .Config}}' ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1 | jq
# or using jq alone
docker inspect ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1 | jq '.[0].Config'


# ----------------------------------------------------------------------- #
## k apply -f
k apply -f k8s/manifests/app-deploy-svc.yaml

### ImagePullBackOff
🔒 1. Create a Kubernetes Secret with your GitHub PAT

kubectl create secret docker-registry github-container-registry \
  --docker-server=ghcr.io \
  --docker-username=$GH_USERNAME \
  --docker-password=$GH_TOKEN

📦 2. Reference the Secret in Your Deployment
In your Deployment YAML (or Helm values), add this under the Pod spec:

spec:
  imagePullSecrets:
    - name: github-container-registry


#### ddx of secret for pulling images
1) secret Type 
kubectl describe secret github-container-registry

it should show that it's of type kubernetes.io/dockerconfigjson. If it's a different type, the secret won’t work for image pulling.

2) the secret is referenced in the pod spec

kubectl get deployment tic-tac-toe -o yaml

under spec.template.spec you should see

imagePullSecrets:
  - name: github-container-registry

k get deploy tic-tac-toe -o yaml | grep imagePullSecrets: -A 1

If it’s missing, the pod won’t use the credentials. You can patch your deployment to add it if needed:

kubectl patch deployment tic-tac-toe \
  --patch '{"spec": {"template": {"spec": {"imagePullSecrets": [{"name": "github-container-registry"}]}}}}'

3) confirm your $GH_TOKEN was injected correctly:

kubectl get secret github-container-registry \
  -o jsonpath="{.data.\.dockerconfigjson}" \
  | base64 --decode \
  | jq


# the value in the auth section should match this
echo -n "$GH_USERNAME:$GH_TOKEN" | base64

echo adds a newline — \n — which alters the base64 output.
echo -n avoids that, giving the correct username:token string for encoding.


4) try pulling image manually on a Node
SSH into a worker node (e.g., EC2 instance):

docker login ghcr.io -u $GH_USERNAME --password-stdin
docker pull ghcr.io/gesatessa/devsecops-tic-tac-toe-react:v1

If this fails there too, then your token is the issue


### test if the app is reachable

k get nodes -o wide

add inbound rules in the sg of one of the nodes and see if the app is reachable.
note: we've already configured the service as NodePort and assigned 32123 as nodeport in svc spec.
http://<EC2_PUBLIC_IP>:32123



# ----------------------------------------------------------------------- #
## ingress

### ALB

#### prerequisites:
0) cluster is up and running

1) your nodes are tagged for ALB:

kubectl get nodes --show-labels
# you should see: eks.amazonaws.com/nodegroup=<your-node-group-name>


2) IAM OIDC provider is set up:
eksctl utils associate-iam-oidc-provider --cluster $this_cluster --approve


#### Install AWS Load Balancer Controller:

1) create the policy
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

rm iam-policy.json


2) Create a Service Account for the controller
AWS_ACC_ID="018733487945"

eksctl create iamserviceaccount \
  --cluster $this_cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::$AWS_ACC_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

3) Install the AWS Load Balancer Controller via Helm
# add the helm repo

helm repo add eks https://aws.github.io/eks-charts
helm repo update

# install the chart
this_vpcId=$(aws eks describe-cluster --name $this_cluster --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$this_cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$this_region \
  --set vpcId=$this_vpcId \
  --set ingressClass=alb \
  --set defaultBackend.enabled=false

4) configure ingress resource for ALB

e.g.,

# k8s/manifests/app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tic-tac-toe-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTP
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: tic-tac-toe
                port:
                  number: 80


N.B. make sure also that the service is ClusterIP as we're using ingress.

apply the changes
k apply -f k8s/manifests/app-ingress.yaml

verify alb creation
k get ing


You should see an ADDRESS populated with the ALB DNS name
(e.g. k8s-default-tictacto-fa12ff1527-1116105324.us-east-1.elb.amazonaws.com)

Once the ALB is provisioned (may take a minute), you can visit the DNS name in a browser:
http://<alb-dns-name>


### NGINX Ingress Controller
add the helm repo & update

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update


# install nginx ingress controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.publishService.enabled=true


The publishService.enabled=true option allows it to expose an external LoadBalancer.

# Wait for the External IP: 
k get svc -n ingress-nginx

k apply -f k8s/manifests/app-ingress-nginx.yaml

# access the app: http://<external-ip>
You should see your app running via NGINX Ingress.



# ----------------------------------------------------------------------- #
# argocd


(optional) install argocd cli 
# Linux (x86_64)
# Get latest version tag
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" \
  | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')

# Download to /tmp to avoid permission issues
curl -sSL -o /tmp/argocd "https://github.com/argoproj/argo-cd/releases/download/${VERSION}/argocd-linux-amd64"

sudo mv /tmp/argocd /usr/local/bin/argocd
sudo chmod +x /usr/local/bin/argocd

argocd version


install ArgoCD
k create namespace argocd
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


access the UI
1) k port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
Then go to: https://localhost:8080

2) loadbalancer
argocd login ac4b5ff208ad34914a507eb2644b42e0-676988942.us-east-1.elb.amazonaws.com \
  --username admin \
  --password 4bV0bqLj9-q3khkW \
  --insecure

argocd app list


### create n Argo CD application:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tic-tac-toe
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://github.com/your-user/devsecops-tic-tac-toe-react.git
    targetRevision: HEAD
    path: k8s/overlays/dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true


k apply -f argocd/application.yaml
# ----------------------------------------------------------------------- #
# gha



# ----------------------------------------------------------------------- #
