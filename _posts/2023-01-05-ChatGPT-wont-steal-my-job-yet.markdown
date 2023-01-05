---
title: ChatGPT won't steal my job yet!
layout: post
date: 2023-01-05 00:00
tag: 
- ChatGPT
- AI
- OpenAI
image: assets/images/chatgpt/cover.png
headerImage: true
projects: false
hidden: false # don't count this post in blog pagination
description: My experience with ChatGPT and why it won't steal my job for another couple of years!
category: blog
author: nicods
externalLink: false
---

<img class="image" src="{{ site.url }}/assets/images/chatgpt/cover.jpg" alt="Cover Image"/>

Since its release, we have heard a lot about [ChatGPT](https://openai.com/blog/chatgpt/), how it is amazing, how it will steal our job, how we are close to Touring completeness bla bla. As always, when a new product is released, there is a lot of hype and a lot of people talking about things they know nothing about. With this post I would like to share my experience, to show how we are far from AI taking over the world.

## The Experiment
It was a lazy morning, I was trying to write an API endpoint in Java that, when invoked could add a user of the app to a specific user group. The identity and access management (IAM) service I was using was [keycloak](https://www.keycloak.org/). It offers both a REST API and a Java API to administer its users. Since, as I said, It was a lazy morning, I wasn't inspired to read tons of documentation to find out how to do it. Suddenly, a light bulb turns on and I decide to try ChatGPT, this magic tool everyone was talking about. After signing up I write the question, and magically enough It answers me, showing also a code sample. I was impressed.

<img class="image" src="{{ site.url }}/assets/images/chatgpt/1.png" alt="Experiment 1 Image"/>

But It was an illusion that lasted for a moment. When I tried the code, it was not suited for my version of the library. Why Do not try to give more details? Ok, let's do it:

<img class="image" src="{{ site.url }}/assets/images/chatgpt/2.png" alt="Experiment 2 Image"/>
Surprise surprise, I start recognizing something is not working. ChatGPT keeps showing me code that is not in the library, invented constructors, and so on. It goes on for a while, I get more and more frustrated, and in the end, I give up.
<img class="image" src="{{ site.url }}/assets/images/chatgpt/3.png" alt="Experiment 3 Image"/>
Once the anger has been disposed of, I understand that I don't have to blame chatGPT but all the hype around It. Again, another new technology isn't presented for what it is.   
Now that I am calm again, I want to apologize to get mad and frustrated at ChatGPT. In the end, I find out how to add a user to a group using the Keycloak Admin API and I want to teach it to ChatGPT. But, sadly, my new friend is not able to learn :'(
<img class="image" src="{{ site.url }}/assets/images/chatgpt/4.png" alt="Experiment 4 Image"/>
After this great coming out, instead of accepting defeat, my friend decides to be the most arrogant in the room, and again gives me the wrong advice on how to do it. At the end of the day, I feel a mix of emotions. I am very sad because I lost a new friend, I am not able to be a friend to an AI who wants to make me wrong, for fun or I don't know what. I am also relieved since my job is safe for another couple of years. 

## Conclusions
I am very annoyed by the communications models when new technology is released. A lot of clickbait articles and posts are around, full of misinformation, especially in the tech industry. And this is bad. People capable of understanding what an AI is are few and they know it is not a mature technology, they know AI, as it is intended now, is very limited to some matrix multiplication, convolutions, and derivatives. And we cannot emulate reasoning with math and computers, not yet... Let's try to do not to forget that technology is made to make our lives easier but not to replace us. 