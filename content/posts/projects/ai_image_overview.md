---
title: "AI Image Overview"
date: 2026-01-24T22:27:39+01:00
draft: false
summary: >
  Combining Canary Synthetics + Lambda + BedRock to periodically describe a graph.
cover:
  image: /posts/projects/images/ai_image_overview_components.png
---

# Intro
I'd like to start by saying that this implementation was done as a test for the flow itself, what was picked was merely for demoing. 
IRL the use case is different, the flow remains the same, now I can get started.

AI Image overview takes an image as an input, sends it to Bedrock LLM, gets the description, that's it in a nutshell. 

## Prerequisites
  - [CloudWatch Synthetics Chrome browser plugin](https://chromewebstore.google.com/detail/cloudwatch-synthetics-rec/bhdnlmmgiplmbcdmkkdfplenecpegfno)
  - An S3 Bucket to store the screenshots (Will also be used as a source that will trigger the lambda function which invokes Bedrock).

## Implementation
### 1. [Canary Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html)
  - Used for monitoring periodically which is why it's great for this use case because the idea is to have weekly screenshots from specific dashboard.
  - Blueprint used was **Canary Recorder** which allows you to record your actions on a specific website and it repeates those actions for you.
  - Source script can be found [here](https://github.com/theJaxon/ai_image_overview/blob/main/canary_synthetics/recordedScript.js) and what it does is that it reaches to coincap, gets the Bitcoin page and screenshots it.

### 2. Lambda
  - Configured with the created S3 bucket as a trigger so that whenever canary synthetics run, the lambda function also runs since files were uploaded to S3.
  - S3 event is checked and we extract the current path of the screenshot so that we describe the correct image for each run.
  - Needs permissions to access the S3 Bucket and Bedrock.

### 3. Bedrock
  - Gets invoked via Lambda and is the service used to describe what's in that image.

## References
- [ai_image_overview repository](https://github.com/theJaxon/ai_image_overview)

---

So far, the implemntation was done through the console, later I'll try to automate it with IaC tool.

Big thanks to [Kiro IDE](https://kiro.dev).

Ciao.