# Deploy and Serve an LLM on OpenShift AI

> **Latest comprehensive guide (RHOAI 3.4 / 3.5):** https://rh-aiservices-bu.github.io/rhoai-maas-guide/modules/main/index.html
> **Source repo:** https://github.com/rh-aiservices-bu/rhoai-maas-guide

A self-service workshop for deploying a large language model on Red Hat OpenShift AI (RHOAI), enrolling it in Models-as-a-Service (MaaS) for managed access, and connecting it to VS Code as an AI coding assistant.

Created: 2026-07-10
Modified: 15 Sep 2026

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Section 0: Prerequisites](#section-0-prerequisites)
- [Section 1: Set Up Your Namespace](#section-1-set-up-your-namespace)
- [Section 2: Deploy the Model](#section-2-deploy-the-model)
- [Section 3: Register Model with MaaS](#section-3-register-model-with-maas)
- [Section 4: Create a Subscription](#section-4-create-a-subscription)
- [Section 5: Generate an API Key](#section-5-generate-an-api-key)
- [Section 6: Connect to VS Code](#section-6-connect-to-vs-code)
- [Appendix: Troubleshooting](#appendix-troubleshooting)

---

## Architecture Overview

```
Developer IDE (VS Code)
    |
    | HTTPS + API Key
    v
MaaS Gateway (Red Hat Connectivity Link)
    |-- Authorino (authentication, API key validation)
    |-- Limitador (rate limiting, token quotas)
    |
    v
LLMInferenceService (vLLM + Qwen3.6-27B-FP8)
    |
    | hf:// pull on first startup
    v
HuggingFace Hub (model weights)
```

**What you will build:**

| Component | Resource | Purpose |
|-----------|----------|---------|
| Model server | `LLMInferenceService` | Runs Qwen3.6-27B-FP8 on a GPU via vLLM |
| MaaS registration | `MaaSModelRef` | Makes the model discoverable through the MaaS gateway |
| Access policy | `MaaSSubscription` | Defines who can use the model and at what rate |
| API key | (created via UI) | Authenticates requests from your IDE |

---

## Section 0: Prerequisites

### MaaS infrastructure setup reference

The MaaS platform infrastructure (operators, gateway, Kuadrant, database) must be deployed before this workshop. The admin should follow the RHOAI MaaS installation guide:

- [RHOAI Models-as-a-Service (MaaS) Guide](https://github.com/rh-aiservices-bu/rhoai-maas-guide) -- Kustomize manifests and automation scripts for deploying RHOAI 3.4+ MaaS on OpenShift
- [Full documentation site](https://rh-aiservices-bu.github.io/rhoai-maas-guide/)

### What the admin has already configured

- RHOAI operator installed and configured
- MaaS infrastructure deployed (Authorino, Limitador, MaaS gateway) -- see [MaaS Guide](https://github.com/rh-aiservices-bu/rhoai-maas-guide) for setup
- GPU worker nodes provisioned (e.g., `g6e.4xlarge` with NVIDIA L40S)
- NVIDIA GPU Operator and Node Feature Discovery installed
- HTPasswd identity provider configured (your login credentials)
- Your namespace (`workshop-<username>`) created
- RBAC permissions for namespace labeling, model deployment, and subscription creation

### What you need

- `oc` CLI installed ([download](https://console.redhat.com/openshift/downloads))
- OpenShift login credentials (username and password provided by admin)
- VS Code with the GitHub Copilot extension installed
- A terminal (macOS Terminal, Linux shell, or Windows WSL)

### Set your username variable

Every command in this workshop uses `$USERNAME`. Set it once at the top of your terminal session:

```bash
export USERNAME=<your-username>
```

For example, if your OpenShift username is `usera`:

```bash
export USERNAME=usera
```

### Log in to OpenShift

```bash
oc login -u $USERNAME -p <your-password> --server=https://api.<cluster-domain>:6443
```

Verify access:

```bash
oc whoami
```

---

<details>
<summary><h2>Section 1: Set Up Your Namespace</h2></summary>

The admin has pre-created a namespace for each participant and granted the required RBAC permissions.

### 1.1 Switch to your project

```bash
oc project workshop-$USERNAME
```

### 1.2 Verify access

```bash
oc whoami
oc project
```

Confirm the output shows your username and `workshop-<username>` as the current project.

### 1.3 Label your namespace for MaaS gateway access

This label allows the MaaS gateway to route traffic to models deployed in your namespace:

```bash
oc label namespace workshop-$USERNAME maas.opendatahub.io/gateway-access=true
```

### 1.4 Verify the label

```bash
oc get namespace workshop-$USERNAME --show-labels
```

Confirm the output includes `maas.opendatahub.io/gateway-access=true`.

</details>

---

<details>
<summary><h2>Section 2: Deploy the Model</h2></summary>

You will deploy [Qwen3.6-27B-FP8](https://huggingface.co/RedHatAI/Qwen3.6-27B-FP8), a 27-billion parameter model quantized to FP8 for efficient GPU inference. The model weights are pulled directly from HuggingFace on first startup.

### 2.1 Review the manifest

The deployment manifest is at `manifests/01-llminferenceservice.yaml`. Key settings:

| Field | Value | Purpose |
|-------|-------|---------|
| `spec.model.uri` | `hf://RedHatAI/Qwen3.6-27B-FP8` | Pull weights from HuggingFace |
| `spec.router.gateway.refs` | `maas-default-gateway` | Route through MaaS gateway |
| `--max-model-len=32768` | 32K context window | Supports IDE-sized prompts |
| `--enable-auto-tool-choice` | Tool calling support | Enables function/tool calling |
| `nvidia.com/gpu: "1"` | 1 GPU requested | Requires a GPU node |

### 2.2 Apply the manifest

Substitute your username and apply:

```bash
sed "s/REPLACE_USERNAME/$USERNAME/g" manifests/01-llminferenceservice.yaml | oc apply -f -
```

### 2.3 Monitor startup

The first startup downloads ~27 GB of model weights from HuggingFace. This typically takes **15-20 minutes**.

Watch pod progress:

```bash
oc get pods -n workshop-$USERNAME -w
```

You will see the pod go through these stages:
1. `Pending` -- waiting for GPU node scheduling
2. `Init:0/1` -- storage initializer downloading model weights
3. `Running` -- vLLM loading the model into GPU memory
4. `1/1 Running` -- model is ready to serve

### 2.4 Verify deployment

Check that the `LLMInferenceService` reports `READY: True`:

```bash
oc get llminferenceservice qwen36-27b -n workshop-$USERNAME
```

Expected output:

```
NAME         READY   AGE
qwen36-27b   True    20m
```

If `READY` shows `False`, check the troubleshooting appendix.

</details>

---

<details>
<summary><h2>Section 3: Register Model with MaaS</h2></summary>

A `MaaSModelRef` tells the MaaS gateway that your model exists and should be reachable through the unified API endpoint.

### 3.1 Apply the MaaSModelRef

```bash
sed "s/REPLACE_USERNAME/$USERNAME/g" manifests/02-maas-model-ref.yaml | oc apply -f -
```

### 3.2 Verify registration

```bash
oc get maasmodelref -n workshop-$USERNAME
```

Expected output:

```
NAME         STATUS   AGE
qwen36-27b   Ready    10s
```

The status should show `Ready`. If it shows `Failed`, see the troubleshooting appendix.

After registration, your model is accessible at:

```
https://maas.apps.<cluster-domain>/workshop-<username>/qwen36-27b
```

However, you cannot call it yet -- you need a subscription and an API key first.

</details>

---

<details>
<summary><h2>Section 4: Create a Subscription</h2></summary>

A `MaaSSubscription` defines **who** can access the model and **at what rate**. Without a subscription, API key creation is not possible.

### Option A: Apply via CLI (recommended)

The manifest at `manifests/03-maas-subscription.yaml` creates a subscription with:
- **Owner**: your user (`$USERNAME`)
- **Rate limit**: 10,000 tokens per minute
- **Priority**: 10

Apply it:

```bash
sed "s/REPLACE_USERNAME/$USERNAME/g" manifests/03-maas-subscription.yaml | oc apply -f -
```

Verify:

```bash
oc get maassubscription -n models-as-a-service | grep $USERNAME
```

### Option B: Create via RHOAI Dashboard UI

1. Open the RHOAI Dashboard in your browser
2. Navigate to **Models as a Service** in the left sidebar
3. Click the **Subscriptions** tab
4. Click **Create subscription**
5. Fill in:
   - **Name**: `qwen36-27b-<your-username>`
   - **Model**: select `qwen36-27b` from your namespace
   - **Owner**: add your username
   - **Token rate limit**: `10000` tokens per `1m`
6. Click **Create**

</details>

---

<details>
<summary><h2>Section 5: Generate an API Key</h2></summary>

API keys are generated through the RHOAI Dashboard. Each key is tied to your subscription.

### 5.1 Create the API key

1. Open the RHOAI Dashboard
2. Navigate to **Models as a Service** in the left sidebar
3. Click the **API Keys** tab
4. Click **Create API key**
5. Select your subscription (`qwen36-27b-<your-username>`)
6. Give it a name (e.g., `workshop-key`)
7. Click **Create**
8. **Copy the key immediately** -- it will not be shown again. It starts with `sk-oai-...`

### 5.2 Test the API key with curl

Replace `<cluster-domain>` and `<your-api-key>` with your actual values:

```bash
curl -sk \
  "https://maas.apps.<cluster-domain>/workshop-$USERNAME/qwen36-27b/v1/chat/completions" \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen36-27b",
    "messages": [{"role": "user", "content": "Hello, what model are you?"}],
    "max_tokens": 100
  }' | python3 -m json.tool
```

You should receive a JSON response with the model's reply. If you get a `401` or `403`, double-check your API key and subscription.

</details>

---

<details>
<summary><h2>Section 6: Connect to VS Code</h2></summary>

Use VS Code custom endpoint (BYOK) feature to connect GitHub Copilot to your self-hosted model.

### 6.1 Handle self-signed TLS certificates

OpenShift uses self-signed certificates by default. VS Code (Node.js) will reject these unless told otherwise.

**macOS:**

```bash
launchctl setenv NODE_TLS_REJECT_UNAUTHORIZED 0
```

**Linux:**

```bash
echo 'export NODE_TLS_REJECT_UNAUTHORIZED=0' >> ~/.bashrc
source ~/.bashrc
```

After setting this, **quit VS Code completely** (`Cmd+Q` on macOS, not just close the window) and reopen it.

### 6.2 Add a custom endpoint

1. Open VS Code
2. Press `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Linux/Windows)
3. Search for **Chat: Manage Language Models**
4. Select **Custom Endpoint**
5. Enter a group name (e.g., `RHOAI Workshop`)
6. When prompted, paste your MaaS API key (`sk-oai-...`). VS Code stores it securely in encrypted Secret Storage.

### 6.3 Configure the model

VS Code creates a `chatLanguageModels.json` file. Edit it to match the following configuration. Replace `<cluster-domain>` with your actual cluster domain and `<username>` with your OpenShift username:

```json
[
    {
        "name": "Custom Endpoint",
        "vendor": "customendpoint",
        "apiKey": "${input:chat.lm.secret.<hash>}",
        "apiType": "chat-completions",
        "models": [
            {
                "id": "qwen36-27b",
                "name": "qwen36-27b",
                "url": "https://maas.apps.<cluster-domain>/workshop-<username>/qwen36-27b/v1/chat/completions",
                "toolCalling": true,
                "maxInputTokens": 28000,
                "maxOutputTokens": 2048,
                "requestHeaders": {
                    "Authorization": "Bearer ${apiKey}"
                }
            }
        ]
    }
]
```

**Important notes on the config:**

| Field | Detail |
|-------|--------|
| `apiKey` | Do NOT replace the `${input:chat.lm.secret.<hash>}` value with the raw key. This is auto-generated by VS Code and references the encrypted secret. To update your key: right-click the endpoint group and select **Update API Key**. |
| `url` | Must include the full path ending in `/v1/chat/completions`. Must use `https://`. |
| `requestHeaders` | Required. Without this block, VS Code does not send the Authorization header correctly to the MaaS gateway. |
| `maxInputTokens` | Set to `28000`. The model supports 32768, but leaving headroom avoids a known Copilot context tree pruning bug ("No lowest priority node found"). |

### 6.4 Test the connection

1. Open GitHub Copilot Chat in VS Code (click the chat icon in the sidebar)
2. At the bottom of the chat panel, click the model selector dropdown
3. Select your custom model (`qwen36-27b`)
4. Send a test message: "Hello, what model are you?"
5. You should receive a response from Qwen3.6-27B

</details>

---

<details>
<summary><h2>Appendix: Troubleshooting</h2></summary>

### Model pod stuck in Pending

**Symptom:** Pod stays in `Pending` state and never progresses.

**Cause:** No GPU node available, or the GPU node has a taint the pod does not tolerate.

**Check:**

```bash
oc describe pod -l app=qwen36-27b -n workshop-$USERNAME | grep -A5 Events
oc get nodes -l nvidia.com/gpu.present=true
```

**Fix:** Confirm GPU nodes exist and are schedulable. The manifest includes a toleration for `nvidia.com/gpu`, which handles the standard GPU taint.

---

### LLMInferenceService READY: False (RefsInvalid)

**Symptom:** `oc get llminferenceservice` shows `READY: False`.

**Cause:** The HTTPRoute cannot find the referenced gateway (`maas-default-gateway`).

**Check:**

```bash
oc get llminferenceservice qwen36-27b -n workshop-$USERNAME -o yaml | grep -A10 status
```

**Fix:** Verify the `maas-default-gateway` exists in the `openshift-ingress` namespace:

```bash
oc get gateway maas-default-gateway -n openshift-ingress
```

If it does not exist, contact the workshop admin -- MaaS infrastructure may not be fully set up.

---

### MaaSModelRef status shows Failed

**Symptom:** `oc get maasmodelref` shows `Failed` instead of `Ready`.

**Cause:** The `LLMInferenceService` is not yet ready, or the namespace is not labeled for MaaS gateway access.

**Fix:**

1. Ensure the model is `READY: True` first
2. Verify the namespace label:

```bash
oc get namespace workshop-$USERNAME \
  -o jsonpath='{.metadata.labels.maas\.opendatahub\.io/gateway-access}'
```

If empty, re-apply the label:

```bash
oc label namespace workshop-$USERNAME maas.opendatahub.io/gateway-access=true
```

---

### curl returns 401 (token expired or invalid)

**Symptom:** `curl` test returns `401 token expired or invalid`.

**Fix:**

- Verify the API key is correct and has not been revoked
- Ensure the URL uses `https://` (not `http://`)
- Confirm the subscription exists and lists your user:

```bash
oc get maassubscription -n models-as-a-service -o yaml | grep -A5 $USERNAME
```

---

### VS Code error: "No lowest priority node found"

**Symptom:** VS Code Copilot shows error `No lowest priority node found (path: ...)`.

**Cause:** `maxInputTokens` in `chatLanguageModels.json` is too low, triggering a Copilot internal context tree pruning bug.

**Fix:** Set `maxInputTokens` to `28000` (not higher than `--max-model-len` minus `maxOutputTokens`).

---

### VS Code error: context length exceeded

**Symptom:** Error message mentioning "maximum context length is 32768 tokens".

**Cause:** The combined input + output tokens exceed the model `--max-model-len`.

**Fix:** Reduce `maxInputTokens` in `chatLanguageModels.json`. Recommended: `28000` with `maxOutputTokens: 2048` gives a total of `30048`, within the 32768 limit.

</details>

---

## File Reference

| File | Description |
|------|-------------|
| `manifests/01-llminferenceservice.yaml` | LLMInferenceService -- deploys Qwen3.6-27B-FP8 from HuggingFace |
| `manifests/02-maas-model-ref.yaml` | MaaSModelRef -- registers the model with MaaS gateway |
| `manifests/03-maas-subscription.yaml` | MaaSSubscription -- creates access policy with rate limits |
