# HPA
Dynamic scaling triggered via CPU utilization

1. Prerequisites
Before proceeding, ensure you have: ✅ AWS CLI, eksctl, kubectl, and Git Bash installed (✔️ you already have them).
✅ IAM permissions to create an EKS cluster and related resources.
✅ A configured AWS profile (aws configure if not already set).

2. Create an AWS EKS Cluster
Run the following command in Git Bash to create an EKS cluster with auto-scaling node groups:
eksctl create cluster --name demo-cluster \
    --region us-east-1 \
    --nodegroup-name demo-nodes \
    --node-type t3.medium \
    --nodes 2 \
    --nodes-min 2 \
    --nodes-max 5 \
    --managed

Explanation:
Creates a cluster named demo-cluster in us-east-1.
Uses an EC2 node group (t3.medium instances).
Starts with 2 nodes, allowing it to scale up to 5.
🚀 This may take 10-15 minutes.

After creation, verify:
eksctl get cluster --name demo-cluster --region us-east-1
kubectl get nodes

3. Deploy an Application to EKS
Let’s deploy a sample Nginx application to simulate real-world load.

Create a Deployment:
kubectl create deployment nginx-app --image=nginx --replicas=2
Expose as a Service:
kubectl expose deployment nginx-app --port=80 --target-port=80 --type=LoadBalancer

4. Install Cluster Autoscaler
Cluster Autoscaler automatically adjusts node count based on demand.

Step 1: Get your cluster’s Auto Scaling Group name
aws autoscaling describe-auto-scaling-groups --region us-east-1
Find your Auto Scaling Group name (eksctl-demo-nodes-xxxx).

Step 2: Deploy Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

Step 3: Patch the Deployment
kubectl patch deployment cluster-autoscaler -n kube-system --type json \
    -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/command", "value": ["./cluster-autoscaler", "--v=4", "--stderrthreshold=info", "--cloud-provider=aws", "--skip-nodes-with-local-storage=false", "--expander=least-waste", "--nodes=2:5:eksctl-demo-nodes-xxxx"]}]'
Replace eksctl-demo-nodes-xxxx with your Auto Scaling Group name.

Step 4: Enable Autoscaler
kubectl annotate deployment cluster-autoscaler -n kube-system cluster-autoscaler.kubernetes.io/safe-to-evict=false

5. Install Horizontal Pod Autoscaler (HPA)
HPA scales pods based on CPU usage.

Step 1: Apply Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Step 2: Verify Metrics Server
kubectl get deployment metrics-server -n kube-system
Step 3: Apply HPA
kubectl autoscale deployment nginx-app --cpu-percent=50 --min=2 --max=10
🚀 This means if CPU usage exceeds 50%, it will scale up to 10 pods.

Step 4: Check HPA Status
kubectl get hpa

6. Test the Autoscaling
Step 1: Simulate High Load
Run the following command to stress-test the application:
 kubectl run load-generator5 --image=alpine -- sh -c "while true; do wget -o- http://nginx-app; done"

Step 2: Watch Autoscaler in Action
Monitor pod scaling:
kubectl get hpa -w
kubectl get pods -o wide
Monitor node scaling:

kubectl get nodes
7. Cleanup
After testing, delete the cluster to avoid AWS charges:
eksctl delete cluster --name demo-cluster --region us-east-1


Troubleshooting:
1. load-generator pod was running and populating traffic onto nginx-app deployment, but, neither number of nodes nor pods changed even after several minutes. Why?
The HPA was unable to detect CPU traffic as the deployment was given resource(requests and limits) section was not configured.
It resolved after the resource section was added.

2. The HPA was triggered as the load was not sufficient to trigger scaling, so I have increased load and decreased CPU utilization target to trigger scaling and the results were immediately seen.
