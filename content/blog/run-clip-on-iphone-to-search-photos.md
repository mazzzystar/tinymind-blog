---
title: Run CLIP on iPhone to Search Photos
date: 2024-09-23T04:46:26.880Z
---

> Update: I have made Queryable [open-source](https://github.com/mazzzystar/Queryable). This might help you learn how to [export Core ML](https://github.com/mazzzystar/Queryable#core-ml-export) models, as well as how to calculate, store, search, and accelerate queries.

I built an app called Queryable, which integrates the CLIP model on iOS to search the Photos album OFFLINE. It's now [available on App Store](https://apps.apple.com/us/app/queryable/id1661598353?platform=iphone), and I thought it might be helpful for others who are as frustrated with the search function of Photos as I was. So, I wrote this article to introduce it.

[![Queryable website](https://mazzzystar.github.io/images/2022-12-28/Queryable-logo.png "Queryable")](https://queryable.app/)

## CLIP

[CLIP](https://openai.com/blog/clip/)(Contrastive Language-Image Pre-Training) is a model proposed by OpenAI in 2021. CLIP can encode images and text into representations that can be compared in the same space. It serves as the foundation for many text-to-image models (e.g., Stable Diffusion) to calculate the distance between the generated image and the prompt during training.
![[Source from OpenAI](https://openai.com/blog/clip/)](https://imgs.zhubai.love/e2671f2f86e64c43ad42f18b824fa854.png)

What I did was to make CLIP run on my own mobile device: an iPhone. To run on iOS devices in real time, I made a compromise between performance and model size, and finally chose the [**ViT-B-32**](https://github.com/openai/CLIP) model, separating the `Text Encoder` and `Image Encoder`.

In **ViT-B-32**:

> - The `Text Encoder` encodes any text into a **1x512** dimensional vector.
> - The `Image Encoder` encodes any image into a **1x512** dimensional vector.

We can calculate the proximity of a text sentence and an image by finding the cosine similarity between their text vector and image vector. Here's some pseudo-code to illustrate:

```lang-python
import clip

# Load ViT-B-32 CLIP model
model, preprocess = clip.load("ViT-B/32", device=device)

# Calculate image vector & text vector
image_feature = model.encode_image("photo-of-a-dog.png")
text_feature = model.encode_text("rainy night")

# cosine similarity
sim = cosin_similarity(image_feature, text_feature)
```

## Integrate CLIP into iOS

I exported the Text Encoder and Image Encoder to CoreML models using the [coremltools](https://coremltools.readme.io/docs/pytorch-conversion) library. The final models have a total file size of 300MB. Then, I started writing in Swift.

Here's how to perform inference with the Text Encoder in Swift:

```lang-swift
// Load the Text Encoder model.
let text_encoder = try MLModel(contentsOf: TextEncoderURL, configuration: config)
// Given a prompt, calculate the CLIP text vector for it.
let text_feature = text_encoder.encode("a dog")
```

I split the Text Encoder and Image Encoder into two models because, when using this Photos search app, **your input text will always change, but the content of the Photos library remains fixed.** This means that all the image vectors can be computed once and saved in advance. Then, the text vector is computed only once for each of your searches.

This approach makes real-time text searching across tens of thousands of photos in your library possible. Below is a flowchart of how Queryable works:

![How does Queryable works](https://mazzzystar.github.io/images/2022-12-28/Queryable-flow-chart.jpg)

## Performance

Compared to the search function of the iPhone Photos app, how much does the CLIP-based album search capability improve? The answer is: it's overwhelmingly better. With CLIP, you can search for a scene in your mind, a tone, an object, or even an emotion conveyed by the image.

![Search for a scene, an object, a tone or the meaning related to the photo with Queryable](https://mazzzystar.github.io/images/2022-12-28/Queryable-search-result.jpg)

To use Queryable, you first need to build the index, which involves traversing your album, calculating all the image vectors, and storing them. This happens only ONCE, and the total time required depends on the number of photos you have. The speed is about 2000 photos per minute on an iPhone 12 mini. When you have new photos, you can manually update the index, which is very fast.

The time cost for a search also depends on your number of photos. For fewer than 10,000 photos, it takes less than 1 second. For me, an iPhone 12 mini user with 35,000 photos, each search takes about 2.8 seconds.


#### 1.On Privacy and security issues.

Queryable is designed as an OFFLINE app that does **not require a network connection** and will **NEVER** request network access, thereby avoiding privacy issues.

#### 2.What if my photos are stored in iCloud?

Due to the inability to connect to a network, Queryable can only use the cache of the low-definition version of your local Photos album. However, the CLIP model itself resizes the input image to a very small size (e.g., ViT-B-32 uses 224x224), so if your image is stored in iCloud, it actually **does not affect search accuracy**. The only limitation is that **you cannot view its original photos** in search results.

\- **Update**: In the latest version, you have the option to grant the app network access to download photos stored in iCloud. This will **only** occur when the photo is included in your search results, the original version is stored in iCloud, and you have navigated to the details page and clicked the download icon. Once you grant the permissions, you can close the app, reopen it, and the photos will be automatically downloaded from iCloud.

#### 3. Any requirements for the device?

- iOS 16.0 or above
- iPhone 11 (**A13 chip**) or later models

#### 4.Have some suggestions or product experience issues?

Feel free to contact me by email: myfancoo@gmail dot com.
