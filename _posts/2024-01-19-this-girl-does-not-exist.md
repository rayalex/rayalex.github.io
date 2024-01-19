---
layout: single
classes: wide
title:  "This girl does not exist."
date:   2024-01-19
categories: stable-diffusion generative-ai image-generation
header:
    overlay_image: /assets/images/i_dont_exist.jpg
    overlay_filter: 0.5
---

The girl in the image below (let's call her "Elsa") doesn't exist. But can you tell? If you look close enough, maybe a slightly out of shape fingernail gives it away? Or the slightly blurry earring? But from the distance... she looks pretty real, doesn't she?


![Stable Diffusion](/assets/images/i_dont_exist.jpg)
> A realistic photo of a girl that looks like Elsa, wearing an elegant blue dress. She has long blonde hair and blue eyes. She is smiling and looking at the camera.

## Stable Diffusion

The image above is created using [Stable Diffusion](https://stability.ai/news/stable-diffusion-public-release), a deep-learning text to image model, running locally on my computer and using off-the-shelf GPU.

I'm an amateur photographer and I've always been fascinated by the art of it, and I've been using Dall-E and Midjourney for quite a while now - either to have fun with it, create art for my home, or to generate images for my blog posts. I've also attempted using them to drive content generation in my previous professional role.

I've always been impressed by the results, but I've never really tried to push the limits of what these models can do.

Some time ago I've decided to play around with open-source models based on Stable Diffusion. I honestly didn't expect much of them because I've always thought that the quality of the generated images is directly proportional to the amount of compute you throw at it. I was wrong, very much so. I've been able to generate some pretty impressive images on my desktop computer using only RTX 3080 (sorry nVidia, but no A5000 needed this time 😆) - and in the less amount of time it would actually take Midjourney or DALL-E to do it. And for an effectively free product (not including the cost of electricity 😆) it's amazing what's possible to do today.

_Created using SDXL and ComfyUI._

---

Next, let's see what Midjoyrney and DALL-E can do.

## Midjourney
Midjourney also excels at generating images of people, but it can get difficult to get photorealistic results in case you're asking for lookalikes of animated characters, or people with very distinct features. It's great for imaginary and fantasy content though. In the case of our "Elsa", results are still pretty decent, however they still have some of that "dreamy" look to them.

![Midjourney v6](/assets/images/i_dont_exist_mj6.jpg)

## DALL-E 3
In the case of DALL-E, I've had the far worst results out of all three. It's next to impossible to get a realistic looking person out of it. It's great for generating objects, but not so much for people. I've tried generating a few images of Elsa, but they all look like a weird amalgamation of the character and a human. It's not a bad result, but it's not what I was looking for... also, it's often funny to see what it comes up with given all the content restrictions in ChatGPT-4.

![DALL-E 3](/assets/images/i_dont_exist_dalle.jpg)

## What's next?
I'm still experimenting with Stable Diffusion, refinement, upscaling, LoRAs, etc. in order to refine my workflow for generating photorealistic content. I'm planning to write a few more posts about this and release some models based on it, so stay tuned for that.