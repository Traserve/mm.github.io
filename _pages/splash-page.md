---
title: "Welcome to Martin's Blog"
layout: splash
permalink: /splash-page/
date: 2016-03-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: /assets/images/unsplash-image-1.jpg
  # actions:
  #   - label: "Learn More"
  #     url: "/terms/"
  caption: "[**Unsplash**](https://unsplash.com)"
excerpt: "不积跬步，无以至千里；不积小流，无以成江海。 `type="left"`"
intro: 
  - excerpt: '个人小站，欢迎交流'
feature_row:
  - image_path: assets/images/unsplash-gallery-image-1-th.jpg
    image_caption: "Image courtesy of [Unsplash](https://unsplash.com/)"
    alt: "placeholder image 1"
    title: "Blog 1"
    excerpt: ""

  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
    alt: "placeholder image 2"
    title: "Blog 2"
    excerpt: ""

    url: "#test-link"
    btn_label: "Read More"
    btn_class: "btn--primary"

  - image_path: /assets/images/unsplash-gallery-image-3-th.jpg
    title: "Blog 3"
    excerpt: ""

feature_row2:
  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
    alt: "placeholder image 2"
    title: "算法"
    excerpt: '算法笔记'
    url: "/categories/algorithm"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row3:
  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
    alt: "技术博客"
    title: "Placeholder Image Right Aligned"
    excerpt: '技术博客'
    url: "/categories/blog"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row4:
  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
    alt: "placeholder image 2"
    title: "随笔"
    excerpt: '记录生活'
    url: "/categories/essay"
    btn_label: "Read More"
    btn_class: "btn--primary"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

{% include feature_row id="feature_row2" type="left" %}

{% include feature_row id="feature_row3" type="right" %}

{% include feature_row id="feature_row4" type="center" %}