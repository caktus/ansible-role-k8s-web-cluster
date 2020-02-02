# caktus.k8s-web-cluster

An Ansible role to help set up Kubernetes clusters for web apps.


# License

This Ansible role is released under the BSD License. See the
[LICENSE](https://github.com/caktus/ansible-role-k8s-web-cluster/blob/master/LICENSE)
file for more details.

Development sponsored by [Caktus Consulting Group, LLC](http://www.caktusgroup.com/services>).


# Requirements

* ``pip install openshift kubernetes-validate``


# Installation

1. Add to your ``requirements.yml``:


```yaml
---
# file: deployment/requirements.yaml

- src: https://github.com/caktus/ansible-role-k8s-web-cluster
  version: init-role  # TODO: remove
  name: caktus.k8s-web-cluster
```

2. Add the role to your playbook:

```yaml
---
# file: playbooks/configure-cluster.yaml

- hosts: k8s_clusters
  roles:
    - role: caktus.k8s-web-cluster
```

3. Add role vars configuration:

```yaml

---
# ----------------------------------------------------------------------------
# caktus.k8s-web-cluster: Configure kubernetes cluster for web apps
# ----------------------------------------------------------------------------

k8s_cluster_type: <aws|gcp|azure|digitalocean>
k8s_context: <name of context from ~/.kube/config>
k8s_letsencrypt_email: <email to contact about expiring certs>

k8s_echotest_hostname: <test hostname assigned to your cluster ip, e.g. echotest.caktus-built.com>
```

4. Run ``playbooks/configure-cluster.yaml``


### Testing that Let's Encrypt is working

1. Find the hostname or IP of your load balancer.
2. Add a DNS record for ``k8s_echotest_hostname`` to
   point to this hostname or IP address (switching the record type if needed).
3. Give the record a minute or two to propagate.
4. Add an echotest playbook:

```yaml
---
# file: playbooks/echotest.yaml

- hosts: k8s_clusters
  tasks:
    - name: Install echo test server
      import_role:
        name: caktus.k8s-web-cluster
        tasks_from: echotest
```

5. Run ``playbooks/echotest.yaml``

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

8. When you're done, delete the echotest resources from the cluster. Run ``playbooks/echotest.yaml --extra-vars "k8s_echotest_state=absent"``
