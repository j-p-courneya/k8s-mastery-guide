# Bioinformatics K8s Lab Exercises
### Real-World Tasks for Your Jetstream2 Cluster

> These exercises are designed around your actual work — *P. falciparum* genomics, RNA-seq,
> spatial transcriptomics, and Nextflow pipelines. Each one teaches specific K8s concepts
> while doing something bioinformatically useful.

---

## Exercise 1: Containerized FastQC at Scale
**K8s Concepts:** Jobs, Pods, resource requests/limits, volume mounts
**Bioinformatics:** QC of FASTQ files — the first step of every pipeline you run

### The Task

Run FastQC on a set of RNA-seq FASTQ files using K8s Jobs instead of submitting
individual Slurm jobs. This is the simplest possible bioinformatics workload on K8s
and a good sanity check that your cluster works.

### Setup

```bash
# Download some public test data (small P. falciparum RNA-seq)
mkdir -p /mnt/k8s-data/fastq
cd /mnt/k8s-data/fastq
# SRA accession from a Pf RNA-seq study — small files for testing
fastq-dump --split-files --gzip SRR1028923
# Or use any small FASTQs you have
```

### K8s Manifest

```yaml
# fastqc-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fastqc-sample1
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: fastqc
        image: biocontainers/fastqc:v0.12.1_cv2
        command: ["fastqc"]
        args:
        - "/data/SRR1028923_1.fastq.gz"
        - "/data/SRR1028923_2.fastq.gz"
        - "--outdir=/output"
        - "--threads=2"
        resources:
          requests:
            cpu: "2"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        volumeMounts:
        - name: data
          mountPath: /data
          readOnly: true
        - name: output
          mountPath: /output
      volumes:
      - name: data
        hostPath:
          path: /mnt/k8s-data/fastq
      - name: output
        hostPath:
          path: /mnt/k8s-data/fastqc-results
```

### What to Observe

```bash
kubectl apply -f fastqc-job.yaml
kubectl get jobs -w                    # Watch it complete
kubectl logs job/fastqc-sample1        # See FastQC output
kubectl describe job fastqc-sample1    # Check resource usage
ls /mnt/k8s-data/fastqc-results/      # Verify HTML reports exist
```

### Level Up

Modify the Job to process multiple samples using `parallelism` and `completions`.
Think about how this compares to a Slurm job array.

---

## Exercise 2: nf-core/rnaseq on K8s with Nextflow
**K8s Concepts:** Nextflow K8s executor, ServiceAccounts, PersistentVolumeClaims, ConfigMaps, RBAC
**Bioinformatics:** Full RNA-seq pipeline — trim, align, quantify

This is the big one. Nextflow has a native Kubernetes executor that submits each
process as a K8s Pod. Instead of Slurm scheduling your STAR/Salmon/HISAT2 jobs,
K8s does it. This is directly relevant to how you'd run pipelines in production
on a K8s cluster.

### Prerequisites

```bash
# Install Nextflow on your control plane (or a dedicated "submit" node)
curl -s https://get.nextflow.io | bash
sudo mv nextflow /usr/local/bin/

# Nextflow needs kubectl access — verify
kubectl get nodes
```

### Storage Setup

Nextflow on K8s needs a shared PersistentVolumeClaim for work directories:

```yaml
# nextflow-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextflow-work
spec:
  accessModes:
  - ReadWriteMany        # Must be RWX for multi-pod access
  resources:
    requests:
      storage: 100Gi
  # storageClassName: depends on your setup
  # On Jetstream2 with Manila, use an NFS-backed StorageClass
```

```bash
kubectl apply -f nextflow-pvc.yaml
```

### RBAC: Nextflow needs permission to create pods

```yaml
# nextflow-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nextflow
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nextflow-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/status"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nextflow-binding
subjects:
- kind: ServiceAccount
  name: nextflow
roleRef:
  kind: Role
  name: nextflow-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f nextflow-rbac.yaml
```

### Nextflow Config for K8s

```groovy
// nextflow.config
profiles {
  k8s {
    process {
      executor = 'k8s'
      // Every Nextflow task becomes a K8s Pod
    }

    k8s {
      namespace = 'default'
      serviceAccount = 'nextflow'
      storageClaimName = 'nextflow-work'
      storageMountPath = '/workspace'
      // Pull policy for containers
      pullPolicy = 'IfNotPresent'
    }
  }
}

// Resource defaults — match your Jetstream2 node sizes
process {
  withLabel: 'process_low' {
    cpus = 2
    memory = '4.GB'
  }
  withLabel: 'process_medium' {
    cpus = 4
    memory = '12.GB'
  }
  withLabel: 'process_high' {
    cpus = 8
    memory = '24.GB'
  }
}
```

### Run nf-core/rnaseq

```bash
# Start with the test profile (uses tiny bundled test data)
nextflow run nf-core/rnaseq \
  -profile test,docker \
  -c nextflow.config \
  -profile k8s \
  --outdir /workspace/rnaseq-results

# Watch pods spin up in another terminal
kubectl get pods -w
```

