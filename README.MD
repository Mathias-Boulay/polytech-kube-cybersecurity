# Quick install procedure

Assuming you have all command line tools installed:

`go install sigs.k8s.io/kind@v0.26.0 && kind create cluster`

`kind create cluster`
`go install sigs.k8s.io/cloud-provider-kind@latest`

`kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml`

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## Application

```bash
# deploy the app
kubectl apply -f ./k8s/
```

On another shell, if you use kind

`cloud-provider-kind`

Test that you can reach the app:

```bash
# get the loadalancer IP

export LOADBALANCER_IP=$(kubectl get services \
   --namespace ingress-nginx \
   ingress-nginx-controller \
   --output jsonpath='{.status.loadBalancer.ingress[0].ip}')

# should output two random words

curl ${LOADBALANCER_IP}/foo
```

## Kyverno

```bash
# Install kyverno
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.13.0/install.yaml
```

```bash
# deploy policies, maybe retry if it fails because of kyverno not being ready
kubectl apply -f ./k8s/kyverno/
```

At this point, some resources are not passing the security policies.
Test some by deleting and re-creating resources

```bash
# Other namespaces cannot be created
kubectl delete -f ./k8s/00_namespace_not_sentence.yml && kubectl apply -f ./k8s/00_namespace_not_sentence.yml
```

```bash
# No target label
kubectl delete -f ./k8s/05_verbs-deployment.yaml && kubectl apply -f ./k8s/05_verbs-deployment.yaml
```

## Falco

First, disable a kyverno policy that is too over-reaching

```bash
kubectl delete -f ./k8s/kyverno/01_restrict_ns_policy.yml
```

Then install falco using helm

```bash
helm install --replace falco --namespace falco --create-namespace --set driver.kind=modern_ebpf --set tty=true falcosecurity/falco
```

For some permission related reasons that I am too lazy to investigate, if you use k3d (or kind) and arch linux it won't work...

// https://github.com/falcosecurity/falco-website/issues/1229
// https://github.com/falcosecurity/falco-website/issues/1135
helm install falco falcosecurity/falco \
 --set driver.kind=modern_bpf \
 --set tty=true \

helm install --replace falco --namespace falco --create-namespace --set driver.kind=modern_ebpf --set tty=true falcosecurity/falco

For some permission related reasons that I am too lazy, if you use k3d and arch linux it won't work...

### Falco payload

```bash
kubectl -n sentence exec nouns-<whatever> -- cat /etc/shadow
```

It should raise more than a notice

## Istio

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
cd ..
```

```bash
# install a basic operator
istioctl install -f ./k8s/istio/00_install_operator.yaml -y
```

```bash
# Enable mutual TLS for every pod inside the namespace sentence
kubectl apply -f ./k8s/istio/05_enable_mtls.yaml
```

Delete and re-create the deployment

```bash
kubectl delete -f ./k8s/ && kubectl apply -f ./k8s/
```

At this point, you may see some resources not passing the blocking kyverno policies.
For now, remove the policies when applying resources (or fix the resources in question)

```bash
kubectl delete -f ./k8s/kyverno/
kubectl apply -f ./k8s/
```

You should see sidecars wit the following command

```bash
kubectl -n sentence get pods
```
