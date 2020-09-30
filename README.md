# caktus.k8s-web-cluster

An Ansible role to help configure Kubernetes clusters for web apps.

Supported cloud providers include GCP (GKE), AWS (EKS), Azure (AKS), and Digital
Ocean. The configuration includes installing:

* Nginx Ingress Controller
* Certificate manager (https://cert-manager.io/docs/) (https://github.com/jetstack/cert-manager)
* Let's Encrypt certificate issuer
* Logspout for Papertrail
* AWS IAM user with limited permissions for CI deploys
* For AWS, granting cluster access to IAM users

## License

This Ansible role is released under the BSD License. See the
[LICENSE](https://github.com/caktus/ansible-role-k8s-web-cluster/blob/master/LICENSE)
file for more details.

Development sponsored by [Caktus Consulting Group, LLC](http://www.caktusgroup.com/services>).


## Requirements

* ``pip install openshift kubernetes-validate``


## Installation

1. Add to your ``requirements.yaml``:


```yaml
---
# file: deploy/requirements.yaml

- src: https://github.com/caktus/ansible-role-k8s-web-cluster
  version: 0.0.7
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
k8s_cluster_name: <display name for your cluster>
k8s_letsencrypt_email: <email to contact about expiring certs>
k8s_echotest_hostname: <test hostname assigned to your cluster ip, e.g. echotest.caktus-built.com>
# aws only:
k8s_iam_users: [list of IAM usernames who should be allowed to manage the cluster]
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

### Adding a limited AWS IAM user for CI deploys

In order to be able to deploy to AWS from CI systems, you'll need to be able to
authenticate as an IAM user that has the permissions to push to the AWS ECR (Docker
registry), and possibly need to be able to read a secret from AWS Secrets Manager (the
`.vault_pass` value). This playbook can create that user for you with the proper
permissions. You can configure this with the following variables (defaults shown):

```yaml
k8s_ci_aws_profile: "default" # profile in your ~/.aws/credentials file, which will be used to create the user
k8s_ci_username: ci-user
k8s_ci_repository_arn: "" # format: arn:aws:ecr:<REGION>:<ACCOUNT_NUMBER>:repository/<REPO_NAME>
k8s_ci_vault_password_arn: "" # format: arn:aws:secretsmanager:<REGION>:<ACCOUNT_NUMBER>:secret:<NAME_OF_SECRET>
```

Only `k8s_ci_repository_arn` is required. The REPO_NAME portion can be found
[here](https://console.aws.amazon.com/ecr/repositories). The
`k8s_ci_aws_profile` value needs to be present in your `~/.aws/credentials` file and
should correspond to an IAM user with sufficient permissions to create a user in the
same AWS account where your k8s cluster lives. Finally, the `k8s_ci_vault_password_arn`
is an optional pointer to a single secret in AWS Secrets Manager. The ARN can be found
by going to this [link](https://console.aws.amazon.com/secretsmanager/home#/listSecrets)
and then clicking on the secret you're sharing with the user. On some projects, we store
the Ansible vault password in SecretsManager and then use an AWS CLI command to read the
secret so other secrets in the repo can be decrypted. This allows the CI user to access
that command.

After you set those variables and run this role, the IAM user will be created with the
proper permissions. You'll then need to use the AWS console to create an access key and
secret key for that user. Take note of the `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` values.

Copy those 2 variables (and `AWS_DEFAULT_REGION`) into the CI environment variables
console.

NOTE: If you're using this with [the web app k8s
role](https://github.com/caktus/ansible-role-django-k8s), be aware that you'll need to
make sure that `k8s_rollout_after_deploy` is disabled (which is the default), because
those commands don't currently use the service account user that this role depends on.
See https://github.com/caktus/ansible-role-django-k8s/issues/25.

### Customize Nginx Error Pages

This role will create a service (and deployment) named `nginx-errors` in the
`ingress-nginx` namespace. That service will run a pod using the image specified in
`k8s_nginx_errors_image`. This defaults to an [image built
here](https://github.com/caktus/custom-error-pages), which provides an image that
returns a customized 503 error page. The main purpose for this is to allow you to turn
on a maintenance page. In the kubernetes world, a maintenance page is rarely needed but
there are times where you might want one. For example, if you are upgrading the database
and just want to turn off web access to the DB for a short time. Since all 503 errors
will return the custom 503 maintenance page, you could do this:

```bash
kubectl scale deployment -n <my-namespace> --replicas=0 --all
```

That would turn off all your apps, and nginx would serve your maintenance page. Then,
when you restarted things (by setting `--replicas=N`), the maintenance page would go
away.

If you want to customize the 503 page, or if you want to add custom pages for any other
nginx error code, you'll need to clone the https://github.com/caktus/custom-error-pages
repo, create new HTML pages there, and then refer to your image in
`k8s_nginx_errors_image`.
