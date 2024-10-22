# OpenFaaS on DigitalOcean Kubernetes

This guide is for provisioning a cluster manually, or with the new 1-Click Kubernetes Marketplace installer.

The second part of this document is what will be presented to a user, just like the [page for Linkerd2](https://marketplace.digitalocean.com/apps/linkerd-beta).

* [Sign-up as a new customer to get free credits](https://m.do.co/c/2962aa9e56a1)

## Listing for DigitalOcean Marketplace

OpenFaaS &reg; is an award-winning open source project that makes it easy for developers to deploy applications to Kubernetes in a Serverless-style. Any microservice, API, binary, or function can be packaged and deployed in a very short period of time. Once [a workload](https://docs.openfaas.com/reference/workloads/) is deployed via the OpenFaaS [CLI](https://docs.openfaas.com/cli/install/), API or UI metrics will be tracked and used to auto-scale your code in response to demand.

OpenFaaS comes with built-in auto-scaling, [detailed metrics](https://docs.openfaas.com/architecture/metrics/) and [queue-processing](https://docs.openfaas.com/reference/async/). You can take advantage of pre-made functions from the Function, or a series of templates for Functions or Microservices covering a wide range of languages such as C#, Java, Go, Ruby, PHP, and more.

Your workloads can be accessed through the OpenFaaS gateway or triggered by a number of [event sources](https://docs.openfaas.com/reference/triggers/) such as Kafka, RabbitMQ, Redis and Cron.

The project is built around open interfaces that can be extended easily. Tutorials and guides can [help you enable TLS](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/), [setup custom domains](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/#20-ssl-and-custom-domains-for-functions), CI/CD, OAuth2, multi-user support, and many other features.

You can find out more about OpenFaaS at [https://www.openfaas.com/](https://www.openfaas.com) or take the [free workshop online](https://github.com/openfaas/workshop/).

## Manual provisioning of DOKS cluster via `doctl`

* First, get the `doctl` binary and run `doctl auth`.

  [https://github.com/digitalocean/doctl](https://github.com/digitalocean/doctl)


  ```sh
  doctl k8s cluster create \
      --region lon1 \
      --version 1.14.5-do.0 \
      --tag openfaas \
      --size s-1vcpu-2gb  \
      --count 3 \
      openfaas-cluster
  ```

  > Note: you can use `doctl k8s options versions` to get the latest version:

## Configure `kubectl` locally

* Point `kubectl` to your cluster

  In order to start using OpenFaaS, you will need to find your password and LoadBalancer IP. To do this, you will need to use `kubectl` and point it at your new cluster.

  ```sh
  doctl k8s cluster list

  f8d01a21-ef2a-4b93-b05f-6774e77a86e5    openfaas-cluster          lon1      1.14.5-do.0    running    openfaas-cluster-default-pool

  doctl k8s cluster kubeconfig save openfaas-cluster

  ```

  Now switch into the context of the cluster:

  ```
  kubectl config get-contexts
  *         do-lon1-openfaas-cluster         do-lon1-openfaas-cluster         do-lon1-openfaas-cluster-admin         

  kubectl config set-context do-lon1-openfaas-cluster
  ```

## Manual setup of OpenFaaS (without `tiller`)

```sh
cd /tmp/
mkdir -p manifests

curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
helm init --client-only

helm fetch \
  --repo https://openfaas.github.io/faas-netes/ \
  --untar \
  --untardir ./charts \
    openfaas

sed -ie s/NodePort/LoadBalancer/ ./charts/openfaas/values.yaml

export NAMESPACE="openfaas"

helm template \
  --namespace $NAMESPACE \
  --values ./charts/openfaas/values.yaml \
  --output-dir ./manifests \
    ./charts/openfaas

# Create namespaces: openfaas, openfaas-fn

kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

# generate a random password
PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)

kubectl -n openfaas create secret generic basic-auth \
--from-literal=basic-auth-user=admin \
--from-literal=basic-auth-password="$PASSWORD"

echo $PASSWORD > password.txt

kubectl apply -f ./manifests/openfaas/templates/
```

## OpenFaaS for the DigitalOcean Kubernetes Marketplace (1-Click)

OpenFaaS has been deployed and is now fully-functioning. During the installation a password was created for your OpenFaaS Gateway along with a LoadBalancer to provide access to your gateway.

### Configure `kubectl` locally

* Point `kubectl` to your cluster

  In order to start using OpenFaaS, you will need to find your password and LoadBalancer IP. To do this, you will need to use `kubectl` and point it at your new cluster.

  ```sh
  doctl k8s cluster list

  f8d01a21-ef2a-4b93-b05f-6774e77a86e5    openfaas-cluster          lon1      1.14.5-do.0    running    openfaas-cluster-default-pool

  doctl k8s cluster kubeconfig save openfaas-cluster

  ```

  Now switch into the context of the cluster:

  ```
  kubectl config get-contexts
  *         do-lon1-openfaas-cluster         do-lon1-openfaas-cluster         do-lon1-openfaas-cluster-admin         

  kubectl config set-context do-lon1-openfaas-cluster
  ```

### Log into OpenFaaS

* Install the OpenFaaS CLI:

  ```sh
  curl -sLSf https://cli.openfaas.com | sudo sh
  ```

  > Note: the CLI is also available via `brew install faas-cli`

* Find your LoadBalancer IP:

  You can use the LoadBalancer's IP to log into OpenFaaS, operate the CLI, and to view the UI.

  ```sh
  export OPENFAAS_URL=$(kubectl get svc -n openfaas gateway-external -o jsonpath='{.status.loadBalancer.ingress[*].ip}'):8080
  echo Your gateway URL is: $OPENFAAS_URL
  ```

  >  Note: your DigitalOcean LoadBalancer may take a few minutes to come up, so if you're not seeing an IP, try again.

* Get your password

  ```sh
  echo $(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode) > password.txt 

  echo Your admin password is: $(cat password.txt)
  ```

  * Now use the password to authenticate the CLI:

  ```sh
  cat password.txt | faas-cli login --username admin --password-stdin
  ```

  > Note: you can use this command at any time to retrieve your password

### Check that everything worked

* Deploy a function

  ```sh
  faas-cli store list

  # Find one you like

  faas-cli store deploy nodeinfo

  # List your functions

  faas-cli list --verbose

  # Check when the function is ready

  faas-cli describe nodeinfo

  Name:                nodeinfo
  Status:              Ready

  # Invoke the function using the URL given above, or via `faas-cli invoke`

  echo | faas-cli invoke nodeinfo
  echo -n "verbose" | faas-cli invoke nodeinfo
  ```

* Try the UI

  You can access the UI at the same URL of the gateway, so try this URL:

  ```
  echo http://$OPENFAAS_URL
  ```

  ![](https://github.com/openfaas/faas/raw/master/docs/inception.png)

  *Demo of the inception function*

### Take the wheel

* Learn OpenFaaS

  Now it's over to you to start learning about Serverless with OpenFaaS and Kubernetes using the hands-on workshop.

  [https://github.com/openfaas/workshop](https://github.com/openfaas/workshop)

* Add TLS for your OpenFaaS Gateway

  You can add TLS to your OpenFaaS Gateway URL with the following documentation: [1.0 SSL for the Gateway](https://docs.openfaas.com/reference/ssl/kubernetes-with-cert-manager/)

* Join OpenFaaS Insiders

  OpenFaaS Insiders get regular email updates on project news and developments including videos, blogs, and early-access to the latest features. Join by becoming a sponsor on GitHub at any tier.

  [GitHub Sponsors](https://github.com/users/alexellis/sponsorship)

* Need help?

  Visit [OpenFaaS Slack](https://slack.openfaas.io/)
