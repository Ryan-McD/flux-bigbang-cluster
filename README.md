# flux-bigbang-cluster-template
Template structure for deploying bigbang

## References

- [k8s-at-home cluster template](https://github.com/onedr0p/flux-cluster-template/tree/v1.0.0)
- [bigbang cluster template](https://repo1.dso.mil/platform-one/big-bang/customers/template/)

## Prerequisites

To deploy Big Bang, the following items are required:

- Kubernetes cluster [ready for Big Bang](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/tree/master/docs/guides/prerequisites)
- A git repo for infrastructure and configuration as code
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [GPG (Mac users need to read this important note)](https://repo1.dso.mil/platform-one/onboarding/big-bang/engineering-cohort/-/blob/master/lab_guides/01-Preflight-Access-Checks/A-software-check.md#gpg)
- [SOPS](https://github.com/mozilla/sops)
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Iron Bank Personal Access Token](https://registry1.dso.mil) - Under your `User Profile`, copy the `CLI secret`.
- [Repo1 Personal Access Token](https://repo1.dso.mil/-/profile/personal_access_tokens) - You will need `read_repository` permissions.
- [Helm](https://helm.sh/docs/intro/install/)


## Confirm nodes are online:
   
```shell
kubectl get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-master-a   Ready    control-plane,master      4d20h   v1.22.7+rke2r2
# k8s-worker-a   Ready    worker                    4d20h   v1.22.7+rke2r2
```

## BigBang Prep

Disable Swap:

  - Find Swap with `cat /proc/swaps` and disable currently with `swapoff` and permanently in `/etc/fstab`

PSPs are unsupported by bigbang (and kubernetes 1.25+), so delete them:

```sh
kubectl patch psp system-unrestricted-psp -p '{"metadata": {"annotations":{"seccomp.security.alpha.kubernetes.io/allowedProfileNames": "*"}}}'
kubectl patch psp global-unrestricted-psp -p '{"metadata": {"annotations":{"seccomp.security.alpha.kubernetes.io/allowedProfileNames": "*"}}}'
kubectl patch psp global-restricted-psp -p '{"metadata": {"annotations":{"seccomp.security.alpha.kubernetes.io/allowedProfileNames": "*"}}}'
```

### Istio Prep

* If SELinux is enabled and the OS hasn't received additional pre-configuration, then users will see istio init-container crash loop.
* Depending on security requirements it may be possible to set selinux in permissive mode: `sudo setenforce 0`.
* Additional OS and Kubernetes specific configuration are required for istio to work on systems with selinux set to `Enforcing`.

By default, BigBang will deploy istio configured to use `istio-init` (read more [here](https://istio.io/latest/docs/setup/additional-setup/cni/)).  To ensure istio can properly initialize envoy sidecars without container privileged escalation permissions, several system kernel modules must be pre-loaded before installing BigBang:

```shell
modprobe xt_REDIRECT
modprobe xt_owner
modprobe xt_statistic
cat >/etc/modules-load.d/istio-iptables.conf <<EOF
br_netfilter
nf_nat_redirect
xt_REDIRECT
xt_owner
xt_statistic
EOF
```

### ECK Specific Configuration

Increase/set max_map_count

```sh
sudo sysctl -w vm.max_map_count=262144      #(ECK crash loops without this)
```

### Sonarqube Specific Configuration (Sonarqube Is a BB Addon App)

Sonarqube requires the following kernel configurations set at the node level:

```shell
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```



This template includes metallb as an optional load balancer, and will need uncommented in the cluster/core kustomize

## Cloudflare API Token

:round_pushpin: You may skip this step, **however** make sure to `export` dummy data **on item 8** in the below list.

...Be aware you **will not** have a valid SSL cert until `cert-manager` is configured correctly

In order to use `cert-manager` with the Cloudflare DNS challenge you will need to create a API token.

1. Head over to Cloudflare and create a API token by going [here](https://dash.cloudflare.com/profile/api-tokens).
2. Click the blue `Create Token` button
3. Scroll down and create a Custom Token by choosing `Get started`
4. Give your token a name like `cert-manager`
5. Under `Permissions` give **read** access to `Zone` : `Zone` and **write** access to `Zone` : `DNS`
6. Under `Zone Resources` set it to `Include` : `All Zones`
7. Click `Continue to summary` and then `Create Token`
8. Export this token and your Cloudflare email address to an environment variable on your system to be used in the following steps

```shell
export BOOTSTRAP_CLOUDFLARE_EMAIL="email@rancherfederal.com"
export BOOTSTRAP_CLOUDFLARE_APIKEY="CloUDfLAreToken+Here"
```


## GitOps with Flux

1. Verify Flux can be installed

```shell
flux check --pre
```

2. Pre-create the `flux-system` and `bigbang` namespace

```shell
kubectl create namespace flux-system
kubectl create namespace bigbang
```

3. Create and/or add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
export GPG_TTY=$(tty)
export FLUX_KEY_NAME="Cluster name (Flux) <email>"

# Figure out what version of gpg your running
gpg --version | head -n 1
# Centos 7 and older AmazonLinux tend to run gpg 2.0.x
# Centos 8, Ubuntu 20, and Debian 11 tend run gpg 2.2.x
# Centos 9 and Mac tend to run gpg 2.3.x

# Generate a GPG master key
# Note:
# By default, gpg 2.3.x generated keys, will generate errors when
# consumed by clients running gpg 2.0.x - 2.2.x
# gpg 2.3.x users must use the following multiline keygen command for
# compatibility with sops and older clients using gpg 2.0.x - 2.2.x
# gpg 2.2.x users should also user the following key gen multiline command
# gpg 2.0.x users substitute --full-generate-key for --gen-key
#
# Using this multiline command to generate the key makes it work in all cases.
gpg --batch --full-generate-key --rfc4880 --digest-algo sha512 --cert-digest-algo sha512 <<EOF
    %no-protection
    # %no-protection: means the private key won't be password protected
    # (no password is a fluxcd requirement, it might also be true for argo & sops)
    Key-Type: RSA
    Key-Length: 4096
    Subkey-Type: RSA
    Subkey-Length: 4096
    Expire-Date: 0
    Name-Real: ${FLUX_KEY_NAME}
    Name-Comment: ${FLUX_KEY_NAME}
EOF

gpg --list-secret-keys "${FLUX_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
# uid           [ultimate] Cluster name (Flux) <email@rancherfederal.com>
# sub   rsa4096 2021-03-11 [E]

export FLUX_KEY_FP=$(gpg --list-secret-keys "${FLUX_KEY_NAME}" | grep -A1 sec | tail -1 | awk '{print $1}')
```

```sh
gpg --export-secret-keys --armor "${FLUX_KEY_FP}" |
kubectl --kubeconfig=./kubeconfig create secret generic sops \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

4. Export more environment variables for application configuration

```shell
# The repo you created from this template
export BOOTSTRAP_GIT_REPOSITORY="https://github.com/Ryan-McD/flux-bigbang-cluster-template"
# Choose one of your domains or use nip.io
export BOOTSTRAP_SECRET_DOMAIN=$(ip a | grep "ens\|eth" | grep inet | awk '{print $2}' | sed 's|/.[0-9]||g').nip.io
export BOOTSTRAP_TIMEZONE="America/New_York"
# Obtain your registry1.dso.mil credentials
export REGISTRY1_USERNAME="user_name"
export REGISTRY1_PASSWORD="somelongtoken"
export BOOTSTRAP_METALLB_LB_RANGE=""
```

5. Create required files based on ALL exported environment variables.

```sh
envsubst < ./tmpl/.sops.yaml > ./.sops.yaml
envsubst < ./tmpl/cluster/cluster-secrets.sops.yaml > ./cluster/config/cluster-secrets.sops.yaml
envsubst < ./tmpl/cluster/cluster-settings.yaml > ./cluster/base/cluster-settings.yaml
envsubst < ./tmpl/cluster/flux-cluster.yaml > ./cluster/flux/flux-system/flux-cluster.yaml
envsubst < ./tmpl/cluster/cert-manager-secret.sops.yaml > ./cluster/apps/networking/cert-manager/issuers/secret.sops.yaml
```

6. **Verify** all the above files have the correct information present

7. Determine best option for certs

Cert-manager is prepped for using letsencrypt, or you can use the certificate created by istio

8. Encrypt secrets with SOPS

```sh
export GPG_TTY=$(tty)
sops --encrypt --in-place ./cluster/config/cluster-secrets.sops.yaml
sops --encrypt --in-place ./cluster/apps/networking/cert-manager/issuers/secret.sops.yaml
sops --encrypt --in-place ./cluster/bigbang/base/secrets.sops.yaml
```

9. **Verify** all the above files are **encrypted** with SOPS

10. If you verified all the secrets are encrypted, you can delete the `tmpl` directory now

11. Push changes to git

12. Create Git credentials for Flux

   ```shell
   # Flux needs the Git credentials to access your Git repository holding your environment
   # Adding a space before this command keeps our PAT out of our history
    kubectl create secret generic private-git --from-literal=username=<Your Repo1 Username> --from-literal=password=<Your Repo1 Personal Access Token> -n bigbang
   ```

13. Deploy Flux to handle syncing

   ```shell
   # Flux is used to sync Git with the the cluster configuration

   # Choose either standard flux or IronBank "Hardened" flux (below).
   kubectl apply --kustomize ./cluster/bootstrap/

   # Wait for flux to complete
   kubectl get deploy -o name -n flux-system | xargs -n1 -t kubectl rollout status -n flux-system

    # If choosing hardened flux, ./cluster/flux/flux-system needs replaced with IB images.
    #    # If you are using a different version of Big Bang, make sure to update the `?ref=1.41.0` to the correct tag or branch.
    #    kubectl apply -k https://repo1.dso.mil/platform-one/big-bang/bigbang.git//base/flux?ref=1.41.0

   kubectl apply --kustomize ./cluster/flux/flux-system/
   ```
