# caktus.k8s-web-cluster


## Changes


### v1.0.0 on TBD

**BACKWARDS INCOMPATIBLE CHANGES:**

* Use Helm to install `ingress-nginx`
  * `ingress-nginx` controller upgraded from `0.26.1` to `0.44.0` (via `3.23.0`
    helm chart release)
* Use Helm to install `cert-manager`
  * `cert-manager` controller upgraded from `v0.10.1` to `v1.2.0`
  * The accompanying
    [caktus.django-k8s](https://github.com/caktus/ansible-role-django-k8s/) role
    must also be updated to >`v0.0.11` to restore certificate validation.
* You must follow the [Digital Ocean instructions](https://www.digitalocean.com/docs/kubernetes/how-to/configure-load-balancers/#accessing-by-hostname-annotation) and set a hostname via `k8s_digitalocean_loadbalancer_hostname` to keep PROXY protocol enabled on Digital Ocean (required to see real client IP addresses).

Upgrade instructions:

1. First, purge the old cert-manager and create a new ingress controller in a new namespace:

   ```
   # Install new ingress controller, but don't delete the old one yet
   k8s_install_ingress_controller: yes
   k8s_ingress_nginx_namespace: ingress-nginx-temp
   k8s_purge_ingress_controller: no

   # Don't install a new cert-manager, but do delete the old one
   k8s_install_cert_manager: no
   k8s_purge_cert_manager: yes
   ```

   If you don't wish to make two DNS changes, you may find it helpful to set
   `k8s_ingress_nginx_namespace` to a more permanent name.

   ```sh
   $ ansible-playbook -l <host/group> deploy.yaml -vv
   ```

2. Look up the IP or hostname for the new ingress controller:

   ```sh
   $ kubectl -n ingress-nginx-temp get svc
   ```

3. Change the DNS for all domains that point to this cluster to use the new IP or hostname.
   You may find it helpful to watch the logs of both ingress controllers during this time to
   see the traffic switch to the new ingress controller.

   The post [Kubernetes: Nginx and Zero Downtime in Production](https://medium.com/codecademy-engineering/kubernetes-nginx-and-zero-downtime-in-production-2c910c6a5ed8) has a more detailed overview
   of this approach.

4. Next, add `k8s_purge_ingress_controller: yes` to your variables file and re-run `deploy.yaml`.
   Note that you will now have both `k8s_install_ingress_controller: yes` and
   `k8s_purge_ingress_controller: yes`, however, the former refers to the new namespace and the
   latter refers only to the old namespace. This should clear out the old ingress controller.

   Note, you may need to run this a few times if Ansible times out attempting to delete everything
   the first time.

5. If you want to switch everything to use the original `ingress-nginx` namespace again, make the
   change in your variables file and re-run `deploy.yaml` with your final configuration.

   Otherwise, simply set `k8s_install_cert_manager: yes` and do not change the namespace.

   ```
   # your variables file (e.g., group_vars/all.yaml)
   k8s_install_ingress_controller: yes
   k8s_ingress_nginx_namespace: ingress-nginx

   k8s_install_cert_manager: yes
   ```

   **Make sure to remove the two `k8s_purge_*` variables as they are no longer needed and will
   be removed in a future release.**

6. If you elected to switch namespaces again:

   * Change the DNS to the new service address as in step 3 and wait for traffic to stop
     going to the temporary ingress controller.
   * Remove the `ingress-nginx-temp` namespace as follows:

     ```
     helm -n ingress-nginx-temp uninstall ingress-nginx
     kubectl delete ns ingress-nginx-temp
     ```

7. Test that cert-manager is working properly by deploying a new echotest pod as described in the README.

8. Update any projects that deploy to the cluster to use the corresponding 1.0 release of
   [ansible-role-django-k8s](https://github.com/caktus/ansible-role-django-k8s).

**Please note that the `k8s_purge_*` variables are intended only for removing the previously-installed
versions of these resources.** If you need to remove the newly installed cert-manager or ingress-nginx
for any reason, you should use the `helm uninstall` method described above.


**Other Changes:**

* Move Papertrail and New Relic to
  [caktus.k8s-hosting-services](https://github.com/caktus/ansible-role-k8s-hosting-services).
  The existing deployments will not be automatically removed, but they are no
  longer managed from this role.


### v0.0.7 on Jul 6, 2020

* Allow Papertrail memory resources to be configurable


### v0.0.6 on Jul 2, 2020

* Support creation of an AWS IAM user with limited perms that can be used on CI to push
  images and deploy.


### v0.0.5 on Jun 29, 2020

* Introduce `k8s_cluster_name` variable


### v0.0.4 on Jun 28, 2020

* On AWS, grant cluster access to IAM users in `k8s_iam_users`.


### v0.0.3 on Jun 25, 2020

* Re-enable validation in cert-manager workspace after installing or updating
  cert-manager and Lets Encrypt.
* Add NewRelic Infrastructure support


### v0.0.2 on Mar 1, 2020

* Added Logspout for Papertrail


### v0.0.1 on Feb 26, 2020

* Initial release
