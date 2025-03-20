---
title: Running LLM and GenAI on Intel Arc GPU - Part 2
description: Make use of Intel Arc Series GPU to run Open WebUI with Ollama to interact with Large Language Models (LLM) and Generative AI (GenAI)
author: eleiton
date: 2025-03-20 00:00:00 +0100
categories: [AI]
tags: [GenAI,LLM]
pin: true
math: true
mermaid: true
image:
  path: /assets/img/2025-03-20/sloth.jpeg
  alt: Slooth's Arctic Expedition - Powered by Intel Arc
---
## Introduction
In my previous post, I outlined the steps involved in installing Ollama on your Intel-powered Linux PC and leveraging the Arc Graphics GPU for running LLM models to power chatbots

Today we'll unlock the power of Generative AI! We’ll be exploring how our GPU can be used to create stunning visuals using OpenSource Software.

We’ll be covering how to create images programmatically using Python and the Jupyter Notebook, which provides a very convenient way to get started with generative AI! 

We’ll also show you a user-friendly approach to image creation through SD.Next.

After reading this post, you'll understand how to:

- [x] Seamlessly deploy a single Docker-based solution that integrates all necessary components
- [x] Streamline Stable Diffusion capabilities with SD.Next's optimization tools
- [x] Maximize performance and efficiency using Intel Arc Series GPUs and Intel Extension for PyTorch

