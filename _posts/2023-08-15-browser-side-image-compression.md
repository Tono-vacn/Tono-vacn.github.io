---
layout: post
title: Browser Side Image Compression
subtitle: How to compress PNG in user's browser
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/image-compression.webp
# share-img: /assets/img/path.jpg
tags: [Frontend, Browser, Digital Image Processing, JavaScript, Canvas]
author: Yuchen Jiang
---

<!-- # Optimizing PNG Compression and Converting to WebP: A Comprehensive Guide -->

## Introduction

This article is a simple guide for 2 different ways for PNG image compresson: reducing color channels with pixel dithering or converting them to WebP format. Here we will discuss some details about 2 different methods and the way to implement them in a web application.

## 1. Overview of Image Formats

Each image format serves a distinct purpose in the realm of web development. Below is a breakdown of the most common ones, highlighting their strengths and weaknesses.

### JPEG (Joint Photographic Experts Group)
- **Strengths**: Ideal for photos, uses lossy compression for smaller file sizes. Quick to load on web pages.
- **Weaknesses**: Detail loss is inevitable at higher compression levels, leading to artifacts and a reduction in image clarity.

### WebP
- **Strengths**: Provides superior compression for both lossy and lossless formats. WebP images are much smaller than JPEGs or PNGs while maintaining high quality.
- **Weaknesses**: WebP is not fully supported in older browsers, although modern web browsers handle it well.

### PNG (Portable Network Graphics)
- **Strengths**: Lossless compression ensures no loss of quality. Supports transparency, making it perfect for graphics, logos, and other assets requiring sharp edges.
- **Weaknesses**: Large file sizes make PNG less suitable for websites requiring fast load times.

### GIF (Graphics Interchange Format)
- **Strengths**: Best for animations and simple graphics due to its support for transparent backgrounds and looping animations.
- **Weaknesses**: Limited color range and large file sizes for complex images.

<!-- ### BMP (Bitmap)
- **Strengths**: No compression means perfect quality preservation.
- **Weaknesses**: Extremely large files make it impractical for web use. -->

## 2. PNG Compression Techniques

There are various methods for compressing PNG files, each with different levels of effectiveness depending on the context and requirements. Here we will show some common techniques for browser side PNG compression.

### [UPNG.js](https://github.com/photopea/UPNG.js)
- UPNG.js is an open-source PNG encoder/decoder developed by Photopea, which provides simple color reduction (directly down sampling the bit depth). 
- It supports lossless compression, but its lack of pixel dithering can lead to visual artifacts, such as color banding. 
- Despite this, it's a valuable tool for basic compression tasks.

### [Resampling with Canvas](https://github.com/Donaldcwl/browser-image-compression/)
- A more common method involves drawing PNG images onto a canvas element and reducing the resolution, making use of the canvas API to generate smaller file sizes. 
- This approach preserves color richness while reducing the overall resolution to achieve the desired compression.
- This method will cause loss of quality, which makes it less suitable for images with sharp edges or text.

### [WebP Transcoding](https://developers.google.com/speed/webp)
- simply converting PNG images to WebP format can achieve significant file size reductions without compromising quality.
- WebP supports both lossy and lossless compression, making it a versatile choice for web developers.

