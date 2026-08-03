---
layout: doc
title: Building a Vision-Language-Action Model from Scratch 
description: >
  This chapter covers the basics of content creation with Hydejack.
video: https://www.youtube.com/watch?v=ueSmN_VxNPg&list=PLlqdnFs9xNwql5KET7v7zyl393y10qxtw&index=4
hide_description: true
---
This chapter covers how to build a VLA model for robotics from scratch.


# Building a Vision-Language-Action Model from Scratch

**Understand VLAs and how to build one yourself**

VLAs have fascinated me ever since I first encountered them in a Reinforcement Learning course. That curiosity led me to dig into papers like RT-2, OpenVLA, π₀ (and its follow-ups), GR00T, and many others throughout the year.

A clear architectural pattern appears across all of them.

---

## Why VLAs Matter Right Now

VLAs deliver generalization. Before RT-2, robots relied on what Google DeepMind called “complex stacks of systems” engaged in an “imperfect game of telephone.” VLAs collapse that fragmentation: a single model can perceive, reason, and act.

Multimodality is what bridges reasoning to control. By treating vision, language, and action as tokens in one unified sequence, VLAs solve the grounding problem. Visual encoders turn camera images into patch embeddings, projection layers map those embeddings into the language model’s space, and the LLM then generates action tokens from the fused representation. A minimal version of this same idea is what I call mini-VLA.

---

## The Generalist Robot Idea

Physical Intelligence put it well in October 2024: just as large language models serve as foundation models for language, generalist robot policies will become foundation models for physical intelligence.

That vision is what pulled me into the space—“ChatGPT for the physical world!”—only to discover it is far harder than it sounds.

Still, the foundations are solid. The Open X-Embodiment dataset (October 2023) gave the community more than a million real-robot trajectories across 22 embodiments—the ImageNet of robot learning. OpenVLA (June 2024) showed that a 7 B model could outperform the 55 B RT-2-X. And π₀ (October 2024) became the first system to successfully fold laundry, make coffee, assemble boxes, and bus tables—tasks no earlier robot learning system had mastered.

---

## How Do You Build Your Own VLA?

The pipeline is conceptually simple:

**Vision → Language → Action**

It consists of four main pieces:

1. A **vision encoder** that turns RGB observations into patch embeddings  
2. A **vision-language projector** that maps those embeddings into the LLM’s representation space  
3. A **language-model backbone** that processes the fused tokens together with text instructions  
4. An **action head** that converts the model’s outputs into executable robot commands  

![VLA pipeline overview](https://substack-post-media.s3.amazonaws.com/public/images/18754e1d-1ffc-4a01-8b44-6192b95782fa_1152x648.gif)

## mini-VLA: A Lightweight Version

I deliberately simplified every component so the whole system would run quickly on an ordinary laptop with no GPU. Below is the full pipeline: data collection, model design, training choices, and inference in simulation.

### Collecting Data

Robots do not come with childhoods full of trial-and-error, so we have to create that experience for them.

Each transition in the dataset contains:

- a raw RGB image (robot hand, object, goal)  
- a state vector (positions, velocities, gripper status, etc.)  
- the action taken by an expert  
- a short textual instruction (“push the object to the goal”)

In Meta-World we can rely on an expert policy instead of tele-operation. The guiding principle is classic behavior cloning:

> Imitate the expert until you become the expert.

### Designing the Model

The policy answers a simple question:

> Given what I see (image), what I know (state), and what I am told (instruction), which action should I take next?

The high-level flow is:

1. Image → image encoder → image token  
2. Text → text encoder → language token  
3. State → state encoder → state token  
4. Fusion → combine the three tokens → fused context  
5. Diffusion head → sample an action conditioned on that context  

#### Seeing — Image Encoder

On a tight compute budget we skip large ResNets or Vision Transformers and use a tiny CNN. It compresses the full image into a compact “gist” that still roughly locates the robot hand and the object.

![Image encoder](https://substack-post-media.s3.amazonaws.com/public/images/dd2e6d4e-d03f-4eb3-9c9d-416dc58919b3_1152x648.gif)

#### Reading — Text Encoder

We keep only the final hidden state of a lightweight text encoder. That single 128-dimensional vector summarizes the instruction (“push the object to the goal”). No heavy transformers—clarity and minimalism come first.

![push the object to the goal](https://substack-post-media.s3.amazonaws.com/public/images/a570906b-62dc-45a8-9e47-7845919f4c13_320x320.gif)

#### Feeling — State Encoder

The simulator already supplies a state vector. A small MLP with LayerNorm turns it into a state token. If the image is “what the world looks like,” the state is “what the robot’s body is feeling and knowing.”

#### Fusion

Fusion creates one unified representation of what the model sees, what it is told, and what the robot knows. Think of it as a short meeting where vision, language, and state each give a quick report and an MLP writes the summary. You could replace this step with attention or a transformer; the rest of the pipeline only cares that it receives a single `(B, d_model)` token.

![Fusion](https://substack-post-media.s3.amazonaws.com/public/images/54239dfc-a04d-4d97-890d-1e4736a91b46_480x276.gif)

#### Acting via Diffusion

This is the policy head. During training we add noise to the expert action; the model learns to reverse that noise, guided by the fused image + text + state context. At inference we start from pure noise and iteratively denoise until we obtain a plausible action. It is a standard DDPM-style process in action space: the forward diffusion follows  

\[ x_t = \sqrt{\bar{\alpha}_t}\,x_0 + \sqrt{1-\bar{\alpha}_t}\,\varepsilon \]  

and the denoiser is a simple MLP conditioned on timestep and the fused context.

![Diffusion / action head](https://substack-post-media.s3.amazonaws.com/public/images/96e73e08-242f-4e5c-b208-a0a35f378428_1152x648.gif)

![Diffusion process](https://substack-post-media.s3.amazonaws.com/public/images/1b365031-ce63-4ddf-80a6-cc9a48e3b9bd_1152x648.gif)

### Design Philosophy

The guiding principles were educational clarity and expandability rather than peak performance:

- **Modularity** — encoders, fusion, and diffusion head are independent modules, so any of them can be swapped without touching the rest of the code.  
- **Small but expressive** — the model must still run on a low-end CPU.  
- **Environment-agnostic** — the same code should work beyond Meta-World.  
- **Distributional actions** — instead of predicting a single action we sample from a learned distribution, which tends to produce smoother, more robust behavior.

## Lessons from Building mini-VLA

Before this project, VLAs felt like mysterious “everything machines.” Once I assembled each piece myself, I saw that a VLA is less a monolithic model and more a negotiated agreement among modalities.

(The detailed technical lessons will appear in a follow-up post.)

## Closing Thoughts

This tiny pipeline is more than a toy. Scale it—better encoders, richer language, more diverse tasks, real robot sensors—and you start walking toward robots that can learn new behaviors from a handful of demonstrations or even from high-level natural language. All of that is possible with only modestly better hardware.

The single most important takeaway is this:

> Real robot intelligence will not be summoned. It will be engineered—carefully, incrementally, and transparently—by people willing to build, break, and rebuild systems like this one.

If you have read this far, you are already one of those people. I hope this project encourages you to build something small that eventually does something big.