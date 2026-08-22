# 1. Executive Summary & Core ML/AI Challenge

### What Michelangelo was solving

Uber’s core problem was **not simply “how do we train better models?”** It was how to make ML development **reliable, reproducible, scalable, collaborative, and production-ready across a huge organization**.

Before Michelangelo, applied scientists used Jupyter Notebooks while engineers built bespoke production pipelines. There was no standardized way to create reliable/reproducible training and prediction workflows, compare experiments, or deploy models without creating custom serving containers.

As Uber's ML adoption exploded, additional problems emerged:

1. **Fragmented ML tooling**
   - Different teams built their own ML platforms.
   - Different model types used different development environments.
   - Developers constantly switched between disconnected tools.
2. **Poor definition of ML quality**
   - Teams often measured only offline metrics such as AUC/RMSE.
   - Important dimensions such as online performance, training-data freshness, reproducibility, and dataset coverage were missed.
3. **Deep-learning infrastructure gap**
   - DL required GPU management, sophisticated feature transformation, distributed training, custom losses, embeddings, and specialized serving.
   - Teams were sometimes spending months building their own DL tooling.
4. **Collaboration/version-control problem**
   - Model configurations and notebook changes were difficult to collaborate on.
   - There was no centralized ML source of truth or proper code review process.
5. **Unequal business criticality**
   - An experimental ML project and an ETA/fraud/safety model shouldn't receive identical infrastructure, support, and outage priorities.
   - Michelangelo therefore introduced explicit ML project tiering.

### Scale

This is where the architecture becomes especially interesting for ML System Design interviews:

| Metric | Scale |
|---|---:|
| Active ML projects          | \~400            |
| Monthly model training jobs | >20K             |
| Production models           | >5K              |
| Peak real-time predictions  | **10M/sec**      |
| Features in Palette         | >20K             |
| GPUs                        | >5,000           |
| Uber cities                 | >10,000          |
| Countries                   | >70              |
| Daily trips                 | 25M              |
| Monthly active users        | 137M             |
| DeepETA training data       | >1B trips        |
| DeepETA model size          | >100M parameters |
| Tier-1 models using DL      | >60%             |

The platform therefore had to operate simultaneously as a **developer platform, distributed training platform, feature platform, model registry, deployment platform, serving platform, and observability platform**.


# 2. ML/AI System & Platform Architecture

## 2.1 High-level architecture

The most important architectural idea is that Michelangelo 2.0 separates the system into **three planes**:

```text
                    ┌──────────────────────────────┐
                    │        CONTROL PLANE         │
                    │ APIs / Lifecycle Management  │
                    │ Project / Pipeline / Model   │
                    │ Deployment / InferenceServer │
                    └──────────────┬───────────────┘
                                   │
                ┌──────────────────┴──────────────────┐
                │                                     │
       ┌────────▼─────────┐                  ┌────────▼─────────┐
       │ OFFLINE DATA     │                  │ ONLINE DATA      │
       │ PLANE             │                  │ PLANE            │
       │                   │                  │                  │
       │ Feature Compute   │                  │ Feature Serving  │
       │ Training          │                  │ Real-time ML     │
       │ Evaluation        │                  │ RPC Services     │
       │ Batch Inference   │                  │ Streaming        │
       └────────┬─────────┘                  └────────┬─────────┘
                │                                     │
        Spark / Ray pipelines                  Online prediction
                │                                     │
                └────────────── Model ─────────────────┘
```

The control plane manages ML entities and their lifecycle; the offline plane performs large-scale computation; and the online plane handles real-time feature access and model inference.


## 2.2 Control plane

Michelangelo adopts the **Kubernetes Operator pattern**.

Its APIs follow Kubernetes API conventions for entities such as:

- Project
- Pipeline
- PipelineRun
- Model
- Revision
- InferenceServer
- Deployment

The underlying Kubernetes API machinery—including API server, etcd and controller manager—is leveraged to provide consistent lifecycle management.

