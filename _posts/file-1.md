---
title: CI/CD Pipeline for Versioned Docker Releases Using GitLab
author: Vamsi Maniyam
date: 2025-06-27 12:00:00 +0800
categories: [Devops]
tags: [gitlab]
math: true
mermaid: true
---


# CI/CD Pipeline for Versioned Docker Releases Using GitLab


```mermaid
graph TD
    A[Push to release-X.X.X branch] --> B[Trigger GitLab CI/CD pipeline]
    B --> C[Build Docker Image with index.html]
    B --> D[Create Git Tag]
    C --> E[Push image to DockerHub]
    E --> F[Manual Deployment Job]

