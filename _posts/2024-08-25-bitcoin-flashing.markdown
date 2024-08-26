---
title: Bitcoin Flashing
layout: post
date: 2024-08-25 16:30
tag: 
- Bitcoin
- Flashing
- Scam
image: assets/images/
headerImage: true
projects: false
hidden: false # don't count this post in blog pagination
description: A guy asked me to replicate Bitcoin flashing software he read about online. Here are the results.
category: blog
author: nicods
externalLink: false
---

 <!-- TODO edit image -->
<img class="image" src="{{ site.url }}/assets/images/btc-flash/Bitcoin.jpg" alt="Cover Image"/>

A guy asked me to replicate Bitcoin flashing software he read about online. Here are the results.

## Part 1: The Request and the Context

A guy from the tennis club I play at (yes, software engineers do have lives away from their computers) reached out to me. He asked if I could meet a friend of his who needed help with a cryptocurrency problem. After a few phone calls to understand the request, it turned out they wanted me to create scam software that could transfer Bitcoin from one wallet to another, only to have the transaction automatically rollback after X days.

After Googling around, I noticed that some people are selling the goose that lays golden eggs on Telegram, Instagram, or [websites](https://coinflashr.com/) (they even advertise this kind of software on [GitHub](https://mmdrza.com/flash-aio-v1-0-3-ultimate/)).

I was a bit skeptical about the request, and even more so about the legitimacy of this software, which promised the ability to issue Bitcoin transactions without any money. But, since we live in an endless quest for financial independence and self-sustainability without sacrificing our time working for others, I thought it might be worth digging deeper to see if there really was a way to grow money on trees.

The quest wasn't successful but was enlightening. I discovered that Bitcoin flashing is a scam technique, perfectly explained at [bitcoinflashing.com](https://bitcoinflashing.com/bitcoin-flashing-in-2024/) (no, it's not a joke!).

Here’s how it works: The Bitcoin network takes some time (10 to 180 minutes on average) to validate a transaction, i.e., moving money from one wallet to another, because some peers in the network need to validate it (you may have heard about the mining process). Peers get a fee to validate the transaction; the higher the fee, the faster the validation. Until validation occurs, you will see the transferred Bitcoin in the recipient’s wallet, but they are in a "pending" state. You won’t know for sure that the Bitcoin is yours until the network approves or rejects the transaction. The catch is: if the fee is too low, the network's peers will never validate the transaction because it’s not worth their time. The transaction will remain in a "pending" state, and you could trick someone into thinking it will eventually be approved when it actually won't.

## Part 2: The Experiment

I decided to tell the tennis club guy’s friend that it wasn’t feasible. But at this point, I was intrigued by the technique and disappointed that there wasn’t a single repository on GitHub with open-source code for "flashing Bitcoin" without asking for money (aren't we supposed to make them appear from nowhere, wtf!!!). So, I decided to figure out how to make this feasible.

The first step was to search for tools that would make our lives easier. Eventually, I chose Python with [bitcoinlib](https://bitcoinlib.readthedocs.io/en/latest/). I then tried, unsuccessfully, to create a raw transaction from one wallet to another with the help of GenAI and documentation. I failed for two reasons: 
1. I wasn’t constructing the transaction properly.
2. The library performs integrity checks before issuing the transaction.

After this severe failure, I was ready to give up, but then, in the darkest moment, inspiration struck.

At this point, I need to digress about modern text editors. I follow some folks online (yes, [theprimeagen](https://www.youtube.com/c/theprimeagen) is one of them) who advocate for one text editor or another. I use modern text editors like IntelliJ or VSCode because I struggle to remember my birthdate, let alone learning infinite character sequences to complete a task. Without taking sides in this debate, there's one feature of modern text editors that helps us in this story: code navigation (maybe vintage editors have this feature too, but I’m not sure...). Basically, you can click on a function name, and the editor will take you to the part of the file where the function is implemented, allowing you to quickly check the code. If you click on the name of a function/method in a library, you’ll be brought to that function's implementation code (for some languages, like Java, it automatically decompiles the code too).

And here’s where the inspiration I was talking about comes in: since the bitcoinlib Wallet object has a method called `send` where you can customize the fee associated with a transaction, why not use it and remove all the blocks of code that check the integrity of a transaction before issuing it to the actual Bitcoin network using code navigation? So, I modified the `wallet.py` file in bitcoinlib version 0.6.15 as follows:

<img class="image" src="{{ site.url }}/assets/images/btc-flash/1.png" alt="[1] Custom wallet.py"/>

<img class="image" src="{{ site.url }}/assets/images/btc-flash/2.png" alt="[2] Custom wallet.py"/>

One last catch before success: the Bitcoin mempool will perform balance validation, i.e., we can flash a transaction, but our wallet must have some UTXOs (for details on how this works, check [here](https://www.alpsblockchain.com/en/media-news/how-bitcoin-transactions-work)). We cannot flash more than the UTXOs we possess.

Finally, you can find the completed Python script on my GitHub@[btc-flashing-demo](https://github.com/nicoDs96/btc-flashing-demo).

## Conclusion

Sadly, we still haven’t found money growing on trees. :(