### Why this matters

The API is **declarative**.

That enables the same ML configuration to be manipulated through:

```text
UI
 ↓
API
 ↓
Git repository / code
```

rather than having UI state and code state become separate sources of truth.

That is a very strong ML-platform design pattern.


# 2.3 Feature engineering / Feature Store

Michelangelo introduced **Palette**, its feature store.

Palette supports:

- Batch feature computation
- Near-real-time feature computation
- Feature sharing across teams
- Reusable features

It currently hosts **20,000+ features** that teams can leverage directly when building ML models.

An important earlier design choice was bundling feature transformations with the model using Spark PipelineModel. This helped prevent **training-serving skew** because the same transformation logic was used in training and serving.

For DL, however, this became insufficient because Spark transformations couldn't run on GPU for low-latency serving.

Michelangelo 2.0 therefore introduced DL-native transformations using:

- Keras operators
- PyTorch operators
- Custom Python transformations

The transformation graph could then be combined with the inference graph in TensorFlow or TorchScript for GPU serving.

### Interview insight

This gives you an excellent answer whenever the interviewer asks:

> **"How do you prevent training-serving skew?"**

You can point to the architectural principle:

**Keep feature transformation logic coupled with the model/inference graph and make the same transformation definition usable during both training and serving.**


# 2.4 Offline ML pipeline

Michelangelo's offline data plane represents ML workflows as **DAGs of steps**.

Typical stages include:

```text
Feature computation
       ↓
Training
       ↓
Scoring
       ↓
Evaluation
       ↓
Batch inference
```

The pipelines support **intermediate checkpoints and resume**, meaning a failed pipeline doesn't necessarily need to recompute every previous stage.

Execution frameworks include:

- **Spark**
- **Ray**

Michelangelo 2.0 moved major training workloads from Spark-based trainers to Ray-based trainers because DL introduced GPU, mini-batch shuffle, all-reduce, and operational challenges that Spark was not handling as effectively.


# 2.5 Training architecture

### Traditional ML

Historically:

```text
Spark
 ├── Feature processing
 ├── Training
 └── Batch inference
```

### Deep Learning

Michelangelo 2.0 supports:

- TensorFlow
- PyTorch
- Horovod
- Ray
- RayTune
- Elastic Horovod

The major evolution was:

```text
                 Ray
                  │
        ┌─────────┴─────────┐
        │                   │
 Distributed Training    RayTune
        │              Hyperparameter
        │                 tuning
     Horovod
        │
   Multi-GPU / Multi-node
```

### Elastic training

Elastic Horovod allows the number of workers to dynamically change during training.

Therefore:

```text
Worker 1 ─┐
Worker 2 ─┤
Worker 3 ─┤── Training
Worker 4 ─┘

Machine disappears

Worker 1 ─┐
Worker 2 ─┤── Training continues
Worker 3 ─┘
```

The job can continue with minimal interruption even as machines join or leave.

This is an important **fault-tolerance + elastic resource utilization** pattern.


# 2.6 Incremental training

Michelangelo also supports **incremental training**.

Instead of:

```text
New data
   ↓
Train from scratch
```

it can do:

```text
Existing model
      +
Additional dataset
      ↓
Incremental training
      ↓
Updated model
```

The stated benefits are:

- Lower resource consumption
- More efficient production retraining
- Better dataset coverage
- Potentially better model accuracy.


# 2.7 Model serving

Tier-1 Uber models are highly latency-sensitive.

Examples include:

- Maps ETA
- Eats homefeed ranking

Michelangelo therefore needed a serving abstraction that:

1. Supports multiple ML frameworks.
2. Hides framework-specific serving complexity from users.
3. Provides GPU-optimized low-latency inference.

Historically, Michelangelo used **Neuropod**.

Michelangelo 2.0 moved toward **NVIDIA Triton** through the Online Prediction Service (OPS). Triton supports:

