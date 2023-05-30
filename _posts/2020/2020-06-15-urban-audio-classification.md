---
title: "Urban Audio Classification"
categories:
  - Blog
tags:
  - link
  - Post Formats
link: https://github.com/grudloff/Salamon2017Replication
---

Replication of a classical paper by Salamon et. al on urban audio classification. The model is a CNN that takes as input
melspectrogram images computed from sound excerpts for which data augmentation is utilized for improved generalization.
The CNN model is implemented in keras, while the preprocessing and augmentation are done through librosa and muda, respectively.
