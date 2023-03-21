# ArgoCD resources for my homelab

This repository is part of my homelab_v2 setup, please check the following repository for more information: https://github.com/diademiemi/homelab_v2  

### [Project Structure](#project-structure)
The project is structured with two main directories: `manifests` and `helm`.  These directories contain the manifests and helm charts to be deployed.  

Helm values files and manifest directories are called either `local` or `cloud` to determine which environment they are for.  

<details><summary>Directory structure</summary>

```
├── helm/[1]
│   ├── namespace/
│   │   ├── project/
│   │   │   ├── chart/
│   │   │   │   ├── Chart.yaml
│   │   │   │   ├── Chart.lock
│   │   │   │   ├── values.yaml (optional)
│   │   │   │   ├── templates/
│   │   │   ├── values/
│   │   │   │   ├── local.yaml[2]
│   │   │   │   ├── cloud.yaml
├── manifests/[1]
│   ├── namespace/
│   │   ├── project/
│   │   │   ├── prod/[3]
│   │   │   │   ├── manifest01.yaml
│   │   │   │   ├── manifest02.yaml
├── projects/[4]
│   ├── prod.yaml
│   ├── stage.yaml (optional)
│   ├── test.yaml (optional)
│   ├── dev.yaml (optional)
```
[1] The `manifests` and `helm` directories are split into namespaces, projects and environments.  
[2]  For Helm, the environments are found in the `values` directory. An Application is made when a file called `cloud.yaml`/`local.yaml` (Or other environment) is found in a directory of a project.  
[3]  For manifests, an Application is made when a directory called `cloud`/`local` (Or other environment) is found in a directory of a project.  
[4]  The `projects` directory contains the ApplicationSets to deploy the applications.    

</details>

### [SOPS](#sops)
Some files in this repository are encrypted with [SOPS](https://github.com/mozilla/sops). They are using the [age](https://github.com/FiloSottile/age) encryption method.  

## [Secrets with Kustomize and SOPS](#sops-kustomize)
Secrets can be managed with [SOPS](https://github.com/mozilla/sops).  
There is a [SOPS plugin for Kustomize](https://github.com/viaduct-ai/kustomize-sops) which makes ArgoCD automatically decrypt the secrets when given a key. You can read more about this in [This Red Hat Article](https://cloud.redhat.com/blog/a-guide-to-gitops-and-secret-management-with-argocd-operator-and-sops)  

To read more about manifests, see [#manifests](#manifests).

### [Installation on workstation](#installation-on-workstation)

Make sure you have [kustomize](https://kustomize.io/), [sops](https://github.com/mozilla/sops#download) and [age](https://github.com/FiloSottile/age#installation) installed on your machine.  

[Follow the instructions here](https://github.com/viaduct-ai/kustomize-sops#installation) to install the plugin for kustomize on your machine. Set `$XDG_CONFIG_HOME` to `~/.config` if it is not set.  

### [Initialize new SOPS age public/private key](#initialize-sops)
Generate a new age keypair.  
```bash
age-keygen -o <path to age private key file>
```
This will return a public key and write the private key to the file. Make sure not to commit the private key to the repository, and keep it safe.  

To use SOPS in this repository, create a `.sops.yaml` in the root of the repository.  
```yaml
creation_rules:
  - path_regex: .*\.sops\.ya?ml
    encrypted_regex: "^(data|stringData)$"
    age: <age publickey>
```
This will encrypt the `data` or `stringData` fields in all files ending in `.sops.yaml` or `.sops.yml` with the age public key. You will need the private key to decrypt the secrets.  

### [Add a secret to a project](#add-a-secret-to-a-project)

Create a `kustomization.yaml` file in the environment directory.  
```yaml
generators:
  - ./kustomize-secret-generator.yaml
```
Create a `kustomize-secret-generator.yaml` file in the project/environment directory. Where the `files` key is a list of files to encrypt/decrypt with SOPS.
```yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  # Specify a name
  name: sops-secret-generator
files:
  - ./secret-manifest.sops.yaml
```

### [Encrypt/Decrypt secrets](#encryptdecrypt-secrets)

Encrypt the secret manifest file with SOPS. This will use the age public key in the `.sops.yaml` file.  
```bash
sops --encrypt --in-place ./secret-manifest.sops.yaml
```

Decrypt the secret manifest file with SOPS.  
```bash
export SOPS_AGE_KEY_FILE=<path to age private key file> 
sops --decrypt --in-place ./secret-manifest.sops.yaml
```

***Make sure to encrypt any secrets before committing them to a repository!***   

## [Manifests](#manifests)
Manifests are stored in the `manifests` directory.  They are processed as follows:

<details><summary>Directory structure</summary>


```
manifests/cert-manager/deploy/cloud/manifest.yaml
│         │       │      │    │
│         │       │      │    └─── manifest file
│         │       │      └──────── environment
│         │       └─────────────── project
│         └─────────────────────── namespace
└───────────────────────────────── root
```

</details>

Kustomize may also be used if a `kustomization.yaml` file is present in the environment directory. Be aware that this will cause Kustomize to manage the entire directory, so any manifests not configured in the `kustomization.yaml` will be ignored.  

## [Helm](#helm)
Helm charts are stored in the `helm` directory.  They are processed as follows:

<details><summary>Directory structure</summary>

```
helm/cert-manager/deploy/chart/Chart.yaml
│    │       │              │     │
│    │       │              │     └─── helm chart file
│    │       │              └───────── chart directory
│    │       └──────────────────────── project
│    └──────────────────────────────── namespace
└───────────────────────────────────── root
helm/cert-manager/deploy/values/cloud.yaml
│    │       │      │      │
│    │       │      │      └─── values file
│    │       │      └────────── environment
│    │       └───────────────── project
│    └───────────────────────── namespace
└────────────────────────────── root
```

</details>

To use external helm charts, add them under `dependencies` in the `Chart.yaml` file. The `Chart.yaml` must still be a valid helm chart, so you will need to pass values like `apiVersion`, `name`, `description` and `version`.  
This will include the helm chart as a subchart in the chart. Be aware that values for the subchart will need to be scoped to the subchart. For example, for cert-manager, the usual values would be:  
```yaml
installCRDs: true
```
But when it is included as a subchart called `cert-manager`, the values would be:
```yaml
cert-manager:
  installCRDs: true
```
You can find more information about this in the [Helm documentation](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/).  

## [Projects](#projects)
Projects are stored in the `projects` directory.  They are processed as follows:

<details><summary>Directory structure</summary>

```
projects/cloud.yaml
│        │
│        └─── project file of this environment
└──────────── root
```
The ApplicationSets in this file will traverse the `manifests` and `helm` directories and search for files matching `values/cloud.yaml` and manifests with a `cloud/` directory. Application objects will be created using a combination of the namespace, project and environment retrieved from these filenames.  
For more information, read through [the ApplicationSet documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) and the [cloud.yaml](projects/cloud.yaml) file.  