- TensorFlow
- PyTorch
- Python
- XGBoost

and is optimized for GPU-based low-latency inference.

The architecture is therefore roughly:

```text
Client / Uber Microservice
          │
          ▼
Online Prediction Service
          │
          ▼
       Triton
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
TensorFlow PyTorch XGBoost
```


# 2.8 GPU resource management

Uber manages **5,000+ GPUs** across:

- On-premise datacenters
- OCI
- GCP
- Multiple regions
- Multiple zones
- Multiple clusters

The compute environment was transitioning from Peloton/Mesos toward Kubernetes.

### Resource sharing

Instead of statically allocating GPUs to teams:

```text
Team A → fixed GPUs
Team B → fixed GPUs
Team C → fixed GPUs
```

Michelangelo supports **elastic CPU/GPU resource sharing**, allowing teams to opportunistically use idle resources belonging to other teams.

### Job federation

Multiple Kubernetes clusters are abstracted behind a job federation layer:

```text
                 Michelangelo
                      │
                Job Federation
             ┌────────┼────────┐
             ▼        ▼        ▼
           K8s-1    K8s-2    K8s-3
          Region A  Region B Region C
```

The developer doesn't need to care about region, zone, or cluster details.

The federation layer itself uses the Kubernetes Operator design and is implemented as a **job CRD controller**.

It supports both Spark and Ray jobs.


# 2.9 Model quality & observability

One of the strongest ideas in this architecture is **Model Excellence Score (MES)**.

Instead of considering:

> "Model has AUC = 0.92, therefore model is good."

Michelangelo evaluates multiple quality dimensions throughout the lifecycle.

Examples include:

- Training model accuracy
- Prediction accuracy
- Model freshness
- Prediction feature quality

MES uses an **SLA-style approach**, borrowing the concept from SRE/DevOps.

The platform also provides:

- Model performance monitoring
- Feature monitoring
- Online/offline feature consistency checks
- MES
- Production runtime validation
- Debugging support.


# 2.10 ML project tiering

Michelangelo created four tiers.

```text
Tier 1 → Critical business ML
Tier 2
Tier 3
Tier 4 → Experimental / exploratory
```

Tier 1 includes critical functionality such as:

- ETA
- Safety
- Fraud detection

Tier 4 generally represents experimental use cases with limited business impact.

Tiering determines:

- Resource allocation
- Outage handling
- Investment
- Best-practice enforcement
- Compliance
- Support priority.

This is **very important for MLSD interviews** because not every model deserves the same infrastructure.


# 2.11 ML development / CI/CD architecture

Project Canvas transformed ML development into something closer to traditional software engineering.

Key components:

### ML Application Framework

Predefined but customizable workflow templates for complex ML techniques such as DL.

### ML Monorepo

Central source of truth for:

- ML code
- Configuration
- Model development

### Dependency management

Uses:

- Bazel
- Docker

Each ML project has customized Docker images.

Training and serving code are packaged into **immutable Docker images**.

### CI/CD

Changes merged into the master branch trigger:

```text
Code change
   ↓
CI tests/validation
   ↓
Deployment
   ↓
Production
```

### Artifact management

Tracks:

- Models
- Datasets
- Evaluation reports
- Metadata
- Lineage

Artifacts are stored in distributed storage and metadata is indexed/searchable.


# 2.12 Generative AI / LLM architecture

The later evolution extends Michelangelo into **LLMOps**.

Uber needs both:

```text
External LLM APIs
       +
In-house open-source LLMs
```

External models are useful for general knowledge and complex reasoning.

Internally hosted/fine-tuned models can leverage Uber's proprietary data and provide lower cost and latency for Uber-specific tasks.

To unify access, Uber built the **Gen AI Gateway**.

It provides:

- Unified LLM access
- Logging/auditing
- Cost guardrails
- Usage attribution
- Safety/policy guardrails
- PII redaction.

Michelangelo then extends across:

```text
Data preparation
      ↓
Prompt engineering
      ↓
Fine-tuning
      ↓
Evaluation
      ↓
Deployment
      ↓
Serving
      ↓
Production monitoring
```

Key components include:

- Model Catalog
- LLM Evaluation Framework
- Prompt Engineering Toolkit
- Hugging Face integration
- PEFT
- Ray-based training
- DeepSpeed model parallelism
- Elastic GPU management.

**Important:** the document does **not** describe a RAG architecture, vector database, retrieval pipeline, or agent orchestration system. Those should not be attributed to this blog.


# 3. Critical ML Engineering Trade-offs & Design Choices

## 3.1 Centralized platform vs. team-owned infrastructure

### Problem

Every team building its own ML infrastructure leads to:

- Duplicate engineering
- Fragmented UX
- Different standards
- Difficult maintenance
- Inconsistent production quality

### Decision

Create a centralized Michelangelo platform while keeping data scientists/ML engineers embedded in product teams.

The blog explicitly identifies this organizational model as an important lesson.

### Interview principle

> **Centralize infrastructure; decentralize domain expertise.**


## 3.2 Standardization vs. flexibility

Michelangelo deliberately does **not** force every ML use case into one rigid workflow.

Instead:

```text
                    Michelangelo
                         │
          ┌──────────────┴──────────────┐
          │                             │
     Standard path                Advanced path
          │                             │
       MA Studio                     Canvas
          │                             │
      Predefined                 Custom workflows
      workflows                  Custom model code
```

Most use cases get high-level abstractions.

Power users can access lower-level infrastructure and create custom pipelines/templates.

That's a classic:

**80% standardized + 20% extensible**

platform strategy.


## 3.3 UI vs. code

Michelangelo chose **API-first**, while retaining UI for visualization and rapid iteration.

Model changes made through the UI can still become version-controlled/code-reviewed changes.

This addresses a major ML engineering problem:

> UI is easy for experimentation, but code is better for reproducibility and collaboration.


## 3.4 Monolithic vs. plug-and-play

Michelangelo intentionally moved toward **plug-and-play components**.

Some components can be:

- Built internally
- Open-source
- Third-party

But the managed platform only exposes a controlled subset to provide the best user experience.

Advanced users can bring their own components.

This is a strong platform design principle:

> **Expose simplicity by default, extensibility when necessary.**


## 3.5 Build vs. buy

The blog explicitly favors evaluating:

- Open source
- Cloud offerings
- In-house systems

It notes that OSS may be preferred over proprietary offerings, while cloud capacity costs must be considered carefully.

The transition from:

```text
Neuropod → Triton
Spark-based trainers → Ray
In-house HPO → RayTune
```

demonstrates this philosophy in practice.


## 3.6 Spark vs. Ray

Spark worked well historically but DL exposed limitations around:

- GPU executors
- Mini-batch shuffle
- All-reduce
- Operational complexity

Michelangelo therefore moved training toward Ray for scalability and reliability.

This is a very useful interview trade-off:

> Don't select a distributed framework purely because it is already present. Workload characteristics can justify introducing a specialized execution engine.


## 3.7 DL vs. simpler models

The blog makes a particularly valuable point:

**Deep learning isn't automatically better.**

Uber explicitly says that in several cases **XGBoost outperforms DL in both performance and cost**.

Therefore:

```text
More sophisticated model
        ≠
Better production system
```

Model choice must consider:

- Business requirements
- Performance
- Cost
- Infrastructure complexity
- Data characteristics


## 3.8 GPU utilization vs. isolation

With 5,000+ GPUs, statically reserving resources would potentially leave capacity idle.

Uber therefore uses elastic resource sharing.

```text
Team A idle GPUs ─────┐
Team B idle GPUs ─────┼──→ Available shared capacity
Team C workload ──────┘
```

This improves utilization while the federation layer abstracts infrastructure topology.


## 3.9 Reliability through checkpoints + elasticity + rollback