### What to Observe

While the pipeline runs, open a second terminal and watch K8s do its thing:

```bash
# Watch Nextflow spawn pods for each process
kubectl get pods -w

# You'll see pods for: FASTQC, TRIMGALORE, STAR_ALIGN, SALMON_QUANT, etc.
# Each one starts, runs, completes, and gets cleaned up

# Check resource consumption
kubectl top pods

# If a step fails, debug it like any K8s pod
kubectl describe pod <failed-pod-name>
kubectl logs <failed-pod-name>
```

### Why This Matters

This is the exact same pipeline you run on Slurm with `-profile slurm`. By switching
to `-profile k8s`, Nextflow submits each process as a K8s Pod instead of a Slurm job.
The pipeline code doesn't change at all — only the executor. This is the power of
Nextflow's abstraction layer, and it means every nf-core pipeline you already use
works on K8s with just a config change.

---

## Exercise 3: Parallel Variant Calling with K8s Jobs
**K8s Concepts:** Job parallelism, completions, indexed jobs, ConfigMaps for parameters
**Bioinformatics:** GATK HaplotypeCaller per-chromosome — exactly what your Pf pipeline does

### The Concept

Your *P. falciparum* variant calling pipeline runs HaplotypeCaller per chromosome.
On Slurm you'd submit 14 array jobs (one per Pf3D7 chromosome). On K8s, you use
an Indexed Job where each pod gets a chromosome index.

```yaml
# gatk-haplotypecaller-parallel.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: haplotypecaller
spec:
  completionMode: Indexed
  completions: 14            # 14 P. falciparum chromosomes
  parallelism: 4             # Run 4 at a time (adjust to cluster size)
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: gatk
        image: broadinstitute/gatk:4.5.0.0
        command: ["/bin/bash", "-c"]
        args:
        - |
          # JOB_COMPLETION_INDEX is auto-set by K8s (0-13)
          CHROM_NUM=$(printf "%02d" $((JOB_COMPLETION_INDEX + 1)))
          CHROM="Pf3D7_${CHROM_NUM}_v3"
          echo "Processing chromosome: ${CHROM}"

          gatk HaplotypeCaller \
            -R /data/ref/Pf3D7.fasta \
            -I /data/bam/sample.bam \
            -L ${CHROM} \
            --heterozygosity 0.0029 \
            --indel-heterozygosity 0.0009 \
            --sample-ploidy 6 \
            --assembly-region-padding 250 \
            -O /output/sample.${CHROM}.g.vcf.gz \
            -ERC GVCF
        resources:
          requests:
            cpu: "2"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "12Gi"
        volumeMounts:
        - name: data
          mountPath: /data
        - name: output
          mountPath: /output
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: pf-reference-data
      - name: output
        persistentVolumeClaim:
          claimName: variant-output
```

### What to Observe

```bash
kubectl apply -f gatk-haplotypecaller-parallel.yaml

# Watch 4 pods run concurrently, completing 14 total
kubectl get pods -l job-name=haplotypecaller -w

# Check a specific chromosome's logs
kubectl logs haplotypecaller-3    # Index 3 = Pf3D7_04_v3

# See completion progress
kubectl get job haplotypecaller
# COMPLETIONS   DURATION
# 8/14          12m
```

This directly mirrors your existing Slurm array job workflow but teaches you
indexed Jobs, parallelism control, and how K8s schedules across nodes.

---

## Exercise 4: Persistent Bioinformatics Services
**K8s Concepts:** Deployments, Services, Ingress, persistent storage, multi-container pods
**Bioinformatics:** Always-on tools your lab actually uses

### Deploy JupyterHub for Your Team

```bash
# Add JupyterHub Helm chart
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update

# Create a minimal values file
cat > jupyterhub-values.yaml << 'EOF'
singleuser:
  image:
    name: jupyter/datascience-notebook
    tag: latest
  cpu:
    limit: 2
    guarantee: 1
  memory:
    limit: 4G
    guarantee: 2G
  storage:
    capacity: 10Gi
  defaultUrl: "/lab"

hub:
  config:
    Authenticator:
      admin_users:
        - admin
    DummyAuthenticator:
      password: changeme
    JupyterHub:
      authenticator_class: dummy

proxy:
  service:
    type: NodePort
EOF

helm install jhub jupyterhub/jupyterhub \
  --namespace jhub \
  --create-namespace \
  --values jupyterhub-values.yaml
```

### Deploy an IGV.js Genome Browser

For visualizing your spatial transcriptomics or variant calling results:

