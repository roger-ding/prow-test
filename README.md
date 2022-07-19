# PROW on Kind Steps

## First create GitHub app
1. Create a GitHub app —> How-To link https://docs.github.com/en/developers/apps/building-github-apps/creating-a-github-app
2. Ensure following permissions are set and save the private key
    1. Repository permissions:
        * Actions: Read-Only 
        * Administration: Read-Only
        * Checks: Read-Only
        * Contents: Read & write
        * Issues: Read & write
        * Metadata: Read-Only
        * Pull Requests: Read & write
        * Projects: Admin
        * Commit statuses: Read & write
      Organization permissions:
        * Members: Read & write
        * Projects: Admin
      1. Subscribe to ALL events
      2. After you saved the app, click "Generate Private Key" on the bottom and save the private key together with the App ID in the top of the page

Create domain and DNS endpoints
1. Create it at freenom.com
2. Get local IP
    1. $> ifconfig -l | xargs -n1 ipconfig getifaddr
3. Two A records (1 blank and 1 WWW with both pointing to local IP address from step above)
4. One CNAME record pointing to desired prow endpoint

Create kind cluster and prow components
1. Create kind cluster
    1. $> export KUBECONFIG=$HOME/kind_kubeconfig_prow
    2. Edit kind/kind-ingress-cluster-config.yaml, uncomment and replace x.x.x.x for ip address lines
    2. $> kind create cluster --name prow --config kind/kind-ingress-cluster-config.yaml --image docker-dev-artifactory.workday.com/envauto-tools/library/kind/node:v1.23.4
2. Create prow namespace, hmac token, GitHub secret, and prowjob CRD
    1. Prow namespace
        1. $> kubectl create ns prow
    2. hmac token
        1. $> openssl rand -hex 20 > hook_secret
        2. $> kubectl create secret -n prow generic hmac-token --from-file=hmac=hook_secret
    3. GitHub secret
        1. $> mv $PATH_TO_PEM_FILE .
        2. $> kubectl create secret -n prow generic github-token --from-file=cert=$PATH_TO_PEM_FILE --from-literal=appid=<ID OF GITHUB APP>
    4. Prowjob CRD
        1. $> kubectl apply --server-side=true -f prow/prowjob_customresourcedefinition.yaml
        2. [Prowjob CRD Link](https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/prowjob-crd/prowjob_customresourcedefinition.yaml)
3. Create rest of prow components
    1. $> kubectl apply -f prow/prow-starter-s3.yaml
    2. [Prow starter component file](https://github.com/kubernetes/test-infra/blob/master/config/prow/cluster/starter/starter-s3.yaml)
4. Create nginx ingress components
    1. $> kubectl apply -f nginx-ingress/nginx-ingress-deploy.yml
    2. $> kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s 
    3. [Nginx deploy for kind clusters](https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/kind/deploy.yaml)

Setup ultrahook
1. Register for ultrahook api on their website
2. $> gem install ultrahook
3. $> echo "api_key: <API KEY GIVEN UPON REGISTRATION>" > ~/.ultrahook
4. ultrahook prow http://prow.<domain_name>q

