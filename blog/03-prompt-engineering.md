---
layout: default
title: "Day 3: Prompt Engineering for Audio — What Actually Works"
date: 2025-02-12
---

# Day 3: Prompt Engineering for Audio — What Actually Works

*Building AmbientGen — a text-to-ambient-sound generator. This is post #3.*

## Before we start: I was using the wrong settings

A quick correction from Day 2. I initially reported that AudioLDM2's output quality was poor — lots of artifacts, distortion, disappointing results. Turns out the problem was largely me, not the model.

I was using 50 inference steps and a guidance scale of 7.0. The settings that actually work well:

- **Inference steps: 40** (not 50, not 200 — 40 was the sweet spot on a T4 GPU)
- **Guidance scale: 3.0** (much lower than what I'd assumed from image diffusion experience)
- **Audio length: 8 seconds** (more stable than 10)
- **`num_waveforms_per_prompt`**: didn't make a practical difference on my setup

Lesson learned: read the documentation before concluding a model doesn't work. The guidance scale was the biggest culprit — at 7.0 the model was essentially "trying too hard" and producing overcooked, distorted audio. At 3.0, things smoothed out considerably.

With that fixed, I ran a systematic set of experiments to understand how prompts affect audio generation quality.

## Experiment 1: How much detail does the model need?

I tested three sound categories — water, wind, and fire — with prompts ranging from one word to a full descriptive sentence.

**Water:**

| Prompt | Result |
|--------|--------|
| "water" | Vague, ambiguous — could be a faucet, rain, anything |
| "water flowing" | Slightly better, but still generic |
| "a stream of water flowing over rocks" | Clear improvement — spatial context helps |
| "a clear mountain stream flowing over smooth rocks in a quiet forest" | Best result. The forest context added a convincing background |

**Wind:**

| Prompt | Result |
|--------|--------|
| "wind" | Generic white noise |
| "wind blowing" | Still generic |
| "wind blowing through trees" | Noticeable texture, more natural |
| "gentle wind blowing through pine trees on a mountain, leaves rustling" | Best, but not dramatically better than the 3-word version |

**Fire:**

| Prompt | Result |
|--------|--------|
| "fire" | Surprisingly decent even with one word |
| "fire crackling" | Good — "crackling" is a very specific acoustic descriptor |
| "a campfire crackling at night" | Good, similar to above |
| "a warm campfire crackling and popping at night with crickets in the background" | Best — the crickets added a nice high-frequency complement |

**Key finding:** There's a sweet spot of specificity. The biggest quality jump happens when you add *spatial context* (where the sound happens) and *acoustic descriptors* (crackling, flowing, rustling). Going beyond 3-4 meaningful elements doesn't improve things much.

Fire was interesting because it worked well even with short prompts. My theory: "crackling" maps to a very specific spectral pattern in the training data, while "water" is hugely ambiguous.

## Experiment 2: Magic words — quality modifiers

I took a mediocre-sounding prompt ("rain falling on a window") and prepended different quality modifiers to see which ones improve the output.

Results, roughly ranked:

1. **"ambient soundscape of"** — best for our use case. The word "soundscape" seems to steer the model toward the right region of its training distribution
2. **"professional field recording of"** — clean, naturalistic results. "Field recording" is associated with high-quality nature recordings in training data
3. **"high quality recording of"** — generic but effective improvement over no modifier
4. **"ASMR recording of"** — interesting results: very intimate, close-mic'd feel. Could be useful for certain types of ambient sounds, but too "whispery" for general use
5. **"clear and detailed sound of"** — minimal improvement. Vague adjectives don't help much
6. **No modifier** — baseline, weakest results

Why do these work? The model learned from audio datasets that have metadata and descriptions. Audio tagged as "field recording" in the training data is likely high-quality nature recordings from sites like Freesound. "ASMR" content has a very specific sonic signature. The modifier doesn't change what the model generates — it changes which part of the training distribution it draws from.

This is the same principle as in image generation, where "professional photograph" or "8K" produces better results than just describing the subject. You're not just describing the content, you're describing the *recording quality and context*.

## Experiment 3: How many elements can you combine?

Ambient soundscapes are usually layered — rain plus thunder plus wind. How well does AudioLDM2 handle multi-element scenes?

| Elements | Prompt | Result |
|----------|--------|--------|
| 1 | "rain falling" | Clean but flat, one-dimensional |
| 2 | "rain falling with distant thunder" | Good improvement, the thunder adds dynamics |
| 3 | "rain falling with distant thunder and wind blowing" | Still OK, but volume balance between elements starts to feel off |
| 4 | "rain falling on a cabin roof with distant thunder, gentle wind, and a fireplace crackling inside" | The model struggled here. Indoor/outdoor mixing was inconsistent — it couldn't clearly separate the spatial logic of inside vs outside |

**Finding:** 2-3 elements is the sweet spot for a single generation. Beyond that, the model tends to either drop elements or blend them unnaturally.

This has a direct implication for AmbientGen: instead of trying to generate complex scenes in a single pass, we should **generate individual layers and mix them**. Generate rain, generate thunder, generate fireplace — then composite them with proper volume control. This will likely produce better results and gives the user more control.

## Experiment 4: Seed lottery

I generated the same prompt ("ocean waves on a sandy beach with seagulls") with five different seeds. The variation was significant — out of 5 generations, 2 were clearly good, 2 were mediocre, and 1 was poor.

This isn't random noise in the perceptual sense. Different seeds produce different starting points in the latent space, and some starting points lead to better denoising paths than others. For the app, this means we should either:

1. Generate multiple seeds and let the user pick, or
2. Auto-select the best one using CLAP similarity scoring (measure how well the audio matches the prompt)

## The prompt formula

Based on all experiments, here's the pattern that consistently produces the best ambient sound results:

```
[quality modifier] + [specific sound] + [acoustic descriptor] + [spatial context] + [1-2 complementary elements]
```

Examples:
- "ambient soundscape of a gentle stream flowing over rocks in a forest with birdsong"
- "field recording of ocean waves crashing on a rocky shore with distant seagulls"
- "high quality recording of a campfire crackling with crickets at night"

And always use:
- **Negative prompt:** "Low quality."
- **Guidance scale:** 3.0
- **Inference steps:** 40

## What's next

The composition limit (2-3 elements per generation) actually shapes the architecture of our app. Instead of one big "generate everything" button, AmbientGen should let you build scenes in layers. In the next post, I'll build the Gradio interface with this layered approach — generate individual ambient elements and mix them together.

---

**Experiment notebook:** Available in the [experiments folder](../experiments/).

*[← Back to blog index](index.md)*
