# ⎈ Kubernetes Mastery Guide

**From HPC/Slurm Admin to K8s Super User — with a bioinformatics focus.**

A comprehensive, self-paced Kubernetes learning guide designed for researchers and bioinformaticians who already know their way around HPC clusters, Slurm, containers, and the command line. Instead of starting from zero, this guide maps familiar concepts (partitions → namespaces, `sbatch` → Jobs, `sinfo` → `kubectl get nodes`) and builds toward advanced topics like GPU scheduling, Nextflow on K8s, and multi-omics pipeline orchestration.

## What's in this repo

| File | Description |
|------|-------------|
| **[index.html](index.html)** | Interactive browser-based guide with sidebar navigation, dark/light mode, search, code copy buttons, and section completion tracking. Open it locally or host it anywhere. |
| **[k8s-mastery-guide.md](k8s-mastery-guide.md)** | The raw markdown source covering all 13 phases — from installation through CRDs, Operators, and GitOps. |
| **[jetstream2-k8s-lab-plan.md](jetstream2-k8s-lab-plan.md)** | A budget-conscious plan for setting up a K8s learning lab on Jetstream2 using ACCESS credits, with SU cost breakdowns for three cluster tiers. |
| **[bioinfo-k8s-exercises.md](bioinfo-k8s-exercises.md)** | Six hands-on exercises that teach K8s through real bioinformatics tasks — FastQC jobs, nf-core/rnaseq on the K8s executor, parallel GATK HaplotypeCaller, JupyterHub deployment, GPU cell segmentation, and more. |

## Guide overview

The main guide covers these phases, roughly one per week:

- **Phase 0** — Prerequisites, system requirements, and five installation paths (minikube, kind, k3s, kubeadm, managed cloud)
- **Phases 1–3** — Core architecture, kubectl fluency, workload types (Deployments, StatefulSets, DaemonSets, Jobs, CronJobs)
- **Phases 4–6** — Networking and Services, persistent storage, ConfigMaps and Secrets
- **Phases 7–9** — RBAC and security, Helm, autoscaling (HPA/VPA/cluster)
- **Phase 10** — GPU and HPC workloads (NVIDIA device plugin, node affinity, taints/tolerations, distributed training)
- **Phase 11** — ML pipelines with Kubeflow
- **Phase 12** — Observability with Prometheus, Grafana, and Loki
- **Phase 13** — CRDs, Operators, and GitOps with ArgoCD

Each section includes a Slurm-to-K8s concept mapping, copy-pasteable YAML manifests, and kubectl commands you can run immediately.

## Bioinformatics exercises

The exercises are designed around real multi-omics workflows:

1. **FastQC Jobs** — Run QC on *P. falciparum* RNA-seq FASTQs as K8s batch Jobs
2. **nf-core/rnaseq on K8s** — Run a full RNA-seq pipeline using Nextflow's native K8s executor
3. **Parallel variant calling** — GATK HaplotypeCaller per-chromosome using indexed Jobs
4. **Persistent services** — Deploy JupyterHub, IGV genome browser, and automated MultiQC reporting
5. **GPU workloads** — Cell segmentation for Visium/Xenium spatial transcriptomics on A100 GPUs
6. **Full stack portal** — Multi-service bioinformatics platform with Ingress routing

## Jetstream2 lab plan

Three progressively complex cluster tiers on Jetstream2:

- **Tier 1** — Single m3.quad with k3s (~240 SUs/month at 15 hrs/week)
- **Tier 2** — 3-node kubeadm cluster (~576 SUs/month at 12 hrs/week)
- **Tier 3** — Multi-node + g3.medium GPU worker (~864 SUs/month)

Includes setup commands, Calico CNI fixes specific to Jetstream2, storage strategies with Manila shares, and a shelving discipline guide to avoid burning credits overnight.

## Using the interactive guide

The `index.html` file is fully self-contained. You can:

- **Open it locally** — just double-click or `open index.html`
- **GitHub Pages (automatic)** — every push to `main` is automatically deployed via the included GitHub Actions workflow. Enable GitHub Pages in your repository settings (Settings → Pages → Source: GitHub Actions) and the site will be live at `https://<your-username>.github.io/k8s-mastery-guide/`.
- **Serve it anywhere** — drop it on any web server, S3 bucket, or `python3 -m http.server`

Features: sidebar navigation with search, dark/light theme toggle, reading progress bar, code copy buttons, per-section completion tracking (saved in localStorage), and print/PDF-friendly styles.

## Who this is for

- Bioinformaticians and computational biologists moving from HPC to cloud-native
- Slurm administrators learning Kubernetes
- Researchers who want to run Nextflow/nf-core pipelines on K8s
- Anyone with ACCESS credits on Jetstream2 who wants a K8s lab

## License

Personal reference material. Use and adapt freely.
