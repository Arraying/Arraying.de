---
title: "Building the Best Anti-Fraud Bot You've Never Heard Of"
author: ""
type: ""
date: 2021-11-06T18:30:54+01:00
subtitle: "An adventurous dive into the world of fraudulent Steam/Discord links"
image: ""
tags: ["python", "andy"]
---

I made a Discord bot that will automatically detect fraudulent Steam/Nitro links on Discord, and ban the person sending these.
The impressive part: it works on never-before seen links, and its false-positive and false-negative rate is *ridiculously* low.
It's totally open source and you can protect your server too!
Check out the [Installation](#installation) section if you are interested.

---

This post is quite technical.
If you're interested in how I developed this idea, then you are in for a treat.
If not, feel free to skip to wherever you want.

**Table of contents:**
- [The plague](#the-plague)
- [Phase 1: Developing the algorithm](#phase-1-developing-the-algorithm)
- [Phase 2: Implementing the algorithm in a bot](#phase-2)
- [Results](#results)
- [What's next](#whats-next)
- [Installation](#installation)

***Note:** This blog post contains domains associated to fraud. 
DO NOT VISIT LINKS IN CODE BLOCKS.
THEY ARE FRAUDULENT LINKS.
All sites that are safe in this post are **hyperlinked**.*

---

## The plague

For a few months now, Discord has been plagued by botnets of stolen accounts that try to scam users.
You've probably seen them around. They look something like this:

![Scams](https://i.imgur.com/g7ZYCES.png)

If you look closely, you'll realise that the links (which I blurred) lead to non-Steam/Discord websites.

The principle behind these scams is simple: compromised accounts spam servers and DMs with links to a fake Nitro redemption site.
Victims then enter their username and password, and become compromised themselves.

The problem got really bad the last few weeks.
I started getting pinged too often because of some sort of scam going on.
Annoyed at the mild inconvenience of a `@Staff` ping, I decided:
**Enough of that! Let's combat Discord & Steam fraud at never-before seen effectivity.**

## Phase 1: Developing the algorithm

### Step 1: What kind of links are we actually dealing with?

First things first!
You can't build an effective anti-fraud algorithm without knowing what kind of data you're dealing with.
Thankfully, one of my fellow staff members was kind enough to provide me with a list of domains.

An exerpt can be seen below:
```txt
https://dlsc​ord.or​g/information-nitro
ht​tps://discordap​p.click/free/nitro
htt​ps://steamcommu​nity-nitro.ru/general
```

As you can see, these links generally follow one of the following trends:
1. A legitimate domain with a non-official TLD.
2. A slightly misspelled domain similar to the official domain.
3. An official looking domain and TLD domain, however some additional keywords are added using hyphens.

The specific type of fraud with misspelled URLs is called **typosquatting**. 
I will be referring to this quite a lot.
We need our algorithm to be able to deal with all 3 of these scenarios.

### Step 2: Exploring our options

Now that we know what we are working with, we need to explore the different ways we can combat these links.

**Denylisting**

The first "solution" that came to mind is to keep a denylist of all domains that are known to be fraudulent.
This solution is, to put it bluntly, primitive.
It's a cat and mouse game between scammers and those keeping the denylist up to date.
New links will never be known until an incident occurs.
As such, I regarded this as a worst-case scenario.

Discord actually [implemented](https://cdn.discordapp.com/attachments/159020907539464193/905601839146340382/Malicious_Link_Blocker.png) a basic denylist using hashes of malicious domains.
Honestly, I'm not sure what I think of it.
It's something, but at the same time, scammers can and easily will get around it.

**Regular expressions**

Naturally, the next step is some form of regular expressions.
We already had a chat-filter bot that supported regular expressions, which we extended with regular expressions for Steam scams.
RegEx does support unknown links, which was definitely a good start.
The problem is that you need to provide some sort of format, and abstracting such a format to be flexible enough to catch the large variety of links is challenging.
For instance, [this is what a URL RegExes looks like](https://mathiasbynens.be/demo/url-regex), which are all not perfect.
RegEx was quite effective at first, however once the Nitro scams started rolling in, our filter essentially became useless.
We just had a huge variety/scalability problem, so I needed to look into something else.

**Machine learning**

Here's a buzzword for you: machine learning.
I read a few papers on using machine learning to detect fraudulent URLs.
However, while ML is definitely incredibly powerful, it still wasn't enough for this.

There were three main problems I came across:
1. **Training.** I had ridiculously little scam links, so training the algorithm becomes a problem. 
In addtion to this, there are only maybe two or three legitimate links for Discord and Steam each. With such little data, it would be incredibly difficult to train a model that does not overfit.
2. **Features.** A lot of the research papers I came across used upwards of 20 features. 
This amplifies my training problem due to the curse of dimensionality. 
But even worse, I didn't even have a lot of the features they used to work with. 
For example, they used the length of the domain (i.e. to combat random character mash domains) or if the domain was an IP address. 
Neither of those applied to me.
3. **Performance.** Machine learning can be really accurate in their detection, if done right. 
That being said, there is a reason why many video game anticheats do not use machine learning. 
Non-learning algorithms can just provide better detection performance.
In this case, I had a hunch that a learning algorithm would not provide low enough false positives.

**Levenshtein distance**

Just was I was giving up, I came across [Levenshtein distances](https://en.wikipedia.org/wiki/Levenshtein_distance). 
Simply put, it can measure the similarities of two strings.
Spellcheckers use Levenshtein distance to compute their predictions.
If we have a domain, `discord.com` and a scam domain `d1scord.com`, we could see how similar they are.
Given that they are "similar enough", this could be an indication that the link is fraudulent.
I just found the perfect method for typosquatting!

### Step 3: Writing the algorithm

I decided to write a suite dedicated to the algorithm, called [Andy](https://github.com/Arraying/Andy).
Instead of Levenshtein distance, I opted to [normalize](https://stackoverflow.com/questions/45783385/normalizing-the-edit-distance) the Levenshtein distance.
That way, I had a continous range `[0-1]` of how similar two strings are, from completely different to identical.

The first problem was actually getting the domain from the link.
Using `urllib` I could parse the domain (`netloc`). 
However, splitting that into the TLD and the actual domain someone could register was quite challenging.
I ended up using the `tldextract` package for this.

Next, I had to get something to compare each fraudulent domain to. 
I opted for a configuration file where the you can define the legitimate domains (+ TLD), and then test link domains will be checked against these.
If the link similarity is above a certain threshold, there is a high probability this is a scam.

In addition to this, I also added keyword checking (is a certain keyword in the domain?), path checking (does the path contain some sort of keyword(s)?), and querystring checking (are there any suspicious querylinks?). 

With that, trend `#2` (refer to the data exploration section) was taken care of. 
I added some basic checks for trend `#1` and `#3` and the suite was good to go!

If you are interested, [the source code can be found here](https://github.com/Arraying/Andy/blob/main/src/andy/suite.py).
I encorage you to read it to see what is going on, it's a little bit too complex to describe here but it should hopefully be somewhat comprehensible.
