---
layout: post
author: Jason Phillips
title: "lama3 intial review"
image: /assets/img/lama3.jpg
tags: 
  - Artificial Intelligence
  - AI
  - META
  - LAMA3
---

# My Intial Impression of Lama3
This post I'll discuss my initial discovery of getting up and running with Lama3. Why would you want to use after all (you've go chatgpt, bing, copilot galor)

<!-- more -->

## Intro: What is "LAMA"
For anyone that isnt aware, LAMA stands for Large Language Model Architecture. It's a type of artificial intelligence that can process and generate human-like text based on input
prompts or sequences. This means it can help with tasks such as language translation, writing text, answering questions, and more.

In simple terms, LAMA is like a super smart computer program that can understand and create written language in the same way humans do. It's an
incredibly powerful tool with many potential applications.

LAMA works by recieveing input (also called prompts) reply back with content. LAMA is completely powerful tool for applications like chatbots, virtual assistants, and content creation.

## LAMA3 vs ChatGPT
On the surface: LAMA3 and ChatGPT 4 (latest model of ChatGPT; alough o3 is in preview) are both language models for Generative AI
 * LAMA3 is well-suited for tasks like language translation, writing assistants, and content generation.
 * ChatGPT-4 is designed for more dynamic applications like chatbots, voice assistants, and even potentially, AI-powered customer service agents.

There are also other considerations

### Localization
- Lama3 can be hosted locally. Theres an extra bit of ease when work with clearance data where data resides and stored location cleared 
- ChatGPT is service hosted on the cloud (where ever those servers are). With that this can be seen as loss of control by some organizations

### Sharing Data / Training Data
- ChatGPT trains itself from prompts that sparked privacy concerns both by individuals and enterprises. Although this can be turned off.
- LAMA 3 however.. your input prompt and output it generates stays entirely entirely local without any connection to the outside world.

## Invoking LAMA
- Lama3 can be involved directly from command line terminal
- Our invoked via a restapi

## Commercialization of OpenAI
The following maybe also be seen as negative for ChatGPT
- The developers of ChatGPT (OpenAI) started down earnest paths of being an open Production with is founding 
- In last year, OpenAI has announced they indeed will become a for proit company

Ok with all that being said ... How do we start Lama
## Install LAMA3
- To use LAMA3 there a few options
- You can leverage the Lama via platform like Olama or LM Studio
- Then download the LAMA3 module
- For me the simplist option was to leverage docker container (under 10min)
```
sudo docker exec -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:latest

```
*Note 
- I keep local volume for data to persist put I could remove
- I also didnt enable gpu for my container which can help spead up the proces

You should now be able to access ollama on port 11434
<b> There's one more step  </b>
To run interactive command/chat ssession
```
sudo docker exec -it ollama ollama run llama3
```
You'll a prompt the lama service
sudo docker exec -it ollama ollama run llama3
>>> '>> Send a message (/? for help)'

## Performance
- Compared to ChatGPT performace is a slower (but its not unbearable). It's worth mentioning I'm running Lama3 on my 32bit laptop with GPU not enabled on docker and no NPU
- I would imagine I would get more gains if I enabled this will another blog based on my findings after that
## Version Info

``` /show info
  Model
    architecture        llama
    parameters          8.0B
    context length      8192
    embedding length    4096
    quantization        Q4_0

  Parameters
    num_keep    24
    stop        "<|start_header_id|>"
    stop        "<|end_header_id|>"
    stop        "<|eot_id|>"

  License
    META LLAMA 3 COMMUNITY LICENSE AGREEMENT
    Meta Llama 3 Version Release Date: April 18, 2024'
```



## Verdict
- I'm truly surprised by Lama3's cabality
- Also some responses are clearly better answers (surprisingly) over the ChatGPT for example this prompt <b>"how do take a flat color profile with my samsung s23 ultra camera" </b>. Lama3 clearly gave me an honest anser that my s23 doesnt support this request but gave me steps to complete with other processes like Camera RAW. ChatGPT gave me answer that was flat out wrong
- I really do like the fact the model is local and I can be assured my prompts are going out to Internet
- Its worth mentioning there is a more power Lama3 Model that recommend to have at least 24GB to run. I'm looking for forward to seeeing how i can push the Lama to the limits and potential subsized if at all remove my dependancy on OpenAI