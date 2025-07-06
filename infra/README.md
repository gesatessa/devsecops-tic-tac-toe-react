



```sh
tf apply -f --auto-approve

kubectl config current-context
# we need to set the cluster
aws eks update-kubeconfig --name $this_cluster --region $this_region
k config current-context
```


Test if everything is ok:
Temporarily make the service a LB.
N.B. As the worker nodes are in private subnet, there won't be any external ips set for them.
Hence, <NODE_EXT_IP>:<NodePort> is not possible to test if the setup is correct.

```sh
kubectl patch svc tic-tac-toe -p '{"spec": {"type": "LoadBalancer"}}'


# after confirmation, revert back to ClusterIP
```

