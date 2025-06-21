
```sh
curl -sL https://run.linkerd.io/install-edge | sh
export PATH=$PATH:/root/.linkerd2/bin
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

```
![image](https://github.com/user-attachments/assets/05e7eb24-66c0-4f1e-99d8-07336acd50b6)

```sh
linkerd check --pre
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check
```
```sh
linkerd viz install | kubectl apply -f -
linkerd viz check
```
