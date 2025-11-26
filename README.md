# AarhusAI GitOps template

To install AarhusAI with this gitops template, you need to have a Kubernetes cluster with Helm 3 installed
and [ceph](https://rook.io/docs/rook/v1.12/Getting-Started/quickstart/) filesystem installed.

__NOTE__: Maybe Deranged can give some more information about the Ceph installation.

This installation documentation must be followed and executed in the correct order as described below, because the
different services require other services to be ready before they can be installed. In particular, ArgoCD needs to be
installed before Argo resources and sealed secrets, before Prometheus and Grafana etc.

The application requires you to generate new api-keys etc. and use sealed secrets to store them before a given service
is installed, and some need the keys/secrets from one to communicate with another.

Also, because all configuration is stored in GitOps as code, you need to update end-point and urls to match your domain.

## Bootstrap

The first step is to create a new GitHub repository based on this template repository.

### ArgoCD

For continues development and using GitOps to handle updates and configuration
changes. [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) needs to be bootstrapped into the cluster.

Configuration to change need in `applications/argo-cd/values.yaml`:

```yaml
global:
  domain: <NAME>
```

Then one needs to install ArgoCD using the following commands:

```shell
helm repo add argocd https://argoproj.github.io/argo-helm
helm repo update
cd applications/argo-cd
helm dependency build
kubectl create namespace argo-cd
helm template argo-cd . -n argo-cd | kubectl apply -f -
```

You can check that ArgoCD is running by opening the ingress url:

```shell
kubectl get ingress -A
```

You can log into the webbased user interface as admin and by getting the password with this command:

```shell
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

As an optional configuration, later one, we can install Authentic as SSO provider for the internal services in the
cluster.

### Argo resources

Now we can install the Argo resources that define the applications and their configuration. But first we need to
change the repository url to our own repository.

Edit `applications/argo-cd-resources/templates/applications.yaml`:

```yaml
spec:
  source:
    repoURL: https://github.com/aarhusai/<YOU REOP>.git
```

One also needs to change the configuration `sourceRepos` in every
`applications/argo-cd-resources/templates/projects/*.yaml` file for each application to argo knows with repos to stay in
sync with.

The last step in Argo installation is to install the resources:

```shell
cd applications/argo-cd-resources/
helm template argo-cd-resources . -n argo-cd | kubectl apply -f -
```

This will install all the application from `applications/argo-cd-resources/values.yaml` that are set to
`automated: true` which is alle the applications that do not need configuration changes. All the other applications will
need to have their configuration updated and changed to automatically sync with the repository.

## Observability

Loki, Tempo and Alloy are automatically installed by ArgoCD above as they do not require configuration changes. But
Grafana needs some basic domain name and url configuration:

```yaml
grafana:
  ingress:
    hosts:
      - <FQDN - grafana.example.com>
    tls:
      hosts:
        - <FQDN - grafana.example.com>

  additionalDataSources:
    grafana.ini:
      server:
        domain: <FQDN - grafana.example.com>
        root_url: https://<FQDN - grafana.example.com>
```

Change the `automated` to true in `applications/argo-cd-resources/values.yaml` for the `prometheus-stack` application.

## Sealed Secrets

Before installing the rest of the applications, we need to install `kubeseal` to generate sealed secrets that can be
safely commited to public GitHub repositories. (Yes, one can reconfigure ArgoCD to use private repositories, but one
would still use sealed secrets to protect secrets. Accidental exposure happens, and human errors occur)

Read more about sealed secrets [here](https://github.com/bitnami-labs/sealed-secrets).

### Install kubeseal

Follow the guide [here](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal) to install
`kubeseal` based on which system you are on.

Next, download the public certification:

```yaml
kubeseal --fetch-cert > public-cert.pem
```

Optionally use the ingress endpoint (`--cert http://sealed-secrets.<FQDN>/v1/cert.pem`). To enable the ingress end-point
for sealed secrets, edit `applications/sealed-secrets/values.yaml` and change the `ingress.enabled` to true. Commit and
push the changes to the repository and with for the re-deployment of sealed secrets by ArgoCD.

### Seal a secret

You can place the unsealed seceret anywhere you want, but this repository has the path `applications/**/local-secrets`
in git-ignore. So with this in mind the commands would be in the form:

```shell
kubectl create -f local-secrets/<NAME_OF_APPLICATION>-secret.yaml --dry-run=client -o yaml | \ 
 kubeseal --cert public-cert.pem --format yaml > templates/sealed-<NAME_OF_APPLICATION>-secret.yaml
```

With an unsealed secret like this (here for litellm):

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: litellm-secrets
  namespace: litellm
stringData:
  PROXY_MASTER_KEY: <KEY>
  CA_VLLM_LOCAL_API_KEY: <KEY>
```

## vLLM (Mistral 24b model)

If you want to use a local hosted model on local hardware in the cluster, you need to configure the vLLM application.
Which requires you to first create a https://huggingface.co/ account, from where the models are downloaded (it's free
simple create an account a generate an access token).

