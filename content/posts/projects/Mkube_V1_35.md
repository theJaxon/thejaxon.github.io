---
title: "Mkube V1.35.0 🎉🎉"
date: 2026-02-05T19:44:28+01:00
draft: false
tags:
  - kubernetes
  - vagrant
  - ansible
cover:
  image: /posts/projects/images/Mkube_V1.35.png
summary: >
  MKube automates Kubernetes cluster creation locally, this release upgrades to Kubernetes V1.35.0.

---

### What changed in this release 
- The newly built [Vagrant box](https://portal.cloud.hashicorp.com/services/vagrant/registries/the-jaxon/boxes/mkube?project_id=37eee32b-e41d-4d5a-afd1-0b3ec23484d3) uses Virtualbox instead of VMWare Fusion since [Virtualbox is now supported on ARM64 architecture](https://blogs.oracle.com/developers/oracle-virtualbox-72) - I just was not aware of this.

![MKube Vagrant Registry](/posts/projects/images/MKube_V1_35_registry.png)

- Kubernetes is now upgraded to version `1.35.0` and the [base OS is now also upgraded to **Fedora 43**](https://github.com/theJaxon/Packer/tree/main/Fedora/43)
- Deprecated ~~assign_static_ip_address~~ script since this issue is now no longer faced after switching to Virtualbox and was a workaround to ensure that the machines will get the correct assigned IP addresses from the Vagrantfile.
- `local-path-provisioner` is now upgraded to `v0.0.34` and the image tag adheres to the [image resolution rules](https://www.replicated.com/blog/breaking-change-kubernetes-1-34-tightens-image-name-resolution-rules----what-it-means-for-you).

---

### Use cases with Argocd
- You can enable gitops by forking the [`homelab` repo](https://github.com/theJaxon/homelab) and `kustomize build . | kubectl apply --server-side -f -`.
- Application definitions can be added to [applications](https://github.com/theJaxon/homelab/tree/main/applications) directory (When you apply the kustomization, this enables argocd to manage itself).

---

### TODOs and blah
- Need to work on CI (Be that directly Github Actions or with Dagger within Github Actions to make it agnostic).
- Gateway API (The used nginx ingress controller will be deprecated anyways).