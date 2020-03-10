## ResNet Tensorflow model serving example on minikube

It is the Hello World example of serving a Tensorflow model via Tensorflow serving over gRPC protocol on k8s (minikube). [Here](https://www.tensorflow.org/tfx/serving/serving_kubernetes) is a reference about how to do it using GCP.

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

- Create your own docker image containing the ResNet model and push it to your preferred docker registry:
```bash
rm -rf /tmp/resnet
mkdir /tmp/resnet
curl -s http://download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v2_fp32_savedmodel_NHWC_jpg.tar.gz | \
tar --strip-components=2 -C /tmp/resnet -xvz
ls /tmp/resnet/*
docker run -d --name serving_base tensorflow/serving
docker cp /tmp/resnet serving_base:/models/resnet
docker commit --change "ENV MODEL_NAME resnet" serving_base docker-registry.your.org.com/$USER/resnet_serving:v0.1
docker push docker-registry.your.org.com/$USER/resnet_serving:v0.1
```
You need to replace the image config in `app.yaml` with the correct `docker-registry.your.org.com/$USER/resnet_serving:v0.1`

### Prepare TLS credentials

This example require proper TLS credentials to work.
1. Create a SSL certificate for your target host(s); here we use `resnet.info`.
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls-resnet.key -out /tmp/tls-resnet.crt -subj "/CN=resnet.info"
```

2. Concatenat TLS cert & key into a pem file (will be used by your client):
```bash
cat /tmp/tls-resnet.crt /tmp/tls-resnet.key > /tmp/tls-resnet.pem
```

3. Note down the paths of files `tls-resnet.key`, `tls-resnet.crt` and `tls-resnet.pem`, which will be needed for the server and client.


### Deploy the app

- Clone this repo and enter the following folder:
```bash
cd k8s-helm-plgd/deployments/resnet-minikube
```

- Create TLS secret:
```bash
kubectl create secret tls resnet-secret --key /tmp/tls-resnet.key --cert /tmp/tls-resnet.crt
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
192.168.64.3 resnet.info
```

- Call the ResNet service (resnet) using a default [cat image](https://tensorflow.org/images/blogs/serving/cat.jpg):
```bash
python grpc_client_resnet.py \
    -server resnet.info:443 \
    -name resnet \
    -credential /tmp/tls-resnet.pem
``` 
the following printout is expected:
```
outputs {
  key: "classes"
  value {
    dtype: DT_INT64
    tensor_shape {
      dim {
        size: 1
      }
    }
    int64_val: 629
  }
}
```

### Remove the app
```bash
kubectl delete ing resnet-ingress
kubectl delete deployment resnet-app
kubectl delete svc resnet-service
kubectl delete secret resnet-secret
```