```yaml
# igv-browser.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: igv-browser
spec:
  replicas: 1
  selector:
    matchLabels:
      app: igv
  template:
    metadata:
      labels:
        app: igv
    spec:
      containers:
      - name: igv
        image: igvteam/igv-webapp:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: genome-data
          mountPath: /data
      volumes:
      - name: genome-data
        persistentVolumeClaim:
          claimName: genome-data
---
apiVersion: v1
kind: Service
metadata:
  name: igv-service
spec:
  type: NodePort
  selector:
    app: igv
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### Deploy MultiQC as a Reporting Service

```yaml
# multiqc-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: multiqc-nightly
spec:
  schedule: "0 2 * * *"       # Run at 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: multiqc
            image: ewels/multiqc:latest
            command: ["multiqc"]
            args:
            - "/data/pipeline-results"
            - "--outdir=/reports"
            - "--force"
            - "--title=Nightly QC Report"
            volumeMounts:
            - name: pipeline-data
              mountPath: /data
            - name: reports
              mountPath: /reports
          volumes:
          - name: pipeline-data
            persistentVolumeClaim:
              claimName: pipeline-results
          - name: reports
            persistentVolumeClaim:
              claimName: qc-reports
```

---

## Exercise 5: Spatial Transcriptomics Processing with DaemonSets and GPU Pods
**K8s Concepts:** GPU scheduling, node affinity, DaemonSets, resource limits for GPU
**Bioinformatics:** Processing Visium/Xenium data, cell segmentation

This one is for when you get to Tier 3 (GPU node). Run Space Ranger or
cell segmentation models on your GPU node.

### GPU-Accelerated Cell Segmentation

```yaml
# cellpose-segmentation.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cellpose-segment
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: cellpose
        image: cellprofiler/cellpose-gpu:latest
        command: ["python", "-m", "cellpose"]
        args:
        - "--dir=/data/images"
        - "--pretrained_model=cyto2"
        - "--use_gpu"
        - "--save_png"
        - "--diameter=30"
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: "8"
            memory: "24Gi"
        volumeMounts:
        - name: spatial-data
          mountPath: /data
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: spatial-data
        persistentVolumeClaim:
          claimName: spatial-datasets
      - name: dshm
        emptyDir:
          medium: Memory
          sizeLimit: "8Gi"
      # Only schedule on GPU nodes
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: "gpu-type"
        operator: "Exists"
        effect: "NoSchedule"
```

---

## Exercise 6: The Full Stack — Microservices for a Bioinformatics Portal
**K8s Concepts:** Multi-service architecture, Ingress routing, ConfigMaps, Secrets, HPA
**Bioinformatics:** Build a mini research portal

### Architecture

```
Internet → Ingress Controller
              │
              ├── /jupyter  → JupyterHub (notebooks)
              ├── /igv      → IGV.js (genome browser)
              ├── /reports  → Nginx serving MultiQC reports
              └── /api      → Flask API (submit pipeline runs)
```

This teaches you real-world K8s patterns: routing traffic to multiple services,
managing secrets for database credentials, scaling the API tier based on load,
and keeping persistent data safe while services come and go.

This is what running bioinformatics infrastructure on K8s actually looks like
in production — not a single tool, but a platform.

---

## Exercise Progression Map

```
Week 1-2 (Tier 1, single node k3s):
  └── Exercise 1: FastQC Jobs
      - Learn: pods, jobs, volumes, resource limits
      - Bioinf: basic QC

Week 3-4 (Tier 1):
  └── Exercise 2: nf-core/rnaseq on K8s
      - Learn: Nextflow K8s executor, RBAC, PVCs, watching pods
      - Bioinf: full RNA-seq pipeline — directly applicable to your work

Week 5-6 (Tier 2, multi-node kubeadm):
  └── Exercise 3: Parallel variant calling
      - Learn: indexed jobs, parallelism, cross-node scheduling
      - Bioinf: GATK HaplotypeCaller per-chromosome (your Pf pipeline)

Week 7-8 (Tier 2):
  └── Exercise 4: Persistent services
      - Learn: deployments, services, ingress, Helm, CronJobs
      - Bioinf: JupyterHub, genome browser, automated QC reports

Week 9+ (Tier 3, GPU):
  └── Exercise 5: GPU workloads
      - Learn: GPU scheduling, node affinity, tolerations
      - Bioinf: cell segmentation for your Visium/Xenium data

Ongoing:
  └── Exercise 6: Full stack portal
      - Learn: microservices architecture, autoscaling, production patterns
      - Bioinf: research infrastructure platform
```

---

## Test Data Sources

For exercises that need data but you don't want to transfer your own:

| Data | Source | Size | Use |
|------|--------|------|-----|
| Pf RNA-seq FASTQs | SRA: SRR1028923 | ~500 MB | Exercises 1, 2 |
| Pf reference genome | PlasmoDB Pf3D7 | ~23 MB | Exercise 3 |
| nf-core test data | Built into `--profile test` | Tiny | Exercise 2 |
| Visium demo data | 10x Genomics datasets page | ~1-5 GB | Exercise 5 |
| MultiQC test reports | `multiqc --example` | Tiny | Exercise 4 |

For the nf-core/rnaseq exercise, the `test` profile includes its own tiny dataset
so you don't need to provide any data at all for the first run.

---

*These exercises are ordered so each one builds on the K8s skills from the previous one.
By Exercise 4, you'll have a working bioinformatics platform on K8s that's genuinely
useful for your daily work — not just a learning exercise.*
