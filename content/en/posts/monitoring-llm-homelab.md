---
title: "The Dashboard That Almost Worked: Monitoring llama.cpp with Prometheus and Grafana"
subtitle: "Observability on a struggling homelab — between bugs, metrics, and a model that never wants to retire"
date: 2026-06-03
lastmod: 2026-06-03
draft: false
author: "Ciro Lucio Tecce"
authorLink: "https://ciroluciotecce.it"
description: "How I tried to set up a monitoring system for llama.cpp with Prometheus and Grafana — and what I learned when a llama.cpp bug brought it all down."
license: ""
images: []

tags: ["llm", "homelab", "monitoring", "llama.cpp", "grafana", "prometheus"]
categories: ["Homelab", "LLM", "DevOps"]

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
---

## Introduction — the mini PC that won't give up (and the problem of watching it)

For the past few months, one of the mini PCs in my homelab has been getting straight-up abused. Every day I make it do things it was never designed for: load language models, process prompts, generate text. This is discrete GPU territory, but it doesn't have one.

And yet, between a 3D-printed expansion card and a docker-compose held together with duct tape, this little Ryzen 5 4650G Pro with 32GB of RAM and its integrated GPU manages to get by. It's no speed demon — to give you an idea, with Qwen3.6 35B A3B in 4-bit quantization it does about 70-90 tokens per second on prefill and roughly 12 tokens per second on generation. Nothing to write home about.

With a lightweight coding agent like [pi](https://github.com/EarendilWorks/pi) and the magic of KV caching, small tasks are manageable. For batch workflows — like the ones I've set up with n8n to leverage the model's multimodal capabilities — no problem at all.

But there's a problem that shows up sooner or later when you're dealing with experimental stuff: **without data, you're blind**.

> Observability and data are the key to making informed decisions. Anything not backed by a number, a chart, or concrete evidence is just an impression. And as such, it can't drive a technical decision-making process.

This post is the story of how I tried to make visible what happens inside llama.cpp. A story made of metrics, dashboards, Prometheus, Grafana — and a bug that, at least for now, forced me to hit pause.

---

## Background — from llama-swap to llama.cpp (and the lost visibility)

Until recently, I used [llama-swap](https://github.com/mut-ex/llama-swap) to manage my models. For those unfamiliar, llama-swap is an abstraction layer on top of llama.cpp that lets you configure multiple models and automatically handles loading and unloading based on incoming requests. All of this long before llama.cpp implemented its native router mode.

llama-swap had one major strength: a web interface that showed the performance of every single request — tokens processed, prompt processing speed, generation speed, average times by percentile. Everything was visible, out of the box.

But over time I started hitting its limits. First: versioning. I could never figure out exactly which version of llama.cpp was bundled in llama-swap's Docker images. Sure, I could have built the image myself, but there was another, more substantial limitation: **parallel requests on different models**. Due to an implementation constraint, llama-swap can't emulate llama.cpp's native ability to handle concurrent requests across different models.

In my current setup, this isn't really a limitation — with my compute capacity, I can barely saturate a single model. But I have new hardware on the way, and I wanted to prepare for a scale-up.

So I switched to llama.cpp directly.

And immediately I missed the monitoring that llama-swap provided. All that visibility, gone in one shot.

---

## Practical case — Prometheus, Grafana, and an unexpected bug

### The search for lost visibility

So I got to work. The goal was simple: replicate what I had with llama-swap, but with standard tools. Prometheus for collection, Grafana for visualization. Everything I later documented in [a GitHub gist](https://gist.github.com/4ae126689db857cc8c2c26f1425c2692.git).

The final dashboard — at least in my intentions — was supposed to show:

- Prefill and generation speed trends per active model, with a per-model filter
- Tokens processed and cumulative processing time
- Currently active concurrent requests
- Models currently loaded in memory

{{< image src="/grafana-llm-dashboard.webp" caption="The Grafana dashboard — when it worked, it was beautiful" >}}

### The scraper problem

The first difficulty came right away. llama.cpp exposes a `/metrics` endpoint, but it needs to be called filtered by model. This means you can't just point a vanilla Prometheus at that endpoint and expect it to work. You need an intermediary layer — a dedicated scraper that handles the per-model filter logic.

Nothing insurmountable, but it was the first sign that the road wouldn't be all downhill.

### The bug that stopped everything

Unfortunately, after setting up the entire infrastructure — Prometheus collecting, Grafana visualizing, the dashboard finally coming to life — I hit a wall.

**A known bug in llama.cpp** related to metrics exposure prevents loaded models from being unloaded.

In plain English: every call to the `/metrics` endpoint resets the model's keep-alive timer. llama.cpp thinks the model is still in use, so it never unloads it when it could go idle.

The consequences are twofold, both painful:

1. **resource waste** — the model stays in memory even when nobody is using it
2. **routing breaks** — if the loaded model is perceived as in use, llama.cpp won't unload it to make room for a different model when a new request comes in

{{< admonition type=warning title="Update 06/03/2026" >}}
As of this writing, the bug is still open and I'm monitoring developments on the llama.cpp repository. For now I've disabled the Prometheus scraper and gone back to manual, sporadic monitoring — the classic furtive glance at `htop` while a model is running.
{{< /admonition >}}

So for now, the solution is on hold. The dashboard is there, the data would be flowing, but I can't use it without breaking the memory management on my little server.

---

## Conclusion — even a defeat is data

At the end of this story, what I'm taking home isn't a working dashboard. It's something else.

I learned that observability isn't optional — it's the foundation on which technical decisions are built. I learned that llama.cpp is a fantastic project, evolving at breakneck speed, but like any software in its growth phase, it has its cracks. And I learned that even a failed experiment produces valuable data — about the technology, about hardware limitations, and about what it really takes to make a system work.

The dashboard will be back. When the bug is fixed, when the new hardware arrives, or when I find a workaround. But in the meantime, this post remains — a snapshot of a moment when I tried to make the invisible visible, and almost succeeded.

> In the end, even a defeat is data. And if there's one thing my homelab has taught me, it's that every problem is an opportunity to understand something more.

The full setup is documented in the [gist](https://gist.github.com/4ae126689db857cc8c2c26f1425c2692.git) — if anyone wants to follow the same path (or help me find a fix for the bug), it's all there.