Reliability isn't implemented through one mechanism.

It is layered:

### Pipeline level

Intermediate checkpoints allow workflows to resume.

### Training level

Elastic Horovod allows worker changes with minimal interruption.

### Deployment level

MA Studio provides:

- Incremental zonal rollout
- Automatic rollback triggers
- Runtime validation.

### Quality level

MES monitors model quality using SLA-style measurements.

This is an excellent example of **defense-in-depth for ML reliability**.


## 3.10 LLM cost vs. capability

Uber uses both external and internally hosted LLMs.

The trade-off described by the blog is essentially:

```text
External LLM
+ Strong general knowledge/reasoning
- External API dependency

In-house fine-tuned LLM
+ Uber-specific performance
+ Lower cost
+ Lower latency
- Requires internal infrastructure
```

The Gen AI Gateway provides the abstraction and guardrails across both choices.


# 4. High-Impact Interview Takeaways

## 4.1 The core design pattern

If an interviewer asks:

> **"Design an ML platform for a large company."**

A strong high-level answer based directly on this architecture is:

```text
                    ML PLATFORM
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Control Plane   Offline Plane   Online Plane
          │              │              │
       APIs          Training        Inference
       Metadata      Evaluation      Features
       Lifecycle     Batch jobs      Real-time
          │              │              │
          └──────────────┼──────────────┘
                         ▼
               ML Quality / Observability
```

Then add:

```text
Feature Store
Model Registry
Artifact/Lineage
CI/CD
GPU Resource Management
Project Tiering
Safe Deployment
Monitoring
```

That gives you a very strong **end-to-end ML platform decomposition** grounded in the blog.


## 4.2 Talking Point #1 — Platform architecture

> **"At scale, I would separate the ML platform into control, offline-data, and online-data planes. The control plane manages lifecycle and declarative APIs; the offline plane handles computationally heavy feature processing, training and evaluation; and the online plane handles low-latency feature access and model inference. This separation allows each plane to scale according to its workload."**

This is probably the **single most important architectural talking point** from the article.


## 4.3 Talking Point #2 — Standardization vs. flexibility

> **"I wouldn't force every ML workload into a single rigid pipeline. I'd provide standardized workflows for the majority of use cases while allowing advanced teams to plug in custom training code, loss functions, and infrastructure components. That gives the platform high developer productivity without blocking sophisticated ML workloads."**

This is especially useful when the interviewer asks:

> "How would you design an ML platform used by hundreds of teams?"


## 4.4 Talking Point #3 — Model quality

> **"For ML systems, offline accuracy alone isn't enough. I'd define quality across the entire lifecycle—model accuracy, prediction quality, model freshness, and feature quality—and manage those dimensions using SLA-like standards. I'd also tier models based on business criticality so that infrastructure investment and incident response match business impact."**

This is an **excellent differentiator** in MLSD interviews because it demonstrates that you're thinking beyond model training.


# The 5 Concepts I Would Memorize for Interviews

If your goal is **FAANG/Tier-1 ML System Design**, I would extract these five mental models from Michelangelo:

### 1. **Three-plane architecture**

**Control → Offline → Online**

### 2. **Feature consistency**

**Same transformation logic → training + serving → reduce training-serving skew**

### 3. **ML quality ≠ offline accuracy**

**Accuracy + freshness + feature quality + online performance + reproducibility**

### 4. **Standardized platform + extensibility**

**MA Studio for common workflows + Canvas for advanced/custom workflows**

### 5. **Reliability is layered**

**Checkpointing + elastic training + resource federation + safe rollout + rollback + monitoring**

And one particularly strong final principle from the blog:

> **Use DL when its advantages justify the additional infrastructure complexity; don't assume the most sophisticated model is automatically the best production choice.** Uber explicitly notes that XGBoost can outperform DL in both performance and cost in some cases.

That is the kind of trade-off an interviewer will usually value more than simply listing technologies.
