
# Setting up a Kubernetes Cluster with Minikube for vLLM on a GPU Machine

This guide provides a detailed, step-by-step process for setting up a local Kubernetes cluster using `minikube` on a GPU-enabled machine (like a Lambda Labs instance) to deploy and serve a vLLM model.

## Prerequisites

Before you begin, ensure that your machine has the following:

*   **NVIDIA Drivers:** The appropriate NVIDIA drivers for your GPU must be installed.
*   **Docker:** `minikube` will use Docker as the driver for creating the Kubernetes cluster.
*   **Homebrew (optional but recommended):** A package manager for installing `minikube` and other tools.

## Step 1: Verify NVIDIA Toolkit Installation

First, we need to ensure that the NVIDIA drivers and the NVIDIA Container Toolkit are installed and working correctly.

**a. Verify NVIDIA Drivers:**

Run `nvidia-smi` to check the status of your GPU and drivers.

```bash
nvidia-smi
```

You should see a table with your GPU information.

**b. Verify NVIDIA Container Toolkit:**

The NVIDIA Container Toolkit allows Docker containers to access the GPU. You can verify its installation by running a test Docker container:

```bash
sudo docker run --rm --gpus all nvidia/cuda:11.6.2-base-ubuntu20.04 nvidia-smi
```

If this command works, it will print the `nvidia-smi` output. This is a crucial verification step. If it fails, you need to install or fix the NVIDIA Container Toolkit installation before proceeding.

## Step 2: Install `minikube` and `kubectl`

Next, we'll install `minikube` and the Kubernetes command-line tool, `kubectl`.

**a. Install `minikube` and `kubectl`:**

```bash
brew install minikube
brew install kubectl
```

**b. Install `conntrack`:**

`minikube` requires the `conntrack` tool.

```bash
sudo apt-get update && sudo apt-get install -y conntrack
```

## Step 3: Start the `minikube` Cluster

Now, we'll start the Kubernetes cluster with GPU support.

```bash
minikube start --driver=docker --gpus=all
```

This command will download the necessary images and create a single-node Kubernetes cluster. To check the status of your cluster, run:

```bash
minikube status
```

## Step 4: Enable GPU Support in `minikube`

`minikube` provides an easy way to enable the NVIDIA device plugin, which makes the GPU available to Kubernetes.

**a. Enable the addon:**

```bash
minikube addons enable nvidia-device-plugin
```

**b. Verify GPU visibility:**

Now, check if Kubernetes can see the GPU. The `kubectl` binary that comes with `minikube` is not automatically added to your `PATH`. You can either use `minikube kubectl --` or create an alias.

To create a temporary alias for your current session:
```bash
alias kubectl="minikube kubectl --"
```

Now, run:
```bash
kubectl describe node minikube
```

In the output, under the `Capacity` and `Allocatable` sections, you should see a line like `nvidia.com/gpu: 1`.

## Step 5: Deploy the vLLM Application

Now, we'll deploy the vLLM application.

**a. Update Hugging Face Token:**

Make sure you have updated your Hugging Face token in the `inference/hf_token.yaml` file.

**b. Update PVC StorageClass:**

`minikube`'s default `StorageClass` is `standard`. You need to update your `inference/pvc.yaml` to use it.

```bash
sed -i '' 's/local-path/standard/g' inference/pvc.yaml
```
*(Note: This command is for macOS. For other Linux distributions, you might need to use `sed -i 's/local-path/standard/g' inference/pvc.yaml`)*

**c. Update Health Probes (Important for stability):**

The `vllm` server can take a long time to start, especially when downloading a large model. It's recommended to increase the `initialDelaySeconds` for the health probes in your `inference/deployment.yaml` to avoid restart loops.

```yaml
# in inference/deployment.yaml
livenessProbe:
  initialDelaySeconds: 600 # 10 minutes
readinessProbe:
  initialDelaySeconds: 600 # 10 minutes
```

For initial debugging, you can also comment out or remove the `livenessProbe` and `readinessProbe` sections entirely.

**d. Deploy the application:**

```bash
kubectl apply -f inference/
```

## Step 6: Verify the Deployment

Here are some commands to check the status of your deployment.

*   **Watch the pod status:**
    ```bash
    kubectl get pods -w
    ```
    Wait for the pod's status to become `Running`.

*   **Check the pod's logs:**
    ```bash
    kubectl logs <your-pod-name>
    ```

*   **Check the logs of a crashed container:**
    ```bash
    kubectl logs <your-pod-name> --previous
    ```

*   **Check the PVC status:**
    ```bash
    kubectl get pvc
    ```
    The status should be `Bound`.

## Step 7: Access the vLLM Service

To access your vLLM service, you can use `kubectl port-forward`.

```bash
kubectl port-forward svc/mistral-7b 8000:80
```

This will forward port `8000` on your local machine to the service. You can then send requests to `localhost:8000`.

## Step 8: (Optional) Deploy and Access Grafana

**a. Deploy the monitoring stack:**

```bash
kubectl apply -f observability/
```

**b. Access the Grafana dashboard:**

The most secure way to access the Grafana dashboard running on your remote machine is to use SSH local port forwarding.

Open a **new terminal on your local computer** and run:

```bash
ssh -L 3000:localhost:3000 user@your-lambda-labs-ip
```

Then, in your local browser, go to `http://localhost:3000`. The default credentials are `admin`/`admin`.
