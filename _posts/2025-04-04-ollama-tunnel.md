---
title: Take your Ollama server online
description: A secure gateway with Cloudflare Zero Tunnel, OAuth and Nginx
author: eleiton
date: 2025-04-03 00:00:00 +0100
categories: [AI]
tags: [LLM,Cloudflare,Nginx,OAuth]
pin: true
math: true
mermaid: true
image:
  path: /assets/img/2025-04-03/secure-network.jpeg
  alt: Protect your Ollama server
---
## Introduction
If you've already experimented with Ollama locally on your home server, you're now familiar with running Large Language Models (LLMs) within your private network. However, it's time to take the next step: exposing your Ollama service securely online, allowing seamless access to your LLMs from anywhere in the world.

In this article, we'll explore how to achieve this using Cloudflare Zero Tunnel and Nginx, leveraging Docker containers for an easy and containerized setup.

## TL;DR
For developers who prefer to dive straight into the code, I've got you covered!
The docker-based deployment solution is available on GitHub. Simply click [this link](https://github.com/eleiton/ollama-tunnel) to access the code repository and start protecting your LLM infrastructure today.

## Requirements
1. Ollama up and running at the default port 11434.
2. Container Management: Podman or Docker
3. Your own domain name.
4. A valid Cloudflare Tunnel token.

If you don't have Ollama, follow the [official documentation](https://github.com/ollama/ollama/blob/main/docs/README.md) for installation instructions.

If you don't have a Cloudflare Tunnel token, follow the[ official documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/) for installation instructions.

## Ollama

Make sure your Ollama server is up and running by running this command:

```bash
curl http://localhost:11434
```
If you see a `Ollama is running` message in your console, it indicates that the Ollama API is up and accessible. However, this also highlights a critical security concern: the lack of protection for our Ollama API. 

Specifically, there are two risks to be aware of:

* No authentication required: Since no authentication is implemented, anyone with access to the server can directly access the Ollama API, posing a significant security risk.
* No TLS encryption: The Ollama API can be accessed via regular HTTP without any encryption, making it vulnerable to eavesdropping and tampering.

To mitigate the first risk, we will employ Nginx as a reverse proxy server, and introduce the need for an API key to access our server.

For the second concern we will rely on Cloudflare Zero Tunnel.

## Containers

To simplify the installation process, we'll use Docker containers to run both Cloudflare Tunnel and Nginx. 
This approach allows for a simple deployment of our application without requiring manual installations or management of underlying infrastructure.

Let's have a look at our [docker-compose](https://github.com/eleiton/ollama-tunnel/blob/main/docker-compose.yml) file.

```yaml
version: "3"
services:
  nginx:
    container_name: nginx
    image: nginx:latest
    volumes:
      - ./templates:/etc/nginx/templates:ro
    env_file: .env
    restart: always

  cloudflared:
    container_name: cloudflared
    image: cloudflare/cloudflared:latest
    network_mode: service:nginx
    command: tunnel --no-autoupdate run
    env_file: .env
    restart: always
    depends_on:
      - nginx
```

The configuration is simple.  We're using the official images of Nginx and Cloudflare, and we're pointing them to read a `.env` file for environment variable configurations.

Let's check how each of these two services are configured.

## Nginx

To enhance the security of our Ollama API, we will configure Nginx to validate incoming requests for Bearer Authorization tokens.

By providing a [default.conf.template](https://github.com/eleiton/ollama-tunnel/blob/main/templates/default.conf.template) file as a starting point, we can add a custom configuration that validates the presence and authenticity of Bearer tokens in each request. 

You can see this in action here:

```shell
if ($http_authorization != "Bearer ${OLLAMA_API_KEY}") {
    return 401;
}
```

Now that we've configured Nginx to authenticate requests, we simply need to set the OLLAMA_API_KEY value securely.

We'll achieve this by defining an environment variable in a `.env` file, which will serve as our secure configuration source. 
By keeping sensitive information like API keys out of our codebase and instead storing it externally, we can maintain better security and scalability.

You can find an example file [here](https://github.com/eleiton/ollama-tunnel/blob/main/example.env).

The value for the OLLAMA_API_KEY can be any string you like. However, to ensure consistency, it is recommended to use a standardized format similar to OpenAI keys. This format consists of the prefix sd_ followed by 32 hexadecimal digits.

For example:
```shell
OLLAMA_API_KEY = "sd_<hexadecimal value>"
```

If you would like to generate one key via code, here's a Python script you can run:

```python
import hashlib

def generate_api_key():
    secret = 'replace_me'
    return f"sd_{hashlib.sha256(secret.encode()).hexdigest()[:32]}"

print(generate_api_key())
```

## Cloudflared

We're going to use Cloudflare Zero Tunnel to access our Nginx service from the internet.

If you're unfamiliar with Cloudflare Tunnel, please read this [announcement](https://blog.cloudflare.com/tunnel-for-everyone/) from Cloudflare.

For the purposes of this blog, we're assuming you already have a domain name, and you've registered it in Cloudflare.
Steps on how you can do this can be found [here](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/).

You should configure a public hostname, which will be used to connect to your Ollama API.
In this example, we're configuring a public hostname called https://llm.example.com that is redirecting to the service http://localhost:80.

![Desktop View](/assets/img/2025-04-03/cloudflare.png){: width="100%" height="auto"}

In our [docker-compose](https://github.com/eleiton/ollama-tunnel/blob/main/docker-compose.yml) file, we configured Cloudflare to run in the same network as Nginx. As a result, when Cloudflare sends a request through the tunnel, it will look for localhost:80. 
Our Nginx service, running on this port, is responsible for handling the request.

Before forwarding the request to its final destination, Nginx checks the OAuth Bearer token associated with the incoming request. If the token is valid, Nginx will forward the request to Ollama API, ensuring that only authenticated requests reach the intended service.

## Running your service

This closes the loop between Cloudflare, Nginx and Ollama API.  Please make sure you have entered both the `OLLAMA_API_KEY` and the `TUNNEL_TOKEN` in the `.env` file, and start up your docker instances.

```shell
podman compose up
```

Now you can give this a try by accessing your public hostname, sending the OLLAMA_API_KEY as your Authorization header.

```shell
curl -i https://llm.example.com \
-H "Authorization: Bearer YOUR_CUSTOM_OLLAMA_API_KEY"
```



Now you can try exploring different tools or services that allow access to LLMs. For example, if you want to run a ChatGPT like app in your Android device, you can run this OpenSource app called [GPTMobile](https://github.com/Taewan-P/gpt_mobile), that can connect to your Ollama instance.


![Desktop View](/assets/img/2025-04-03/gpt-mobile-1.png){: width="300" height="auto" .w-45 .normal}
![Desktop View](/assets/img/2025-04-03/gpt-mobile-2.png){: width="300" height="auto" .w-45 .right}


## References
GitHub Repository: <https://github.com/eleiton/ollama-tunnel>
