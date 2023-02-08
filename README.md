# caktus.k8s-web-cluster

An Ansible role to help configure Kubernetes clusters for web apps.

Supported cloud providers include GCP (GKE), AWS (EKS), Azure (AKS), and Digital
Ocean. The configuration includes installing:

* Nginx Ingress Controller ([Helm Chart](https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx))
* Certificate manager ([Helm Chart](https://github.com/jetstack/cert-manager/tree/master/deploy/charts/cert-manager))
* Let's Encrypt certificate issuers (staging and production)
* For AWS, granting cluster access to IAM users

## License

This Ansible role is released under the BSD License. See the
[LICENSE](https://github.com/caktus/ansible-role-k8s-web-cluster/blob/master/LICENSE)
file for more details.

Development sponsored by [Caktus Consulting Group, LLC](http://www.caktusgroup.com/services>).


## Requirements

* ``pip install openshift kubernetes-validate``
* [helm](https://helm.sh/docs/intro/install/)
  * Important: The version must be [supported by](https://helm.sh/docs/topics/version_skew/#supported-version-skew) the current Kubernetes cluster version).
  * For Caktus projects, refer to the [Developer Documentation](https://caktus.github.io/developer-documentation/developer-onboarding/kubernetes/) for the currently recommended version.


## Installation

1. Add to your ``requirements.yaml``:


```yaml
---
# file: deploy/requirements.yaml

- src: https://github.com/caktus/ansible-role-k8s-web-cluster
  version: v1.0.0
  name: caktus.k8s-web-cluster
```

2. Add the role to your playbook:

```yaml
---
# file: deploy/deploy.yaml

- hosts: k8s
  roles:
    - role: caktus.k8s-web-cluster
```

3. Add role vars configuration (see `defaults/main.yml` for a list of all configurable options):

```yaml

---
# ----------------------------------------------------------------------------
# caktus.k8s-web-cluster: Configure kubernetes cluster for web apps
# ----------------------------------------------------------------------------

k8s_cluster_type: <aws|gcp|azure|digitalocean>
k8s_context: <name of context from ~/.kube/config>
k8s_letsencrypt_email: <email to contact about expiring certs>
k8s_echotest_hostname: <test hostname assigned to your cluster ip, e.g. echotest.caktus-built.com>
# Pin ingress-nginx and cert-manager to current versions so future upgrades of this
# role will not upgrade these charts without your intervention:
# https://github.com/kubernetes/ingress-nginx/releases
k8s_ingress_nginx_chart_version: "3.23.0"
# https://github.com/jetstack/cert-manager/releases
k8s_cert_manager_chart_version: "v1.2.0"
# AWS only:
# Use the newer load balancer type (NLB). DO NOT edit k8s_aws_load_balancer_type after
# creating your Service.
k8s_aws_load_balancer_type: nlb
# List of IAM usernames who should be allowed to manage the cluster
k8s_iam_users: []
```

4. Run ``deploy.yaml`` playbook:

```sh
ansible-playbook -l <host/group> deploy.yaml -vv
```


## Usage documentation

### Testing that Let's Encrypt is working

1. Find the hostname or IP of your load balancer.
2. Add a DNS record for ``k8s_echotest_hostname`` to
   point to this hostname or IP address (switching the record type if needed).
3. Give the record a minute or two to propagate.
4. Add an echotest playbook:

```yaml
---
# file: echotest.yaml

- hosts: k8s
  tasks:
    - name: Install echo test server
      import_role:
        name: caktus.k8s-web-cluster
        tasks_from: echotest
```

5. Run ``echotest.yaml`` playbook:

```sh
ansible-playbook -l <host/group> echotest.yaml -vv
```

6. Give the certificate a couple minutes to be generated and validated. While waiting,
   you can watch the output of:

       kubectl -n echoserver get pod

   When the ``cm-acme-http-solver`` pod goes away, the certificate should be
   validated. Now, navigate to ``k8s_echotest_hostname`` and ensure that you
   have a valid certificate. If you don't, you want to follow the
   [cert-manager troubleshooting](https://docs.cert-manager.io/en/latest/getting-started/troubleshooting.html)
   steps in the documentation. But, be sure to reload a few times, and close the
   browser tab and open a new one to make sure it's really broken, because
   sometimes it takes a few minutes to go through and the browser gets stuck
   with the temporary certificate.

7. You should see the ``*-tls`` secret in the **echoserver** namespace:

       kubectl -n echoserver get secret
       NAME                  TYPE                                  DATA   AGE
       default-token-62pdt   kubernetes.io/service-account-token   3      5m
       echoserver-tls        kubernetes.io/tls                     3      5m

   If not, you may need to re-create the ingress by deleteing and re-applying
   it.

8. When you're done, delete the echotest resources from the cluster. Run:

```sh
ansible-playbook -l <host/group> echotest.yaml --extra-vars "k8s_echotest_state=absent" -vv
```


### Helm charts

After installing the `ingress-nginx` and `cert-manager` Helm charts, you can
view them with `helm list`:

```
❯ helm -n ingress-nginx list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ingress-nginx   ingress-nginx   1               2021-02-11 15:59:27.008281 -0500 EST    deployed        ingress-nginx-3.23.0    0.44.0 

❯ helm -n cert-manager list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
cert-manager    cert-manager    2               2021-02-11 15:41:47.024147 -0500 EST    deployed        cert-manager-v1.2.0     v1.2.0 
```

[helm
upgrade](https://helm.sh/docs/intro/using_helm/#helm-upgrade-and-helm-rollback-upgrading-a-release-and-recovering-on-failure)
has not been tested yet, but the hope is that the helm charts will support
upgrades.
