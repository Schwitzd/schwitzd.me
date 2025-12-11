+++
title = 'Argocd: from Opentofu to Applicationset'
date = 2025-11-09T15:46:09Z
draft = true
+++

Since starting to work on my [Home Cluster](https://github.com/Schwitzd/IaC-HomeK3s), I have already tried three different deployment strategies for managing workloads. To provide context for the new approach, it is helpful to review how the workflow has evolved.

Initially, I used [OpenTofu](https://opentofu.org/), deploying everything with the `helm_release` resource from the [Helm provider](https://registry.terraform.io/providers/hashicorp/helm/latest). This was straightforward and fully controlled from the IaC repository.

Later, I introduced [Argo CD](https://argo-cd.readthedocs.io/en/stable/), adopting a hybrid strategy. OpenTofu continued to deploy workloads using the `helm_release` resource until Argo CD was fully operational; then it created Argo CD applications using the `argocd_application` resource. While this approach was effective, it resulted in duplication and unnecessary coupling between OpenTofu and Argo CD.

My latest approach is to replace the OpenTofu [Argo CD provider](https://registry.terraform.io/providers/argoproj-labs/argocd/latest) entirely. I now rely on the [ApplicationSet controller](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/), which allows Argo CD to automatically generate applications directly from my GitOps repository.

## GitOps repository structure

Every workload (except **Argo CD** itself) lives in its own directory under the top-level `apps` folder. Each application exposes a single descriptor file, `app.yaml`, which is the entry point consumed by Argo CD's ApplicationSet.

```tree
├── apps
│   ├── traefik
│   │   ├── app.yaml
│   │   ├── kustomization.yaml
│   │   ├── ...
│   │   └── values.yaml
│   └── vault
│       └── ...
├── secrets-store-csi-driver
│   ├── apps-autodiscovery.yaml
│   ├── projects
│   │   └── ...
│   └── values.yaml
```

## ApplicationSet Controller

Argo CD [ApplicationSet controller](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/) automates the creation and lifecycle of multiple Argo CD Applications from a single definition. Instead of manually defining each Application, the controller reads a source of truth and dynamically generates Applications based on templates.

In my setup, a single ApplicationSet manages every workload in the cluster. It uses the [Git generator in file mode](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Git/#git-generator-files), scanning the repository for `apps/*/app.yaml` descriptors. Each descriptor provides metadata such as the application name, project, namespace, and deployment mode.

```yaml
# yamllint disable rule:line-length
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-autodiscovery
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]

  generators:
    - git:
        repoURL: <GitOps repository URL>
        revision: HEAD
        files:
          - path: apps/*/app.yaml

  template:
    metadata:
      name: "{{ .name }}"
    spec:
      project: "{{ .project }}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{ .namespace }}"

  templatePatch: |
    spec:
      {{- if eq .mode "kustomize" }}
      source:
        repoURL: <GitOps repository URL>
        targetRevision: HEAD
        path: '{{ .path.path }}'
        kustomize:
          buildOptions: "--enable-helm"
      {{- end }}

      {{- if eq .mode "helm" }}
      sources:
        - repoURL: '{{ .helm.chartRepo }}'
          chart: '{{ .helm.chart }}'
          targetRevision: '{{ .helm.chartVersion }}'
          helm:
            releaseName: '{{ .name }}'
            valueFiles:
              - $values/{{ .path.path }}/{{ default "values.yaml" .helm.valuesFile }}

        - repoURL: <GitOps repository URL>
          targetRevision: HEAD
          ref: values
      {{- end }}
      {{- if .syncPolicy }}
      syncPolicy: {{ toJson .syncPolicy }}
      {{- end }}
```

The ApplicationSet uses Go templating to inject these values into a shared Application template. A templatePatch section then applies the logic needed for each deployment mode. When a directory contains a **Kustomize-based** app, the patch produces a standard Argo CD Application pointing at the folder. When it contains a **Helm-based** app, the patch builds a multi-source definition: one source for the Helm chart, and one for the local values in the GitOps repo.

## Deployment scenarios

### Kustomize mode

A workload using **Kustomize-based** keeps its manifests inside the app directory and lets Argo CD apply the Kustomization directly:

```yaml
name: traefik
project: infrastructure
namespace: infrastructure
mode: kustomize
syncPolicy:
  automated:
    prune: false
    selfHeal: true
```

With this descriptor in `apps/traefik/app.yaml`, ApplicationSet generates an Argo CD Application pointing at the folder `apps/traefik/`, using the local `kustomization.yaml` to build the manifests.

### Helm mode

A **Helm-based** workload instead defines the chart location and optional values:

```yaml
name: secrets-store-csi-driver
project: infrastructure
namespace: infrastructure
mode: helm

helm:
  chart: secrets-store-csi-driver
  chartRepo: https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
  chartVersion: 1.5.4
  valuesFile: values.yaml

syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

This instructs ApplicationSet to generate an Argo CD Application using the Helm chart found in the apps/vault directory (local path or OCI registry), applying the provided values.yaml.

## Limitations

One limitation of the ApplicationSet approach is that, as soon as Argo CD is ready, it scans the 'apps' folder and starts creating all the workloads. This will not work because some applications depend on each other, requiring a more controlled deployment approach. I still need to think of a solution for when I re-create the Home Cluster from scratch.

## Closing words

I would like to thank [u/chin_waghing](https://www.reddit.com/user/chin_waghing/) from Reddit for providing valuable input on how to improve my home cluster, allowing me to remove redundant OpenTofu code.