### [Pixel Dithering](https://en.wikipedia.org/wiki/Dither)
- Pixel dithering is a technique that introduces noise to smooth color gradients and reduce color banding in images.
- By spreading the error in color mapping across adjacent pixels, dithering creates the illusion of a smoother transition between colors.
- This technique is commonly used in image processing to improve visual quality.
- However, dithering can introduce noise and reduce image sharpness, so it may not be suitable for all scenarios.
- Here we will use [image-q](https://github.com/ibezkrovnyi/image-quantization) to quantize the color channels and add pixel dithering to improve image quality.

### [Image-q](https://github.com/ibezkrovnyi/image-quantization)
- Image-q is a pure TypeScript image processing library that supports various dithering algorithms and palette sampling methods.

## 3. TinyPNG vs UPNG.js: Compression Analysis between existing tools

One famous tool for PNG compression for reference is TinyPNG, which is a paid service that offers lossy compression for PNG images. Here we will compare TinyPNG with UPNG.js in terms of compression quality and file size reduction.

TinyPNG uses advanced pixel dithering and color quantization techniques to reduce file size while maintaining image quality, which can be discovered by scrutinizing the edge of color blocks in the image.

The result shows smoother color gradient and less noticeable color banding, compared to UPNG.js.

- **Dithering**: A process where the error in color mapping is spread across adjacent pixels, giving the illusion of a smoother transition between colors.
- **Color Banding**: Occurs when there are insufficient colors to represent smooth gradients, leading to noticeable bands of color.

## 4. PNG to WebP Conversion

WebP offers superior compression rates and retains image quality far better than PNG. The process of converting a PNG to WebP typically follows this flow:

- **Conversion Path**: `PNG Source → Canvas Object → Blob (WebP) → File Object`
- **WebP Quality Control**: Use `canvas.toBlob({quality:1})` to control output quality, ranging from 0 to 1, where 1 is the highest quality.

By converting PNG images to WebP, you can achieve significant file size reductions without sacrificing visual fidelity, making it an ideal choice for web development.

## 5. PNG Compression, Color Quantization and Pixel Dithering

WebP is great, but what if you want to stick with PNG? Simply Quantizing the color channels and add pixel dithering. That's how tinyPNG works.

And this method is also feasible in the browser side. Here is a simple explanation of the whole process:

- Create a palette using [Image-q](https://github.com/juunini/webp-converter-browser), apply dithering, then compress the image again to reduce extra info using UPNG.js.
- [Image-q](https://github.com/juunini/webp-converter-browser): A pure TypeScript image processing library for browsers and Node.js, supporting various dithering algorithms and palette sampling methods.

<!-- ## 6. Implementation Strategy

A straightforward implementation for a single-page application (SPA) would:
- Allow users to upload multiple images for local conversion and compression.
- Display file details and previews after compression.
- Offer compressed image downloads and upload options directly from the interface. -->

## 6. Optimization Strategies

While the compression methods detailed above work well, there are still several areas where performance can be optimized:

### Parallel Processing
- Using Web Workers can offload image compression tasks to prevent UI blocking during long-running operations. This improves the user experience by allowing the main thread to remain responsive.
- On the other hand, Web Workers have limited access to the DOM and require data serialization, which can complicate implementation.

### Alternative Image Libraries
Switching to more optimized image libraries like @thi.ng/pixel, which leverages WebGL, can drastically reduce the time taken to compress images, especially when dealing with large batches.

### GPU Acceleration
Using WebGL or WebGPU allows image manipulation to be offloaded to the GPU, enabling faster matrix operations and compression times. This, however, requires more advanced implementation and is ideal for high-performance applications.


## 7. Some further discussions

### Pixel Dithering vs. Reducing high-frequency content
- Better Detail Preservation: 
  
  Pixel Dithering involves averaging or blurring adjacent pixels, which helps retain details in the image. In contrast, directly reducing high-frequency content (e.g., through low-pass filters or noise reduction algorithms) may result in excessive loss of details, particularly in edges or textures.
- Reduced Visual Fatigue:
  
  Pixel blending softens sharp high-frequency regions that can cause visual fatigue, especially in areas with high contrast. It makes these regions more comfortable to look at, compared to directly reducing high frequencies, which may result in overly sharp or jarring regions.

- Handling Fine Textures:

  Pixel blending is better at handling fine, complex textures. It can reduce noise while still preserving subtle texture information, whereas directly cutting high frequencies often smudges or blurs these fine details.

### What about the text?

- Text is a high-frequency component that can be challenging to compress without losing clarity.
- Pixel dithering might also blur the text edges, making it less readable.
- For images with text, consider using a gradient upperlimit to avoid dithering in text regions.

## Reference

For additional resources and code implementations, explore the following:
- [TinyPNG](https://tinypng.com)
- [UPNG.js GitHub](https://github.com/photopea/UPNG.js)
- [WebP Image Format](https://developers.google.com/speed/webp)
