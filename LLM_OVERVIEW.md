# OpenWebUI + LiteLLM + Gemini on Kubernetes

Okay, let's get OpenWebUI talking to Gemini via LiteLLM on Kubernetes.

Here's a breakdown of the components and the steps involved:

* **OpenWebUI**: The frontend application container. It needs to know the API endpoint of the LLM backend.
* **LiteLLM**: The proxy/middleware container. It will expose an OpenAI-compatible API endpoint that OpenWebUI can talk to. It needs your Gemini API key and will forward requests to Google's Gemini API.
* **Kubernetes Secret**: To securely store your Gemini API key.
* **Kubernetes Deployments**: To manage the running instances (Pods) of OpenWebUI and LiteLLM.
* **Kubernetes Services**: To provide stable network addresses for OpenWebUI (external access) and LiteLLM (internal access for OpenWebUI).

## Prerequisites

* A running Kubernetes cluster.
* `kubectl` configured to interact with your cluster.
* Your Google Gemini API Key. Get one from Google AI Studio.
* A namespace in Kubernetes where you want to deploy these components (optional but recommended, e.g., `llm-stack`). If you don't have one, create it:

    ```bash
    kubectl create namespace llm-stack
    ```

## Step 1: Create the Kubernetes Secret for Gemini API Key

Replace `<YOUR_GEMINI_API_KEY>` with your actual key. Replace `llm-stack` if you use a different namespace.

```bash
kubectl create secret generic gemini-api-key-secret \
  --from-literal=api-key='<YOUR_GEMINI_API_KEY>' \
  -n llm-stack

Step 2: Create Kubernetes Manifests (Deployment & Service)

Create a file named llm-stack.yaml (or similar) and paste the following content. This file defines the Deployments and Services for both LiteLLM and OpenWebUI.

Important:

Review the image tags (ghcr.io/berriai/litellm:latest, ghcr.io/open-webui/open-webui:main). Consider using specific version tags for production stability instead of latest or main.

Adjust replicas if needed.

Note the service.beta.kubernetes.io/aws-load-balancer-type: "nlb" annotation in the OpenWebUI service. This is specific to AWS EKS for using a Network Load Balancer. Remove or adjust it based on your cloud provider or if you're using a different Ingress controller or NodePort.

apiVersion: v1
kind: Service
metadata:
  name: litellm-service
  namespace: llm-stack
  labels:
    app: litellm
spec:
  selector:
    app: litellm
  ports:
    - protocol: TCP
      port: 8000 # Port the service listens on
      targetPort: 4000 # Port the LiteLLM container listens on (LiteLLM default)
  type: ClusterIP # Only needs to be reachable within the cluster

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: litellm-deployment
  namespace: llm-stack
  labels:
    app: litellm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: litellm
  template:
    metadata:
      labels:
        app: litellm
    spec:
      containers:
      - name: litellm
        image: ghcr.io/berriai/litellm:latest # Use a specific tag for stability
        ports:
        - containerPort: 4000
        # Command to start LiteLLM, expose Gemini Pro, listen on all interfaces
        # It will automatically pick up GEMINI_API_KEY from env
        command: ["litellm", "--model", "gemini/gemini-pro", "--host", "0.0.0.0", "--port", "4000", "--debug"]
        # Mount the API key from the Secret as an environment variable
        env:
        - name: GEMINI_API_KEY
          valueFrom:
            secretKeyRef:
              name: gemini-api-key-secret
              key: api-key
        # Optional: Add resource requests and limits for better scheduling
        # resources:
        #   requests:
        #     memory: "256Mi"
        #     cpu: "100m"
        #   limits:
        #     memory: "512Mi"
        #     cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: openwebui-service
  namespace: llm-stack
  labels:
    app: open-webui
  # Add annotations specific to your cloud provider's LoadBalancer if needed
  # Example for AWS NLB
  # annotations:
  #   service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: open-webui
  ports:
    - protocol: TCP
      port: 80 # Port the LoadBalancer listens on
      targetPort: 8080 # Port the OpenWebUI container listens on
  type: LoadBalancer # Exposes the service externally using a cloud provider's load balancer
  # Or use NodePort for simpler setups/local testing
  # type: NodePort
  # Then access via <NodeIP>:<NodePort>

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openwebui-deployment
  namespace: llm-stack
  labels:
    app: open-webui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: open-webui
  template:
    metadata:
      labels:
        app: open-webui
    spec:
      containers:
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END
