# Deploy Grafana to AKS with Azure AD Integration

[Grafana](https://grafana.com/) is the open source analytics and monitoring solution for every database. I am going to introduce how to deploy grafana to Azure K8S.

## Create AKS Service Resource
Take care of K8S version, which will impact TLS. I use the default version `1.15.10`.

## Install Helm and tiller
Use [Azure cloud shell](https://docs.microsoft.com/en-us/azure/cloud-shell/overview) to install [Helm and tiller](https://helm.sh/).

Pull the AKS credentials from the AKS cluster.

```
az aks get-credentials --resource-group AdsFastDashboard --name AdsFastBI
```

Install Helm and tiller
```
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

# serviceaccount/tiller created
# clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

## Use Helm to deploy an NGINX ingress controller

Create a namespace for ingress resources.
```
kubectl create namespace ingress
```

Deploy NGINX.
```
helm install stable/nginx-ingress --namespace ingress --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux --generate-name
```

Update DNS Name.
```
# Show Ingress services
kubectl get service -l app=nginx-ingress --namespace ingress

/*
NAME                                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
nginx-ingress-1586706416-controller        LoadBalancer   10.0.121.215   40.91.111.33   80:30974/TCP,443:30442/TCP   51s
nginx-ingress-1586706416-default-backend   ClusterIP      10.0.247.13    <none>         80/TCP                       51s
*/

# Public IP address of your ingress controller
IP="40.91.111.33"

# Name to associate with public IP address
DNSNAME="test"

# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

# Update public ip address with DNS name
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME

```

## Install cert-manager
Secure the traffic. [Cert-manager](https://docs.cert-manager.io/en/latest/) will handle the certification requests and issue the SSL certificates automatically via [Let’s Encrypt](https://letsencrypt.org/).

```
# Install the CustomResourceDefinition resources separately
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.1/cert-manager.crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v0.14.1

# Configuring Issuer. Create ./issuer_production.yaml and add the following configuration.

apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: ingress
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: weiyyao@microsoft.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx

# Apply the issuer.
kubectl apply -f issuer_production.yaml

# Check info of the issuer
kubectl describe issuer letsencrypt-prod
```

## Install grafana

Create a file called `./custom_values.yaml` and then configure it.

```
## PERSISTENCE STORAGE CONFIGURATION
persistence:
        type: pvc
        enabled: true
        accessModes:
          - ReadWriteOnce
        size: 10Gi
        finalizers:
          - kubernetes.io/pvc-protection
#### LOAD PLUGIN
plugins:
          - grafana-clock-panel
          - grafana-piechart-panel
## AZURE AD Integration https://community.grafana.com/t/some-questions-on-authentication-with-azure-ad/18133/7
grafana.ini:
        auth.generic_oauth:
          name: "Azure AD"
          enabled: true
          allow_sign_up: true
          client_id: [CLIENT ID SERVICE PRINCIPAL] # NEED TO CHANGE
          client_secret: [CLIENT KEY SERVICE PRINCIPAL] # NEED TO CHANGE
          scopes: "openid email name"
          auth_url: https://login.microsoftonline.com/[TENANT ID]/oauth2/authorize # NEED TO CHANGE
          token_url: https://login.microsoftonline.com/[TENANT ID]/oauth2/token # NEED TO CHANGE
### REPLY URL SERVER
        server:
          root_url: # NEED TO CHANGE TO CUSTOM URL !! MUST MATCH REPLY URL ON SERVICE PRINCIPAL !!
```

We are ready to install grafana.
```
helm upgrade --install grafana --namespace ingress  stable/grafana -f ./custom_values.yaml
```

## Apply routing.
We need to tell NGINX to route all traffic on port 80 from the cluster to Grafana. Create a file called `./custom_values.yaml` and configure it as follows.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
        name: grafana-ingress
        namespace: ingress
        annotations:
          kubernetes.io/ingress.class: nginx
          cert-manager.io/issuer: letsencrypt-prod
          nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
        tls:
        - hosts:
          - test.westus2.cloudapp.azure.com
          secretName: tls-secret
        rules:
        - host: test.westus2.cloudapp.azure.com
          http:
            paths:
            - backend:
                serviceName: grafana
                servicePort: 80
              path: /(.*)

```
And then apply it.
```
kubectl apply -f ingress_route.yaml
```

## Ref
[Azure Monitor – Install AKS Monitoring Grafana Dashboard With Azure AD Integration Using Helm](https://www.stefanroth.net/2019/10/18/azure-monitor-helm-install-aks-monitoring-grafana-dashboard-with-azure-ad-integration/)

grafana admin pwd:
JVIX5aEhONbGx3ysYeRpumDyTT8lHEOHfQWIcZTN

