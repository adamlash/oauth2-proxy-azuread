# oauth2-proxy-azuread
Oauth2 Proxy on K8s with a Demo App and on Azure. The below will assume a FRESH cluster has been made, but you can also do this on an existing one, just add or remove where applicable (eg ingress controller). 

This is also for the nginx ingress contoller so if you are using something else (Traefik etc.). 

The Final point is the below is using the **dummy builtin self-signed K8s Certs as well**, you may want to look at using something like cert-manager (https://cert-manager.io/docs/installation/kubernetes/) for actually creating *real* certificates with letsencrypt! This is just a quick demo mainly focussing on the oauth2-proxy config, and all the cert stuff is just incidental...


## Step 0 - Get a Domain
Some Preqs..
- You will need a real Domain for this, a subdomain works fine too, for this example I will use `test.qill.in`. So anywhere you see this you may wish to replace your own domain.
- You will also want to configure some DNS records to your Ingress Controller Public IP (will be setup below), for ease of use I will use an A Record Wildcard: `*.test.qill.in` which will point to the Load Balancer IP of my AKS Cluster.


The domains we will use for this will be:
- App 1: echo1.test.qill.in
- Auth Proxy: auth.test.qill.in

The auth proxy will be used by the oauth2-proxy ingress and the apps for their relevant ingresses.

## Step 1 - Register App in AAD
Creating the Registration in AAD
### 1. Create App Registration in Azure Active Directory:
- Browse to Azure Active Directory > App Registrations
- Create New Registration
- Configure with Name, Scope and Redirect URI of https://auth.test.qill.in/oauth2/callback (denote the URL here is the **Auth Proxy** with the callback prepended.)

### 2. Retrieve Information for App
- Browse to Certificates and Secrets > Add 'New Client Secret'
- Save this secret, and denote the following information under the 'Overview' Tab for later
    - Secret Key
    - App (Client) ID
    - Directory (Tenant) ID

## Step 2 Install Nginx Ingress (If one does not exist already)
As per the https://kubernetes.github.io/ingress-nginx/deploy/#using-helm
### 1. Create Namespace and Add the Repo/Update Helm Repo
```
kubectl create ns ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
### 2. Add the Nginx Ingress Repo and install (using Helm)
```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx
```

### 3. Add your DNS Records
- Denote the External IP of your Ingress Controller (`kubectl get svc --namespace ingress-nginx`)
- In this case, I am using the Azure DNS and I made a wildcard record for all *.test.qill.in to point to this IP.



## Step 3 - Install Test App
1. Deploy the Test App for Testing, all this does is print a string into the browser.
```
kubectl apply -f echo-app/echo1.yaml
```

2. Create Ingress (non authed) and Test our Apps work

**edit ingress/echo_noauth_ingress.yaml and replace my URLS (`echo1.test.qill.in`) with yours.**
```
kubectl apply -f ingress/echo_noauth_ingress.yaml
```

Should be able to browse to echo1.test.qill.in with a reply (and a dummy K8s cert, again if you want real certs check out cert-manager!)

## Step 4 - Install and Configure oauth2-proxy from Helm
This will be installing the oauth2-proxy for the Azure App


### 1.  Collate and Deploy Secrets

oauth2-proxy wants the azure-tenant, client-id, client-secret and cookie-secret. To do this we need to create a secret configmap in k8s using the values from Step 1.


- azure-tenant = Directory (Tenant) ID
- client-id = App (Client) ID
- client-secret = Secret Key
- cookie-secret = can be generated with (python3) `python -c 'import os,base64; print(base64.b64encode(os.urandom(16)))'`

Populate where applicable + Run the following command to create a K8s Secret. The proxy will use these:
```
kubectl create secret generic oauth2-proxy-creds \
    --from-literal=cookie-secret=asdf \
    --from-literal=client-id=asdf \
    --from-literal=client-secret=asdf \
    --from-literal=azure-tenant=asdf
```

### 2. Install oauth2-proxy helm chart

**Edit the relevant values in oauth2-proxy/oauth2-proxy-config.yaml, make sure to replace my URLS (`auth.test.qill.in`) with yours.**

Once edited, install the helm chart:
```
helm upgrade oauth2-proxy --install stable/oauth2-proxy \
--reuse-values \
--values oauth2-proxy/oauth2-proxy-config.yaml
```

## Step 5
Configure the Ingress to use the oauth2 proxy.

### 1. Deploy the Ingress
This will overwrite the existing ingress config with some new rules to force any requests for the websites to push over to the auth site and then ask for AAD Login.
```
kubectl apply -f ingress/echo_auth_ingress.yaml
```

## Testing
1. Browse to echo1.test.qill.in
2. You will be redirected to auth.test.qill.in and and from there, auth will be checked
3. If no auth, the oauth2-proxy will redirect you into AAD login, and ask you to login/authenticate
4. One Authed, it will then redirect you back to your original site, in this case echo1.test.qill.in
5. All future sessions will be authed accordingly

If you are also already authenticated to AAD, you may be asked to just approve this login and pushed straight through.

*Please one again denote the certs are the dummy K8s ones, so cert error are expected for this demo!*
