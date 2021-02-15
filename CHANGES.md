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

To upgrade, manually delete the `ingress-nginx` and `cert-manager` namespaces
and re-deploy this role.

**Other Changes:**

* Move Papertrail and New Relic to
  [caktus.k8s-hosting-services](https://github.com/caktus/ansible-role-k8s-hosting-services).
  The existing deployments will not be automatically removed, but they are no
  longer mangaged from this role.


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