## TL;DR
For developers who prefer to dive straight into the code, I've got you covered!
The docker-based deployment solution is available on GitHub. Simply click [this link](https://github.com/eleiton/ollama-intel-arc) to access the code repository and start building your own AI-powered workflow today.

## Requirements
1. GPU: Intel Arc Series
2. Operating System: Linux
3. Container Management: Podman or Docker
4. Python: Version 3.10 or higher

## PyTorch
We will be leveraging PyTorch, a powerful open-source machine learning library built upon the Torch foundation, ideal for applications like computer vision and natural language processing. 
Specifically, we’ll be utilizing Intel Extension for PyTorch, which provides significant performance enhancements specifically tailored for Intel hardware.

To keep our project organized, we'll create a folder called `~/gen-ai`. This folder will hold files and the models we download from the internet to generate images.

```shell
mkdir ~/gen-ai
mkdir ~/gen-ai/data
mkdir ~/gen-ai/app
mkdir ~/gen-ai/models
mkdir ~/gen-ai/huggingface
```

Let's now run Intel's official docker image for PyTorch:

```shell
podman run -it --rm \
  --device /dev/dri \
  -v ~/gen-ai/huggingface:/root/.cache/huggingface \
  -p 9999:9999 \
  intel/intel-extension-for-pytorch:2.6.10-xpu
```

With the `--device` flag we're allowing the container access to our GPU.

The `-v` flag instructs Docker to map the container’s folder storing model files to our host folder. This ensures that these files persist across container runs. 

With `-p` we have made the port 9999 available for our Jupyter notebooks.

After running the command, you should be presented with a command prompt within the container.

Let's go ahead and install jupyter.
```shell
pip install -q jupyter diffusers transformers accelerate
jupyter notebook --allow-root --ip 0.0.0.0 --port 9999
```

This will download jupyter, so will take some time, but after it runs, it should show you something similar to this output:

```bash
[I ServerApp] Jupyter Server 2.15.0 is running at:
[I ServerApp] http://6b3444b23b4d:9999/tree?token=e54599e065c0
[I ServerApp]     http://127.0.0.1:9999/tree?token=e54599e065c0
[I ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[W ServerApp] No web browser found: Error('could not locate runnable browser').
[C ServerApp]  
    To access the server, open this file in a browser:
        file:///root/.local/share/jupyter/runtime/jpserver-8-open.html
    Or copy and paste one of these URLs:
        http://6b3444b23b4d:9999/tree?token=e54599e065c0
        http://127.0.0.1:9999/tree?token=e54599e065c0
```
Follow the link to `127.0.0.1` with the given token, and this should open jupyter in your browser.

Once there, open a new notebook with a Python 3 kernel: `File -> New -> Notebook`

And run the following code:

```python
import intel_extension_for_pytorch as ipex
import torch
from diffusers import StableDiffusionPipeline

# load the Stable Diffusion model
pipe = StableDiffusionPipeline.from_pretrained("stable-diffusion-v1-5/stable-diffusion-v1-5",         
    torch_dtype=torch.float16,
    safety_checker = None,
    requires_safety_checker = False)

# move the model to Intel Arc GPU
pipe = pipe.to("xpu")

# generate the image
pipe(
    prompt="prosciutto e funghi pizza",
    negative_prompt="blurry, distorted, watermark, text",
    num_inference_steps=8,
    guidance_scale=7,
    prng_seed = torch.manual_seed(106),
    width=512,
    height=512
).images[0]
```

In lines 6 - 9 we're loading the model we want to use for Stable Diffusion.  For the purposes of this example, we'll use [stable-diffusion-v1-5](https://huggingface.co/stable-diffusion-v1-5/stable-diffusion-v1-5).
The first time you run this code it will download all necessary files for the model to run, so it will take some time depending on your network connection.

In line 12 we indicate we want to run the pipeline in our discrete GPU.

In lines 15 - 23 we generate the model.  Let's dig into how this is happening:

* `prompt`: Here we indicate what is it that we want to generate.
* `negative_prompts`: Crucially important. These tell the model what not to generate.
* `num_reference_steps`:  Inference steps controls how many steps will be taken during the image generation process. The higher the value, the more steps that are taken to produce the image (also more time).
* `guidance_scale`: This controls how strongly the model adheres to your prompt.
* `prng_seed`: Using a fixed seed ensures that you get the exact same image every time you run the code with the same prompt and parameters. This is invaluable for debugging and comparing different settings.
* `width and height`: Specify the desired image dimensions. Larger images take longer to generate and require more memory.

Running this code should render something like this:

<video width="100%" height="auto" autoplay loop muted controls>
  <source src="/assets/img/2025-03-20/jupyter.webm" type="video/webm"/>
</video>

## SD.Next

Numerous open-source tools are available for text-to-image generation, offering diverse creative possibilities.

Here we'll be focusing on one that can be [integrated](https://docs.openwebui.com/tutorials/images#using-image-generation) with OpenWebUI.

Let's try out SD.Next, which is based on [Stable Diffusion Web UI](https://github.com/AUTOMATIC1111/stable-diffusion-webui).

To facilitate running this tool, we'll build a Dockerfile. This Dockerfile will define the environment and dependencies needed to execute the tool within a container.

Create a file named `Dockerfile` with this content in it.

```shell
FROM intel/intel-extension-for-pytorch:2.6.10-xpu

# essentials
RUN apt-get update && \
    apt-get install -y --no-install-recommends --fix-missing \
    software-properties-common \
    build-essential \
    ca-certificates \
    wget \
    gpg \
    git

# set paths to use with sdnext
ENV SD_DOCKER=true
ENV SD_DATADIR="/mnt/data"
ENV SD_MODELSDIR="/mnt/models"

# git clone and start sdnext
RUN echo '#!/bin/bash\ngit status || git clone https://github.com/vladmandic/sdnext.git .\npython /app/launch.py "$@"' | tee /bin/startup.sh
RUN chmod 755 /bin/startup.sh

# run sdnext
WORKDIR /app
ENTRYPOINT [ "startup.sh", "-f", "--use-ipex", "--uv", "--listen", "--debug", "--api-log", "--log", "sdnext.log" ]

# expose port
EXPOSE 7860

# stop signal
STOPSIGNAL SIGINT
```

Now let's build a docker image using this file:
```shell
podman build \
  --tag sdnext-ipex \
  --file ./Dockerfile .
```

And let's run a docker container created from this docker image:
```shell
podman run -it \
  --name sdnext-ipex \
  --device /dev/dri \
  -p 7860:7860 \
  -v ~/gen-ai/app:/app \
  -v ~/gen-ai/data:/mnt/data \
  -v ~/gen-ai/models:/mnt/models \
  -v ~/gen-ai/huggingface:/root/.cache/huggingface \
  sdnext-ipex:latest
```

Go to [http://127.0.0.1:7860/](http://127.0.0.1:7860/) and this should open the SD.Next site.

Play around with models, prompts, guidance, steps, etc to fine-tune your image generation.

For example, I’ve created an image of a Costa Rican sloth embarking on a fantasy journey to the North Pole! Let's explore some inputs and see what kind of results you can achieve.

<video width="100%" height="auto" autoplay muted controls>
  <source src="/assets/img/2025-03-20/sdnext.webm" type="video/webm"/>
</video>

Now that you’ve learned how to begin your image generation journey with your Intel Arc GPU – whether through coding in Python with PyTorch or utilizing open-source software – we’re excited to move on. In the next part of this series, we’ll introduce Open WebUI and demonstrate how to seamlessly integrate it with Ollama for a unified image generation interface.

## References
GitHub Repository: <https://github.com/eleiton/ollama-intel-arc>
