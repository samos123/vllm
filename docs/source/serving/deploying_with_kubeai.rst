.. _on_cloud:

Deploying with KubeAI
================================================


KubeAI (https://github.com/substratusai/kubeai) is a Kubernetes operator with built-in support for running and scaling vLLM instances.

Features provided by KubeAI:

- Autoscaling based on concurrent requests.
- Scale from 0 without requiring any dependencies.
- Single endpoint for all vLLM instances serving different models.
- Validated model configurations for GPU, TPU and CPU.


Prerequisites
-------------

- You have a Kubernetes cluster.
- You have `helm` installed.

Install KubeAI
--------------

Add the KubeAI helm repository:

.. code-block:: console

  helm repo add kubeai https://www.kubeai.org


Install the KubeAI operator:

.. code-block:: console

  helm install kubeai kubeai/kubeai --wait --timeout 10m

Afterwards you should see the KubeAI operator running:

.. code-block:: console

  kubectl get pods

KubeAI provides a proxy service that sits in front of the vLLM instances. By default
the KubeAI service is exposed within the Kubernetes cluster only.

You can access the KubeAI service within the Kubernetes cluster on the following endpoints:

- ``http://kubeai/openai/v1``
- ``http://kubeai.{namespace}.svc.cluster.local/openai/v1``

To access the KubeAI endpoint locally you can setup a port-forward:

.. code-block:: console

  kubectl port-forward svc/kubeai 8000:80

Then you can access the KubeAI service at `http://localhost:8000/openai/v1`.

For example to list all the models you can use the following command:

.. code-block:: console

  curl http://localhost:8000/openai/v1/models

Deploying a GPU vLLM instance
-----------------------------

Create a file named `llama-3.2-11b-vision-instruct.yaml` with the following content:

.. code-block:: yaml

  apiVersion: kubeai.org/v1
  kind: Model
  metadata:
    name: llama-3.2-11b-vision-instruct-l4
  spec:
    features: [TextGeneration]
    owner:
    url: hf://neuralmagic/Llama-3.2-11B-Vision-Instruct-FP8-dynamic
    engine: VLLM
    args:
      - --max-model-len=8192
      - --max-num-batched-token=8192
      - --gpu-memory-utilization=0.95
      - --enforce-eager
      - --disable-log-requests
      - --max-num-seqs=8
    env:
      VLLM_WORKER_MULTIPROC_METHOD: spawn
    minReplicas: 1
    maxReplicas: 1
    targetRequests: 32
    resourceProfile: nvidia-gpu-l4:1

Deploy the model:

.. code-block:: console

  kubectl apply -f llama-3.2-11b-vision-instruct.yaml

Wait until the newly created model pod is in the Running and Ready state:

.. code-block:: console

  kubectl get pods

You can access the model at ``$ENDPOINT/openai/v1/chat/completions``.

Lets test the model using the OpenAI Python client. Run the following Python script:

.. code-block:: python

  from openai import OpenAI
  
  # Modify OpenAI's API key and API base to use KubbeAI's API server.
  openai_api_key = "ignored"
  # Replace this with http://kubeai/openai/v1 when running inside the K8s cluster.
  # This assumes kubectl port-forward svc/kubeai 8000:80 has been run.
  openai_api_base = "http://localhost:8000/openai/v1"
  
  client = OpenAI(
      api_key=openai_api_key,
      base_url=openai_api_base,
  )
  
  models = client.models.list()
  model = models.data[0].id
  
  # Single-image input inference
  image_url = "https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"
  
  ## Use image url in the payload
  chat_completion_from_url = client.chat.completions.create(
      messages=[{
          "role":
          "user",
          "content": [
              {
                  "type": "text",
                  "text": "What's in this image?"
              },
              {
                  "type": "image_url",
                  "image_url": {
                      "url": image_url
                  },
              },
          ],
      }],
      model=model,
      max_tokens=64,
  )
  
  print(chat_completion_from_url.choices[0].message)


Now let's run a benchmark using the vLLM benchmarking script:

.. code-block:: console

  git clone https://github.com/vllm-project/vllm.git
  cd vllm/benchmarks
  wget https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered/resolve/main/ShareGPT_V3_unfiltered_cleaned_split.json
  python3 benchmark_serving.py --backend openai \
      --base-url http://localhost:8000/openai \
      --dataset-name=sharegpt --dataset-path=ShareGPT_V3_unfiltered_cleaned_split.json \
      --model llama-3.2-11b-vision-instruct-l4 \
      --seed 12345 --tokenizer neuralmagic/Llama-3.2-11B-Vision-Instruct-FP8-dynamic
