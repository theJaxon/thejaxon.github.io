---
title: "Mkube V1.37.1 🎉🎉"
date: 2026-10-24T18:00:00+02:00
draft: false
tags:
  - kubernetes
  - vagrant
  - ansible
  - packer
  - dagger
cover:
  image: /posts/projects/images/Mkube_V1.37.png
summary: >
  MKube automates Kubernetes cluster creation locally, this release upgrades to Kubernetes V1.37.1, moves the box to GitHub Releases and adds a Dagger based CI.

---

### What changed in this release
- Kubernetes is now upgraded to [version `1.37.1`](https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/) with **CRI-O 1.37**, and the [base OS is now **Fedora 44**](https://github.com/theJaxon/Packer/tree/main/Fedora/44).
- The box no longer lives on HCP, [HashiCorp is shutting down HCP Vagrant](https://www.hashicorp.com/blog/hcp-vagrant-deprecation-important-dates-and-migration-guidance) (no new boxes since October 1st and everything is gone by December 31st, 2026). It's now published on [MKube's GitHub Releases](https://github.com/theJaxon/MKube/releases/tag/v1.37.1), each release holds the box together with the Vagrant catalog (`mkube.json`) and the `Vagrantfile` reads it from the latest release, no registry needed.
- The box version now follows the full Kubernetes version (`1.37.1` instead of `1.37`), so the release tag tells you exactly what's inside.
- Provisioning is faster since the box now ships with:
  - **Ansible baked in**, previously Vagrant installed it on every single `vagrant up` (on both machines).
  - **Pre-pulled images** for everything MKube deploys (kubeadm, local-path), pinned by tag in [`images.txt`](https://github.com/theJaxon/Packer/blob/main/Fedora/44/images.txt). The Packer build fails if kubeadm isn't exactly `v1.37.1` or if an image kubeadm needs is missing, so results are predictable.
- The box size was reduced from **2.78 GB down to 1.64 GB** by skipping weak dependencies and `linux-firmware` (a VM doesn't need it), cleaning caches and trimming free disk space before export, which also keeps it under GitHub's 2 GiB asset limit.
- Fixed `ImageInspectError: short name mode is enforcing`, [CRI-O 1.34+ now enforces short image names](https://www.replicated.com/blog/breaking-change-kubernetes-1-34-tightens-image-name-resolution-rules----what-it-means-for-you) and Fedora lists more than one search registry, so short names are now resolved against `docker.io` only ([#8](https://github.com/theJaxon/MKube/pull/8)) and the fix is also baked in the box.

---

### CI with Dagger
- The pipeline now lives in a [Dagger module](https://github.com/theJaxon/MKube/tree/main/dagger) (Packer `fmt`/`validate`, Ansible syntax check, Vagrantfile check, publishing to GitHub Releases), GitHub Actions only runs `dagger call lint` so the same steps run on my laptop and in CI ([#9](https://github.com/theJaxon/MKube/pull/9), [#10](https://github.com/theJaxon/MKube/pull/10)).
- The Packer build itself stays local, it needs Virtualbox on Apple silicon and [GitHub's arm64 macOS runners don't support nested virtualization](https://github.com/actions/runner-images/issues/9254), so a `Makefile` glues it together: `make release` runs lint → build → publish → bump → smoke test.
- Rebuilding the same Kubernetes version replaces that release's box instead of pushing a new version.

---

### Upgrading
- If you're on `1.35`, pull the latest changes then `vagrant destroy -f && vagrant up`, the new box is downloaded from GitHub Releases automatically.

---

### TODOs and blah
- ~~Need to work on CI~~ ✅ done with Dagger (Only linting though).
- ✅ Make MKube fully use the pre-pulled images (pin the Cilium chart, pass `--kubernetes-version` to kubeadm) to cut provisioning time even more.
- Gateway API, [ingress-nginx is now retired](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/) so it's being replaced with **Traefik** (Ingress + Gateway API) and **MetalLB** for LoadBalancer IPs, next release.
