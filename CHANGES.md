# caktus.k8s-web-cluster


## Changes


### NEXT

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
