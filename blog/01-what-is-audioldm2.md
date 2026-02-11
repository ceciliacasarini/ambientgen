---
layout: default
title: "Day 1: What is AudioLDM2 and why I'm using it"
date: 2025-02-11
---

# Day 1: What is AudioLDM2 and Why I'm Using It

*Building AmbientGen — a text-to-ambient-sound generator. This is post #1.*

## What I'm building and why

I want to type "rain on a tin roof with distant thunder and crickets" and get back an audio file that sounds like exactly that. Not a recording pulled from a library — a *generated* soundscape that never existed before.

This is a real product category now. Apps like myNoise and Noisli have millions of users who want ambient soundscapes for focus, sleep, or relaxation. But they all rely on pre-recorded loops. What if you could describe *any* atmosphere and have it created on the fly?

To build this, I need a model that takes text and produces audio. I chose AudioLDM2. This post is about what it is, how it works, and why I picked it over the alternatives.

## The problem: text-to-audio is harder than you think

If you've been following AI image generation — Stable Diffusion, DALL-E, Midjourney — you might think: "surely the same approach works for audio?" And broadly, yes. But there are some important differences.

Images are spatial. Audio is temporal. A 10-second clip at 16kHz is 160,000 samples — that's a lot of raw data to generate. And unlike images where "a dog sitting on a couch" has a fairly obvious visual interpretation, the connection between language and sound is often less direct. What does "cozy" *sound* like? What does "distant" mean in audio terms?

There's also a data problem. We have billions of image-text pairs scraped from the internet (every image has an alt tag or caption). Audio-text pairs are much scarcer. Most audio datasets have simple labels like "dog barking" rather than rich descriptions like "a small dog barking excitedly in a large echoey hallway."

So the challenge is: generate high-dimensional temporal data from text, with relatively limited paired training data. AudioLDM2's clever trick is finding a way around this data scarcity.

## Latent diffusion: the core idea

If you understand Stable Diffusion for images, you're 70% of the way to understanding AudioLDM2. The key insight of latent diffusion is: *don't generate in pixel/sample space — generate in a compressed latent space*.

Here's the pipeline:

1. **Compress**: A VAE (variational autoencoder) learns to compress audio (specifically, mel spectrograms) into a much smaller latent representation. Think of it as a zip file for audio features.
2. **Generate**: A diffusion model (U-Net) works in this compressed space. It starts from random noise and iteratively denoises it, guided by your text prompt. This is *much* cheaper than working with raw audio.
3. **Decompress**: The VAE decoder converts the generated latent representation back into a mel spectrogram, which gets converted to a waveform via a vocoder (HiFi-GAN).

```
Text → [Text Encoders] → [Diffusion U-Net in latent space] → [VAE Decoder] → [Vocoder] → Audio
```

The diffusion process itself is the same as in image generation: gradually remove noise, step by step, with the text conditioning telling the model *what* to generate at each step. If you want to understand diffusion models deeply, the DDPM paper is the reference, but for now the intuition is: the model learns to reverse a noising process, and at generation time you give it noise and let it "imagine" its way to clean audio.

## What makes AudioLDM2 different: the Language of Audio

Here's where it gets interesting. AudioLDM (v1) used CLAP embeddings to condition the diffusion model. CLAP — which I'll explain in a moment — basically gives you a way to say "this text and this audio mean the same thing." That worked, but it was limited.

AudioLDM2 introduces a concept called **"Language of Audio" (LOA)** — a shared intermediate representation that any type of audio can be mapped to. The LOA is based on AudioMAE (Audio Masked Autoencoder), a self-supervised model trained on large amounts of unlabeled audio.

Why does this matter? Because AudioMAE learns from *audio alone* — no text labels needed. This means:

- The VAE and AudioMAE can be pre-trained on huge amounts of unlabeled audio (self-supervised)
- Only the GPT-2 translation module needs paired text-audio data
- The same framework handles speech, music, AND sound effects

The generation process works in two stages:

**Stage 1**: Text → GPT-2 → LOA features (translating from language to the "language of audio")

**Stage 2**: LOA features → Latent Diffusion Model → Audio (generating audio conditioned on LOA)

This two-stage approach is the key innovation. By having an intermediate audio representation (LOA), the model separates "understanding what to generate" from "actually generating it." And because the LOA is learned from raw audio, the generation stage can be trained self-supervised.

## CLAP: the bridge between words and sounds

I mentioned CLAP above. Let me explain it properly because it's a foundational piece.

CLAP stands for **Contrastive Language-Audio Pretraining**. If you know CLIP (for images), CLAP is the audio equivalent. The idea is simple and powerful:

1. Take an audio encoder (e.g., HTS-AT, a CNN-based model) and a text encoder (e.g., BERT)
2. Train them together so that matching audio-text pairs end up close in a shared embedding space, while non-matching pairs are pushed apart
3. Now you have a way to measure similarity between *any* text and *any* audio

In AudioLDM (v1), CLAP embeddings were the main way to condition the diffusion model — the text got encoded by CLAP's text encoder, and this embedding guided the audio generation.

In AudioLDM2, CLAP is still part of the picture but it's joined by FLAN-T5 (a large language model) as an additional text encoder. Both feed into the GPT-2 module that produces LOA features. The multiple text encoders give the model a richer understanding of the text prompt — CLAP provides audio-semantic alignment while FLAN-T5 provides deeper language understanding.

## Why AudioLDM2 and not something else?

I considered several alternatives:

**AudioGen (Meta)**: Autoregressive model — generates audio token by token, like GPT generates text. Good for sound effects but slower at inference and less flexible for long-form ambient generation.

**Stable Audio Open (Stability AI)**: Very capable, especially for music. Uses a DiT (diffusion transformer) architecture with timing conditioning. It's newer and arguably higher quality, but more oriented toward music production than ambient sound generation.

**Make-An-Audio 2**: Interesting approach with temporal enhancement, but less community support and harder to get running.

I picked AudioLDM2 because:

1. **It runs in 5 lines of code** via HuggingFace Diffusers. The barrier to getting started is near zero.
2. **It handles sound effects well**. It was explicitly trained on general audio, not just music.
3. **Active community and documentation**. When you're learning, this matters more than raw performance.
4. **It's a great teaching model**. The architecture touches latent diffusion, contrastive learning, autoencoders, self-supervised learning, and autoregressive models — basically a tour of modern generative AI.

I might benchmark it against Stable Audio later. But for getting started and learning the fundamentals, AudioLDM2 is the right choice.

## What I'll do next

In the next post, I'll set up the environment, run AudioLDM2 for the first time, and generate my first ambient sounds. I'll try different prompts and start building intuition for what works and what doesn't.

The goal: go from paper to sound in one session.

---

**Paper:** [AudioLDM 2: Learning Holistic Audio Generation with Self-supervised Pretraining](https://arxiv.org/abs/2308.05734) (Liu et al., 2023)

**Also referenced:**
- [CLAP: Learning Audio Concepts from Natural Language Supervision](https://arxiv.org/abs/2206.04769) (Elizalde et al., 2022)
- [Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239) (Ho et al., 2020)
- [Latent Diffusion Models (Stable Diffusion)](https://arxiv.org/abs/2112.10752) (Rombach et al., 2022)

*[← Back to blog index](index.md)*
