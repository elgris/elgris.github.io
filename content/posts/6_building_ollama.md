+++
title = 'Exploring opensource projects by building (Ollama)'
date = 2024-12-25T16:32:09+01:00
author = 'elgris'
categories = ["tech"]
+++

I haven't touched opensource code for quite a while, and that made me sad. There is lots of biz, but not much fun in the corpo world. One evening I got me a beer and decided to get some fun! And since LLMs are on hype, why not try running a local one? That's how I came first to [llama.cpp](https://github.com/ggerganov/llama.cpp), then to [Ollama](https://github.com/ollama/ollama). And I asked myself: "What is the best way to start exploring Ollama's internals?" The answer is: try building it yourself. So here we go!
## 1. Hard: build it as-is
My Docker skills are rather rusty, so I started with the "old school way" first and tried to build everything on my machine. Which is an Asus G14 laptop with **AMD RX 6700s GPU** running **Fedora 41**. Here's what I learned:
1. No Nvidia CUDA for me, I need to go with AMD [ROCm](https://rocm.docs.amd.com/en/latest/) instead., and it requires the following deps:
```bash
$ sudo dnf install git make gcc go \ # core deps
  hipcc rocm-hip-devel hipblas-devel rocblas-devel #rocm-related deps
```
   Finding all these deps required me to dig quite a few sources. The official [Ollama doc for ROCm]([https://github.com/ollama/ollama/blob/main/docs/development.md#linux-rocm-amd](https://github.com/ollama/ollama/blob/main/docs/development.md#linux-rocm-amd)) was the obvious entry point, the rest - Reddit posts, Gists, Github issues and Medium articles. It's great that people discuss and share such things in the searchable space! Unfortunately, Discord isn't a part of it =\.
2. Assuming that you have checked out the repo already (`$ git clone https://github.com/ollama/ollama.git`), let's build Ollama runners:
```bash
$ HSA_OVERRIDE_GFX_VERSION=10.3.0 make -j 12 dist
```
   Where `HSA_OVERRIDE_GFX_VERSION` allows the build script to detect my GPU (`gfx1032`) and build a `rocm` runner that Ollama will use for running models. The whole process takes ~30 minutes on the laptop, and produces  `cpu_avx`, `cpu_avx2` and `rocm_avx` runners in `dist/linux-amd64/lib/ollama/runners/` along with a bunch of shared libraries.
   A typical problem here is a broken/weird rocm installation on the host, so that the build script is unable to detect it and refuses to build the `rocm` runner. If you get blocked by it, try pulling the most recent Ollama repo state or build in an isolated clean environment, e.g. in an interactive container that you can start, e.g., like this: 
```bash
$ docker run -it \
  --device=/dev/kfd \
  --device=/dev/dri \
  --group-add=video \
  fedora:41 bash
```
3. The final touch, let's build the Ollama binary itself:
```bash
go build -trimpath -o dist/linux-amd64-docker/ollama
```

4. Let's run the thing! Just make sure you feed it all produced libraries. (assuming that the ollama repo is checked out into `$HOME`):
```bash
$ HSA_OVERRIDE_GFX_VERSION=10.3.0 LD_LIBRARY_PATH=$HOME/ollama/dist/linux-amd64/lib/ollama/ $HOME/ollama/dist/linux-amd64/ollama serve
```
   Check the output, you should see something like this:
```
level=INFO source=types.go:131 msg="inference compute" id=0 library=rocm variant="" compute=gfx1032 driver=0.0 name=1002:73ef total="8.0 GiB" available="8.0 GiB"
```
  This piece of log tells us that the GPU was recognized correctly and should be used for loading a model onto it. Let's run a model!
```bash
$ $HOME/ollama/dist/linux-amd64/ollama run gemma2:9b-instruct-q4_K_M
```
   The expected output is something like this:
```
level=INFO source=sched.go:714 msg="new model will fit in available VRAM in single GPU, loading" model=/home/elgris/.ollama/models/blobs/sha256-cb654129f57b4f44e372fab3d6540c0d6784c3adc33125d7c38ab47122ee2a54 gpu=0 parallel=1 available=8556523520 required="7.1 GiB"
level=INFO source=server.go:376 msg="starting llama server" cmd="/home/elgris/_workspace/ollama/dist/linux-amd64/lib/ollama/runners/rocm_avx/ollama_llama_server runner --model /home/elgris/.ollama/models/blobs/sha256-cb654129f57b4f44e372fab3d6540c0d6784c3adc33125d7c38ab47122ee2a54 --ctx-size 2048 --batch-size 512 --n-gpu-layers 43 --threads 8 --parallel 1 --port 42305"
```
This way is the hardest, yet very flexibile. You can play with different versions of dependencies, try the newest bleeding edge. You can wreck code havoc and build your own Frankenstein! Or you can find deficiencies in the build script on a specific host, fix it and contribute back!
Although, I'd use it if I were an Ollama developer who works on it every day and needs to test and debug it all the time. Otherwise - the setup is rather brittle. I won't be surprised, if this instruction gets obsolete soon.
## 2. Medium: build with a builder container
The "Hard" way looks too low-level. What if we had a script that encapsulates the whole process and makes rebuilding reliable and repeatable? Hmm... Containers to the rescue! We already touched on them in a very rocm-specific Fedora-specific way. However, the Ollama community provides universal [Docker image](https://github.com/ollama/ollama/blob/main/Dockerfile) for building Ollama and all its runners.
1. Let's build a builder image for my system (assuming, you're inside of `ollama/` directory):
```bash
$ docker build -t builder-amd64 \
  --platform linux/amd64 \
  --build-arg GOLANG_VERSION=1.23.4 \
  --build-arg ROCM_VERSION=6.2.4 \
  --target unified-builder-amd64 \
  -f Dockerfile .
```
2. Then run it and build `ollama` binary with its dependency libraries inside of it (takes quite some time):
```bash
$ docker run --platform linux/amd64 --rm -it -v \
  $(pwd):/go/src/github.com/ollama/ollama/ builder-amd64
$ make -j 12 dist && go build -trimpath -o dist/linux-amd64/ollama .
```
3. Finally, run the produced `ollama` binary on your machine, the same way as before::
```bash
HSA_OVERRIDE_GFX_VERSION=10.3.0 LD_LIBRARY_PATH=$HOME/ollama/dist/linux-amd64/lib/ollama/ $HOME/ollama/dist/linux-amd64/ollama serve
```
This way strikes the perfect balance IMO: it takes away some of dependency management yet leaves you a few ways for customization. Additionally, it makes you learn some Docker. 
## 3. Easy: use pre-built containers
Ollama community also provides pre-built Docker images with ROCm runners (and [docs](https://github.com/ollama/ollama/blob/main/docs/docker.md) ), so you can run it as easy as:
```
$ docker run -d \
  --device /dev/kfd \
  --device /dev/dri \
  --env HSA_OVERRIDE_GFX_VERSION="10.3.0"
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama \
  ollama/ollama:rocm
```
This approach takes away the whole mess of building, dependencies, etc. Suitable for getting something up'n'running quickly, but wouldn't work as good for debugging.
We can still get more fun from it and pair an Ollama container with [Open WebUI](https://github.com/open-webui/open-webui), which also comes as a Docker image. You can learn some of Docker Compose for that. My setup looks like [this](https://github.com/elgris/webui-ollama-docker/blob/main/compose.yaml), although Open WebUI repo provides a ready-to-use [compose config](https://github.com/open-webui/open-webui/blob/main/run-compose.sh).
## Outro
Yep, you can make it much easier and just run Ollama or/and Open WebUI installation scripts and they will do everything for you. But where is fun then? :) Going through the whole build process and getting your hands dirty with a bunch of stuff:
* If you're new to your OS - you'll learn its packaging system, get burned by (un)supported versions of the packages you need and overall improve your understanding of the OS.
* You'll learn some of build tools, e.g. `make`, `ninja` , and their configuration.
* You can try containers. Start with Docker, eventually learn that there is also Podman that comes out of the box with Fedora and there are many more other alternatives. Binding containers together is also a nice exercise.
While trying all that, getting frustrated, looking for answers on the Internet, talking to people on Discord, Github and Reddit, you'll expand your scope of knowledge and eventually get some ideas on how to improve something in those tools and maybe contribute back to the community. And that is a good start in the opensource world! Take it from here and move on! 

P.S. I realized that Open WebUI is a large project that I have little knowledge about, so I'd likely spend some time poking at it. Let's see how it goes!