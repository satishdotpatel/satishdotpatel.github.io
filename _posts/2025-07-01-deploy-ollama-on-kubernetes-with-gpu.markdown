---
title: "Deploy Ollama on Kubernetes with GPU"
layout: post
date: 2025-07-01
image: /assets/images/2025-07-01-deploy-ollama-on-kubernetes-with-gpu/ollama-gpu.png
headerImage: true
tag:
- Ollama
- Kubernetes
- GPU
- Nvidia
- AI
category: blog
blog: true
author: Satish Patel
description: "Deploy Ollama on Kubernetes with GPU"

---

Ollama is an open-source tool and runtime environment for running Large Language Models (LLMs) locally, typically on your own machine. It functions as a CLI and server runtime for managing and executing LLMs, downloading and running GGUF-format models optimized for CPU/GPU processing.

I deployed Ollama on a small Kubernetes cluster equipped with NVIDIA H200 GPUs, utilizing Kai-Scheduler to allocate fractional GPU resources.

## Prerequisites

- Functioning Kubernetes cluster
- Worker nodes with GPU installed and configured

## Step 1: Deploy Ollama

Create a deployment manifest (tiny-ollama.yaml) with Kai-Scheduler for fractional GPU allocation (0.5 GPU):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ollama
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ollama
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      name: ollama
  template:
    metadata:
      labels:
        name: ollama
        runai/queue: test
      annotations:
        gpu-fraction: "0.5"
    spec:
      schedulerName: kai-scheduler
      containers:
      - name: ollama
        image: ollama/ollama:latest
        env:
        - name: PATH
          value: /usr/local/nvidia/bin:/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        - name: LD_LIBRARY_PATH
          value: /usr/local/nvidia/lib:/usr/local/nvidia/lib64
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: compute,utility
        ports:
        - name: http
          containerPort: 11434
          protocol: TCP
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

Apply the manifest:

```
$ kubectl apply -f tiny-ollama.yaml
namespace/ollama unchanged
deployment.apps/ollama created
service/ollama unchanged
```

Verify deployment:

```
$ kubectl get deploy -n ollama
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
ollama   1/1     1            1           2m24s

$ kubectl get pod -n ollama
NAME                      READY   STATUS    RESTARTS   AGE
ollama-5547f5dfd5-5frfp   1/1     Running   0          2m1s
```

## Step 2: Install Ollama Client

Access the pod and install the Ollama client:

```
$ kubectl exec -it ollama-5547f5dfd5-5frfp -n ollama -- bash
root@ollama-5547f5dfd5-5frfp:/#
root@ollama-5547f5dfd5-5frfp:/# apt update -y
root@ollama-5547f5dfd5-5frfp:/# apt install curl -y
```

Download and extract Ollama:

```
root@ollama-5547f5dfd5-5frfp:/# curl -L https://ollama.com/download/ollama-linux-amd64.tgz -o ollama-linux-amd64.tgz
root@ollama-5547f5dfd5-5frfp:/# tar -C /usr -xzf ollama-linux-amd64.tgz
```

Execute the Llama 3.2 3B model:

```
root@ollama-5547f5dfd5-5frfp:/# ollama run llama3.2:3b-instruct-fp16
pulling manifest
pulling e2f46f5b501c: 100%
pulling 966de95ca8a6: 100%
pulling fcc5a6bec9da: 100%
pulling a70ff7e570d9: 100%
pulling 56bb8bd477a5: 100%
pulling 1bc315994ceb: 100%
verifying sha256 digest
writing manifest
success
>>> Send a message (/? for help)
```

Interact with the model at the prompt:

```
>>> Who are you?
I am an artificial intelligence model, which means I'm a computer program designed to
simulate human-like conversations and answer questions to the best of my knowledge.
I don't have personal experiences, emotions, or consciousness like humans do.
```

```
>>> what is llm?
LLM stands for Large Language Model. It refers to a type of artificial intelligence (AI)
model that uses deep learning techniques to process and understand human language.
```

Enable verbose mode to see performance metrics:

```
>>> /set verbose
>>> Tell me a knock knock joke in a single line
Knock, knock!

total duration:       247.083012ms
load duration:        88.303796ms
prompt eval count:    3182 token(s)
prompt eval duration: 32.244837ms
prompt eval rate:     98682.47 tokens/s
eval count:           6 token(s)
eval duration:        49.158717ms
eval rate:            122.05 tokens/s
```

Monitor GPU usage during queries:

```
$ gpustat -i
gpun1                Tue Jul  1 20:54:59 2025  570.133.07
[0] NVIDIA H200 NVL  | 40'C,   0 % |   604 / 143771 MB |
[1] NVIDIA H200 NVL  | 48'C,  62 % |  8701 / 143771 MB | root(8088M)
```

The deployment successfully establishes a local ChatGPT-like environment running within the Kubernetes cluster.
