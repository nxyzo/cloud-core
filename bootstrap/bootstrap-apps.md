Copy the age private key from age.key in the bootstrap/sops-age.sops.yaml and encrypt it with sops.

```bash
sops -e -i bootstrap/sops-age.sops.yaml
```

Then type in kubernetes/components/sops/cluster-secrets.sops.yaml the domain. It´s used for variables in all further files. Encrypt it afterwards
```bash
sops -e -i kubernetes/components/sops/cluster-secrets.sops.yaml
```

get an cloudflare api token for the desired dns zone. and put it in kubernetes/apps/cert-manager/cert-manager/app/secret.sops.yaml

```bash
sops -e -i kubernetes/components/sops/cluster-secrets.sops.yaml
```

generate with an secret for flux reciver and paste it in to _kubernetes/apps/flux-system/flux-instance/app/secret.sops.yaml_
```bash
 openssl rand -hex 32
```






# Bootstrap Apps

Create all needed namespaces
```bash
namespaces=(
    "cert-manager"
    "flux-system"
    "network"
    "default"
    "kube-system"
)

for namespace in "${namespaces[@]}"; do
    kubectl create namespace "$namespace"
done
```


After that all namespaces for the initial setup are created.




Now we need to apply the sops secrets befor the helmfile charts are installed.


```bash
secrets=(
    "kubernetes/components/sops/cluster-secrets.sops.yaml"
    "bootstrap/sops-age.sops.yaml"
    "kubernetes/apps/cert-manager/cert-manager/app/secret.sops.yaml"
)

for secret in "${secrets[@]}"; do
    sops exec-file "${secret}" "kubectl --namespace flux-system apply --server-side --filename {}"
done
```

Now we apply the needed crds this part is from https://github.com/onedr0p

```bash
helmfile --file bootstrap/helmfile.d/00-crds.yaml template --quiet \
    | yq eval-all 'select(.kind == "CustomResourceDefinition")' - \
    | kubectl apply --server-side --filename -
```

If thats done we can now deploy our first apps for the initial cluster.


```bash
helmfile --file "${helmfile_file}" sync --hide-notes
```