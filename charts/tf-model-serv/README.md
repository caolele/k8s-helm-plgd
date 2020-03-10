Tensorflow model serving on K8s using Helm
======

Only gRPC port is exposed in this example. This guide is generic for both minikube and GCP. The trained Tensorflow model is assumed to exported already into a GCS bucket `gs://my-project/resnet/export/123456`, which you need to change accordingly.

## Prerequisites

Many [references](https://github.com/kubernetes/ingress-nginx/issues/2444) have stated that gRPC model service require a proper TLS certificate to work. Follow the steps below (according to [this](https://github.com/kent-williams/grpc-python-kubernetes)) to generate them properly.

1. Create a SSL certificate for your target host(s); here we use `*.my.org`.
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/tls-tfms.key -out /tmp/tls-tfms.crt -subj "/CN=*.my.org"
```

2. Concatenat TLS cert & key into a pem file (will be used by your client):
```bash
cat /tmp/tls-tfms.crt /tmp/tls-tfms.key > /tmp/tls-tfms.pem
```

3. Note down the paths of files `tls-tfms.key`, `tls-tfms.crt` and `tls-tfms.pem`, which will be needed for the server and client.

4. Knowing the GCS location of the model to be served, in this example, we put the model in `gs://my-project/resnet/export/123456`

5. Obtain a GCP service key file and put it in e.g. `/tmp/sa.json`, which will be used to copy the model from GCS to a mounted volume.


## Install a model serving

Make sure you properly configured file `~/.kube/config`, and you are at the correct context, if not, set like this:
```bash
kubectl config use-context minikube
```

If you are testing this locally using say minikube, do not forget launch the Nginx ingress controller using:
```bash
minikube addons enable ingress
kubectl get pods -n kube-system
```

Deploy a ResNet model (assume you are in the same folder as this README):

```bash
helm install \
    -n tf-model-serv  \
    --set-file gcp.saKey=/tmp/sa.json \
    --set-file tls.cert=/tmp/tls-tfms.crt \
    --set-file tls.key=/tmp/tls-tfms.key \
    --set model.name=resnet \
    --set model.url=gs://my-project/resnet/export/123456 \
    resnet .
```

Note: add option `--dry-run` to verify the yaml without actually installing.

## Update a Helm installation

This will update the resnet1 installation:
```bash
helm upgrade --install \
    -n tf-model-serv  \
    --set-file gcp.saKey=/tmp/sa.json \
    --set-file tls.cert=/tmp/tls-tfms.crt \
    --set-file tls.key=/tmp/tls-tfms.key \
    --set model.name=resnet1 \
    --set model.url=gs://my-project/another-model/export/123456 \
    another-model .
```

## Prediction client

Add the following DNS entry in `/etc/hosts` file if you are testing locally using minikube (Note that you have to run `minikube ip` to find out the IP address).
```
192.168.64.3 resnet-model-serv.my.org
```

Call the ResNet model prediction service (resnet) using a [boat image](http://www.c-map.no/img/news/091106.jpg)
```bash
python grpc_client_resnet.py \
    -server resnet-model-serv.my.org:443 \
    -name resnet \
    -credential /tmp/tls-tfms.pem \
    -imageurl http://www.c-map.no/img/news/091106.jpg
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

## Uninstall
```bash
helm uninstall resnet
```


## Misc
You can also skip ingress for resnet1 by
```bash
kubectl -n tf-model-serv  port-forward svc/resnet-tf-model-serv 8080:8500
```
Then you can bypass credential when calling the prediction service:
```bash
python grpc_client_resnet.py -server 127.0.0.1:8080 -name resnet
```

