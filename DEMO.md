# Demo Walkthrough

## Acknowledgements 
Continuation of the awesome work by everett-toews.
* https://gist.github.com/everett-toews/ed56adcfd525ce65b178d2e5a5eb06aa

## Watch DevOps with Kubernetes and Helm Demo video (45 min)

## Watch Jenkins Demo

https://www.youtube.com/watch?v=eMOzF_xAm7w

# Prerequisites
kubectl access to a Kubernetes 1.4+ cluster

If need be, you can run Kuberentes from within Docker for Desktop. More information [here](https://blog.docker.com/2018/07/kubernetes-is-now-available-in-docker-desktop-stable-channel/). This will give you access to a local Kubernetes cluster to use with the walkthrough here.

## Fork repo
``` 
https://github.com/jldeen/croc-hunter#fork-destination-box
```

# Setup Tiller Service account

```
kubectl apply -f rbac-config.yaml
```

# Install Helm (Mac OS)

```
brew install kubernetes-helm
helm init --service-account tiller
helm repo update
```

## Install Cert Manager
```
helm install --name cert-mgr stable/cert-manager
```

In order to begin issuing certificates, you will need to set up a ClusterIssuer or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them can be found in the cert manager documentation:

https://cert-manager.readthedocs.io/en/latest/reference/issuers.html

## Install a ClusterIssuer

```
kubectl apply -f cluster-issuer.yaml
```

## Install Nginx ingress chart
```
helm install stable/nginx-ingress

Follow the notes from helm status to determine the external IP of the nginx-ingress service
```
To learn more about ingresses, you can checkout my blog post [here](https://jessicadeen.com/tech/microsoft/aks-and-helm-charts-ingress-controllers/). 

## Add a DNS entry with your provider and point it do the external IP
```
blah.test.com in A <nginx ingress svc external-IP>

or *.test.com in A <nginx ingress svc external-IP>

```

## Set Codefresh cluster rolebinding (if using Codefresh)
```
kubectl create clusterrolebinding default-admin --clusterrole cluster-admin --serviceaccount=default:default 
```

## Set Jenkins cluster rolebinding (if using Jenkins)
```
kubectl create clusterrolebinding jenkins --clusterrole cluster-admin --serviceaccount=jenkins-demo:default 
```
# Below is just for Jenkins
## Update jenkins.values.yaml
```
Find and replace `jenkins.acs.az.estrado.io` with the DNS name provisioned above

helm --namespace jenkins --name jenkins -f ./jenkins-values.yaml install stable/jenkins

watch kubectl get svc --namespace jenkins # wait for external ip
export JENKINS_IP=$(kubectl get svc jenkins-jenkins --namespace jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
export JENKINS_URL=http://${JENKINS_IP}:8080

kubectl get pods --namespace jenkins # wait for running
open ${JENKINS_URL}/login

printf $(kubectl get secret --namespace jenkins jenkins-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode) | pbcopy
```

## Add credentials for private container registry

Create username and password credentials in Jenkins Credential Manager per Jenkinsfile.json (project config)

Reference to the secret name must also be added to the chart values.yaml or set on install.

Go to ```{JENKINSURL}/credentials```

Click System

Click Global credentials

Click add credentials

or

Go to ```{JENKINSURL}/credentials/store/system/domain/_/newCredentials``` directly

Ensure ID = line17 of Jenkinsfile.json

## Login and configure Jenkins and setup pipeline
```
# username: admin
# password: <paste>

If you're not using quay you can configure this to alternate locations in Jenkinsfile.json
# Credentials > Jenkins > Global credentials > Add Credentials
#   Username: lachie83
#   Password: ***
#   ID: quay_creds
#   Description: https://quay.io/user/lachie83

# Open Blue Ocean
# Create a new Pipeline
# Where do you store your code?
#   GitHub
# Connect to Github
#   Create an access key here
#     Token description: kubernetes-jenkins
#   Generate token > Copy Token > Paste back in Jenkins  
# Which organization does the repository belong to?
#   lachie83
# Create a single Pipeline or discover all Pipelines?
#   New pipeline
# Choose a repository
#   croc-hunter
# Create Pipeline
```

## Watch Jenkins build agents run
```
kubectl get pods --namespace jenkins
```

## Update Org to build PRs
```
# Classic Jenkins
# lachie83 (GitHub org)
# Configure
# Advanced
#   Build origin PRs (merged with base branch)
# Save
```


## Setup Webhook in Github
``` 
printf ${JENKINS_URL}/github-webhook/ | pbcopy

# https://github.com/lachie83/croc-hunter/settings/hooks
# Add webhook
#   Payload URL: <paste>
# Which events would you like to trigger this webhook?
#   Send me everything.
# Add webhook
```

## Update croc-hunter ingress records
```
Update croc-hunter.acs.az.estrado.io in charts/croc-hunter/values.yaml

Configured DNS A record to point to the Nginx Ingress IP
Once master branch is pushed it should be available at that name
```


## Pushing Game update
```
git checkout dev
sed -i "" "s/game\.js/game2\.js/g" croc-hunter.go
git commit -am "Game 2"
git push
```

### Building and releasing
```
open ${JENKINS_URL}/blue/organizations/jenkins/lachie83%2Fcroc-hunter/activity/

open https://github.com/jldeen/croc-hunter

# master builds and deploys new version
```
