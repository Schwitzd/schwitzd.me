+++
title = 'My Openvino Notes'
date = 2025-12-07T07:52:41Z
draft = true
+++

These are my notes explaining on how to run models on an Arch Linux machine using the [OpenVINO model server](https://docs.openvino.ai/2025/model-server/ovms_what_is_openvino_model_server.html) inside a Docker container. My Medion notebook is equipped with both an Intel [NPU](https://en.wikipedia.org/wiki/Neural_processing_unit) and an Intel ARC GPU. I tried running models on both.

## Requirements

1. Install required driver packages:

    - **GPU**: intel-compute-runtime
    - **NPU**: intel-npu-driver

1. Add the user to groups `render` and `video`:

    ```sh
    sudo usermod -aG render,video "$USER"
    ```

1. To check that everything has been installed on the local machine, run the following code snippet inside a Python virtual environment with the `openvino` package installed.

    ```python
    from openvino import Core

    ie = Core()
    devices = ie.available_devices
    print("Available devices:", devices)
    ```

    It should return the following output:

    ```sh
    Available devices: ['CPU', 'GPU', 'NPU']
    ```

## `/dev/accel` vs `/dev/dri`

On Intel systems that include both a GPU and an NPU, Linux exposes the two through different device nodes. They may look similar at first glance, but they represent very different pieces of hardware.

**`/dev/accel`** is created by Intel's NPU driver and exposes the **Neural Processing Unit**, a hardware block dedicated to running neural network inference. The NPU focuses on tensor operations and model execution with very low power consumption, making it ideal for language models, vision pipelines, and other AI workloads. This device node is the direct path software uses to send inference tasks to the NPU.

**`/dev/dri`** is part of the Linux **Direct Rendering Infrastructure** and exposes the **GPU and display subsystem**. Through this device node, software can access the GPU for rendering, video acceleration, and general-purpose compute. AI frameworks can also use the GPU through `/dev/dri`, but this goes through the GPU's compute engines rather than a dedicated neural accelerator.

## Models

### NPU model

```yaml
services:
  ovms:
    image: openvino/model_server:latest-gpu
    container_name: ovms
    user: "${UID}:${GID}"
    devices:
      - "/dev/accel:/dev/accel"
    command: >
      --source_model OpenVINO/Qwen3-8B-int4-cw-ov
      --model_repository_path /models
      --task text_generation
      --rest_port 8000
      --target_device NPU
      --tool_parser hermes3
      --enable_prefix_caching true
    volumes:
      - ./models:/models
    ports:
      - "8000:8000"
    restart: unless-stopped
```

### GPU model

```yaml
services:
  ovms-gpu:
    image: openvino/model_server:latest-gpu
    container_name: ovms-gpu
    user: "${UID}:${GID}"
    devices:
      - "/dev/dri:/dev/dri"
    command: >
      --source_model OpenVINO/Phi-3-mini-4k-instruct-fp16-ov
      --model_repository_path /models
      --task text_generation
      --rest_port 8000
      --target_device GPU
      --tool_parser phi4
      --enable_prefix_caching true
    volumes:
      - ./models:/models
    ports:
      - "8000:8000"
    restart: unless-stopped
```

### Run & talk with the model

#### Run the model

Save the file as `docker-compose.yaml` and start the service with:

```sh
docker compose up
```

When the container starts, the log will report which hardware devices OpenVINO can use. Look for a line similar to:

```sh
ovms  | [2025-12-08 10:37:28.294][1][modelmanager][info] Available devices for Open VINO: CPU, GPU
```

This message shows the accelerator options detected inside the container (e.g., CPU, GPU, NPU). It's a quick way to verify which processor is available for inference.

## Talk to the model

You can interact with the model using a basic `curl` request. This sends a prompt and retrieves a generated response directly from OVMS:

```sh
curl -X POST "http://localhost:8000/v3/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "OpenVINO/Phi-3-mini-4k-instruct-fp16-ov",
    "messages": [
      { "role": "user", "content": "Explain what an NPU is in one sentence." }
    ],
    "max_tokens": 64,
    "temperature": 0.7
  }'
```

This example sends a single question and returns the model's generated answer.

## Next steps

The next step is to empower the running model by adding tools. For example, it should have the ability to fetch webpages. I researched online and found that there are different ways to accomplish this task, but the best option for me seems to be using Ollama with OpenVINO as a backend. However, the integration seems to be in the early stages and is not very Docker-friendly at the moment. You may be interested in this article: [Ollama Integrated with OpenVINO, Accelerating DeepSeek Inference][https://blog.openvino.ai/blog-posts/ollama-integrated-with-openvino-accelerating-deepseek-inference).

## Glossary

**Inference** - is the phase where a trained AI model is used to make predictions. Instead of learning new patterns, the model applies what it already knows to produce an output. For example, generating text, recognizing an image, or classifying sound. It's the "use" stage of a model, not the training stage.

**Temperature** – controls how random or deterministic a language model's output is. A low value (close to 0) makes the model predictable and focused, choosing the most likely words. A higher value adds creativity and variation, allowing less common words or ideas to appear. It's essentially a "creativity dial" for text generation, typically ranging between **0.0 and 2.0**.

**Multimodal** – the model can understand or generate more than just text. It can work with multiple kinds of data (images, audio, video, or structured documents) and combine those signals when reasoning. A multimodal model might read a paragraph and look at an image to answer a question, analyze charts inside a PDF, or describe a photo from a URL.