Create an unsealed secret for `hf-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: hf-secret
  namespace: vllm
stringData:
  TOKEN: <TOKEN>
```

Seal it:

```yaml
kubectl create -f local-secrets/hf-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-hf-secret.yaml
```

Next we create a secret for the vLLM API, this key is used internally in the cluster to communicate with the vLLM
service, and you should generate a random key:

```shell
echo "sk-vllm-$(cat /dev/urandom | tr -dc 'a-f0-9' | fold -w 32 | head -n 1)"
```

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  creationTimestamp: null
  name: vllm-secret
  namespace: vllm
stringData:
  KEY: <GENERATED API-KEY>
```

Seal it:

```yaml
kubectl create -f local-secrets/vllm-secret.yaml --dry-run=client -o yaml | \
kubeseal --cert public-cert.pem --format yaml > templates/sealed-vllm-secret.yaml
```

Change the `automated` to true in `applications/argo-cd-resources/values.yaml` for the `vllm` application. Add, commit
the changes to have argo deploy wllm automatically. __Note__, that it may take some time to deploy vLLM for the first
time as it downloads the model from huggingface.com and loadeds it into GPU memory.

-----------------------------

-----------------------------

-----------------------------

## Backup (S3)

__NOTE__: Maybe Deranged can give more information about the S3 backup installation.

## SSO

https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal

Only for internal SSO.

### ArgoCD

argo-cd/templates/sealed-oidc-authentik-client-secret.yaml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: oidc-authentik-client-secret
  namespace: argo-cd
spec:
  encryptedData:
    oidc.authentik.clientSecret: AgCrTymTzo5MvUTprglpRRtGzgMo8R7UGAuu4iOoiev0ayQBGLOZnGM5PhLFVQ+ZlgSXZjOR5L5e3iySp/wn4YzTy9z04WVvrk/rf3Y1TbIZLV8zJYc77Av2TgWv3F/XPJ69nTbUNwOzI3Mn5yMh+1CwkEI3dzvW76kbrphfKe0HOmyuhSSGH/6aW40x/7S6XjGGLtfqqTpOdfuAwTPs+gojU83RqwnU+Fj9gzWHqVtBSEN/dtsr75btp38XiTWXmo5dkp0uYJayCHIm3F45UzuBCU2dgKr6DZufpeK69/qtUt0ngPf2qOXjprRKLWywTm94DaZ1jehXaR5u9lk8DmISxfaoZuWo3Ls5iFUeG6wNxPYi8Xc9kKxbnrXFAGMBljy16EaRSPNYK4cDvUPHfbRllZvctozAt+bHUx6Hp9m64T7OmtcyFnNCdakYXq7T2PmyJn1nY+DqhnU6E7pJBxyqLlrYoqNfaoRQYgGC1B5Z9fjP2IuZ5nxroLHj9ldjvF5dAvSOlNrb+joUcA3EYlk4aol+Npl1Obd1aFPEFVD3sQj0l3cKr4z0qa4EozWbSV8Tgvl0lo7o97nlZGUbtko+hMzdG9zwTHkYvWipU8aDXBvBCsISJYt3MiGRVdBoHlZwYtYuqkTrusf7ZhSU6H/iwB3lIexEs8nV4C5+1aUXSOGHbbC82uDqpVj9dHYdmHV5bmxHYN+MAMva2BDjd0lWDx7oH98KsUguG7tMwrBKmFFHHzhWu2F0WZSuh1IFyCMRmO19TLhDTJL36JS8EoCm/ye0dfzGQ7lW2DOmfeqS2fE2Jh7v3I5YEd8IkIvGejF9ATmhDDt12/0+t5JZXSOzdzY3WBCtUhwdd8MCXO0CwA==
  template:
    metadata:
      creationTimestamp: null
      labels:
        app.kubernetes.io/part-of: argocd
      name: oidc-authentik-client-secret
      namespace: argo-cd
    type: Opaque
```

Configuration to change need in `applications/argo-cd/values.yaml`:

```yaml
global:
  configs:
    cm:
      oidc.config: |
        name: AarhusAI SSO
        issuer: https://auth.kom1.deranged.dk/application/o/argocd-kom1/
        clientID: bJDCFviyQcUXVIr2uXqbos5l6ZjIjBZlY0j647T4
        clientSecret: $oidc-authentik-client-secret:oidc.authentik.clientSecret
        requestedScopes: [openid, profile, email, groups]
```

### Grafana/Prometheus

prometheus-stack/templates/sealed-auth-generic-oauth-secret.yaml

