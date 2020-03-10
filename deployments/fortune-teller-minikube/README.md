## Fortune-teller example on minikube

It is the Hello World example of testing gRPC service over k8s.

### Prerequisites
- Install and config [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

- Make sure you properly configured file `~/.kube/config`, and you are at the correct context, if not, set like this:
```bash
kubectl config use-context minikube
```

- launch and check the Nginx ingress controller with
```bash
minikube addons enable ingress
kubectl get pods -n kube-system
```

### Deploy the app

- Clone this repo and enter the following folder:
```bash
cd k8s-helm-plgd/deployments/fortune-teller-minikube
```

- Deploy everything:
```bash
kubectl apply -f app.yaml
kubectl apply -f svc.yaml
kubectl apply -f ingress.yaml
```

### Test the deployed app

- Find out the exposed IP address (assume you get 192.168.64.3):
```bash
minikube ip
```

- Add the following DNS entry in `/etc/hosts` file:
```
192.168.64.3 fortune-teller.info
```

- Test with grpcurl:
```bash
grpcurl -insecure fortune-teller.info:443 build.stack.fortune.FortuneTeller/Predic
```

Note that this example works without a proper TLS certificate.

### Remove the app
```bash
kubectl delete ing fortune-ingress
kubectl delete deployment fortune-teller-app
kubectl delete svc fortune-teller-service
```