```yaml
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: auth-generic-oauth-secret
  namespace: monitoring
spec:
  encryptedData:
    client_id: AgAsCr1A2QChQSocZtiy8dZPfsA2AtRU7CIFH/5eaVwRB0BZJ8NbpifFC+MJs7nKZOjeEJB3A+x/AB4WyJvf+dSgiYWpKmtMQjx+DobtErIfEQyQq6rdpjYzefNG9KdNWA5j5LspcGhrcXyq3kJIKNZHgn0PPALU3Vo3wGkoCj5MBxcMi1QoKaSOdbwh75JZN5ASrgRi1NHY17izIPOB7bEKDR+LTLe1AW9bj0xZAdCOg5MSde9W7gSBmVErdIWvvpd3H1/1IkDHHJ9hvhGjyLDS8kGlIdXmAYG+14C0sRgSYeV6lTpirv7sJkprYIafIvXIhn8M3y8aXtmMUwyW6n3kqpQQfuQdm9p+g345zLwC/EJU3oizosI8N8AAJRgMGsZFT9W2DGApCp+j0TQBPk/RiQRTnaA7BDdeLoYUHSk0tFaQdsMZ9jfaRD/mgw3xZS0ds++uJUKTtmOdBbIwOGKif9UAP/OfYNMfVD8PJTRg04VwZ+X88d9DcGAbuLO3knEa8+HjpTSYtaP+raVezM1LqhfgIYlTZFJlTSLZj50KQjhd/LTFD2nu79t2c6rCW/1qw4+Nd+AiI5NNNl9l2t4yJycCswGIFLoxlmRW7KiogjMvGHCFPniPS5K3nY37LtvcVxqa9Qr4DIztcr0PBIq5O8WQ2njZlvsDHSXBS6eLFcrO2ojcnLFzMQZ6i8GGW9vyYRjC0CylCPjFUWB4pvcLW3jChsp7eXWWwFDiqH9+L0tI/ntKT5O2
    client_secret: AgAWOHFBqsp5tYjDLrJeBeBO8ehvY8HufAyrdBM+MT/rl1dSxEVrK2U1FtP1w1GAd4IdKJ+zI961jZs9iz3yo3m4VNEqEwOpEhhYALWJ6ah4rCS3uj210vYW2FwdTjMv2YtTydw6IFMT818yn0Y/ndt7DMtlmqLAJNiT+JltRglZPnIlMhG59hAri/OIWN/KC8MZjNwlwUFwlwTxJ0+OpNCntpZDaMFTo55jBM6pmNj/+MQJ6Mb7tAlNabWfVzDWqwOq6le5GcinbMUwum+7ztHy8oKQ8DkLMHZdHB0mv5tTqpm337MdrVDkqcvWvu7MguZvI5DBVogGUH9O3Ae3p8rKpPRwdup9gtTLEPpUi5DLv12+Ka3NAefWzVIY1QRQlwCK6YPxUoQ6TrbD7LEK4nCeuOJot7u/oo65kiicI5vnQYjMc4VFOBsCgi391/Sd8baSH68Hd9P8aRzn9aBCGkC4NO91GRCaj1OU/m5gkRKk8+k2PwWP70xzo27ZiDKz/Yc/BGeI0EbvlAqsfJflfWF88blUMF64mbSPButzmIzbCRWtAi7z02KhOLFDwBHTnImRl5cgxGSR0mT89yk9W5wLXNWAsmzBUzBAxlOHHVOgEr8yHbBg6txj5PVcjbbu9JFDQAm1oGfuOm6J+/g/drJk9SkCMPbx1u+xKTbAUEdSmdtnM4/fimhKjmcKqM/1OU0tkAq8gSYaK7o38BQzVTfsefFMaHjg/EnxbQ3rnQJsvr7zf/eVV/tIdrhQG5RGUjQSLGG2WMZLplmTtVVqaAZsBgp72CxLemk25oQ8CyytU/Shb+R9AC4MOEM6aM1JmGWs4QbmfsVmG+qwF/s95ho2mQMKCGPYMPCnObrqxms/tA==
  template:
    metadata:
      creationTimestamp: null
      name: auth-generic-oauth-secret
      namespace: monitoring
    type: Opaque
```

prometheus-stack/values.yaml

```yaml
grafana.ini:
  server:
    auth.basic:
      disable_login_form: true
    auth:
      signout_redirect_url: https://auth.<FQDN>/application/o/grafana-kom1/end-session/
      oauth_auto_login: true
    auth.generic_oauth:
      name: AarhusAI SSO
      enabled: true
      client_id: $__file{/etc/secrets/auth_generic_oauth/client_id}
      client_secret: $__file{/etc/secrets/auth_generic_oauth/client_secret}
      scopes: openid profile email
      auth_url: https://auth.<FQDN>/application/o/authorize/
      token_url: https://auth.<FQDN>/application/o/token/
      api_url: https://auth.<FQDN>/application/o/userinfo/
      # Optionally map user groups to Grafana roles
      role_attribute_path: contains(groups, 'grafana') && 'Admin' || 'Viewer'
extraSecretMounts:
  - name: auth-generic-oauth-secret-mount
    secretName: auth-generic-oauth-secret
    defaultMode: 0440
    mountPath: /etc/secrets/auth_generic_oauth
    readOnly: true
```