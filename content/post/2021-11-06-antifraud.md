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
- [Phase 2: Implementing the algorithm in a bot](#phase-2-implementing-the-algorithm-in-a-bot)
- [Results](#results)
- [Installation](#installation)
- [What's next](#whats-next)
- [Final remarks](#final-remarks)

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

## Phase 2: Implementing the algorithm in a bot

Now that I had the suite, I could implement it quite easily in a bot! 
I decided the name [AndyInAction](https://github.com/Arraying/AndyInAction) was quite fitting.

### Step 1: Writing a basic bot

The most difficult part about implementing the algorithm was using a regular expression that detects URLs properly.
In all honesty, that was a bit of a nightmare.
At the end of the day though, I found a functioning one on the internet and we were good to go!

Once the bot checked if the link was fraudulent and it found it to be fraudulent, something had to be done.
I basically made it log a message to a channel, and if enabled in the config, instant ban the person and delete their recent messages.
Before the instant bad, I had the courtesy to message them that they were going to be banned. 
So when or if the person gets their compromised account back, they'll know what happened.

And it worked so well!

**What staff members see:**
![Banned 1](https://i.imgur.com/s8iqowJ.png)

**What the banned user sees:**
![Banned 2](https://i.imgur.com/FXXWYoo.png)

The message sent to the banned user is customisable.
This is just the message that is used on the [r/ibo](https://discord.gg/ibo) server.

### Step 2: Quality of life

There were two minor issues with the bot.
I invited the bot's first contributor, [`@ilikecubesnstuff`](https://github.com/ilikecubesnstuff) to help out!

Firstly, staff members were not immune.
This is probably ideal security wise, but in the long term staff being paranoid of being instant banned is probably not great.
The solution to this is simple: add a list of role IDs to the config, and if a user sends a suspicious link and have one of these roles, no action is taken.

Secondly, compromised accounts spam links using bots.
In many cases, staff members would see the staff message repeated multiple times, as the ban took a while to go through.
So, he added a small ban/notification cache to ensure that that does not happen.

### Step 3: Redirects

A few hours, maximum one day, after Discord implemented their hash denylist, the scammers decided to evade it.
Fair enough honestly, if I was a scammer I'd probably do that too.
Their technique: redirects!
Scam links were now behind `bit.ly` links. 
This posed a problem for the bot: it did not resolve whatever is behind the redirect.

Luckily, a HTTP `HEAD` request exists.
It's essentially the same as a `GET` request, only that just returns the header and no response body.
So, the solution was for every link encountered, do a `HEAD` and check if it redirects. 
If it does, make sure to check the link it redirects to as well.

If only it were that simple. 
Some domains could not be resolved by DNS. 
This made my GET request incredibly slow as I had to wait for timeouts, and setting the request timeout did nothing.
So in the end, I had to first see if I can resolve the domain (with a short timeout!), and only then check for redirects.

## Results

### Benchmarks

There were no cases of false negatives and some false positives during my beta testing (i.e. figuring out a good config with thresholds).
The final configuration I created produced **no false positives or false negatives**.
This was using ~25 scam links that we collected (some of the domains are not unique, but the links are), and [a bunch of random + selected challenging links](https://pastebin.com/ikzL20sR).

### Performance in production

I introduced *AndyInAction* with banning enabled to the world on the 16th of October.
Since then (until the date of publishing):
- **1943** links were sent in the server.
- **18** unique people were automatically picked up and banned.
- **0** false bans have occurred.
- **A single link** was not detected.
We had to lower a threshold by `3%` and then it got picked up.

That's absolutely *bonkers*!

No, seriously, that's ridiculous. 
Our `@Staff` pings regarding scam links reduced to 0, excluding that link (+ the redirects before that checker was implemented).
The robots have taken over our job!
Perhaps we will finally have a way to combat these scammers, since both the technical measures and awareness that Discord are pushing clearly is not enough to counter the problem.

## Installation

In order to use [*AndyInAction*](https://github.com/Arraying/AndyInAction) on your server, you have to:
1. [Make a bot account and add it to your server](https://discordpy.readthedocs.io/en/stable/discord.html).
2. Clone the repository.
3. Create the `bot.json` config and fill it out.
4. Create the `config.json` config and fill it out.
5. Install the requirements for it to run.
6. Run the bot.

The installation instructions and bot config template is found [here](https://github.com/Arraying/AndyInAction#installation-configuration--usage).
I do however have one gift for you.
Filling out the `config.json` and configuring it is quite tedious.
Below is the configuration that I personally use on my servers.

```json
{
  "domain": {
    "discord": ["co" ,"com", "design", "dev", "gg", "gift", "media", "new", "store", "tools"],
    "discordapp": ["com", "io", "net"],
    "discordmerch": ["com"],
    "discordpartygames": ["com"],
    "discord-activities": ["com"],
    "discordactivities": ["com"],
    "discordcdn": ["com"],
    "discordsays": ["com"],
    "discordstatus": ["com"],
    "steamcommunity": ["com"],
    "steampowered": ["com"]
  },
  "domain_threshold": 0.77,
  "domain_keywords": ["nitro", "gift", "airdrop"],
  "domain_keywords_threshold": 0.4,
  "path": ["nitro", "gift", "airdrop", "trade", "tradeoffer"],
  "path_threshold": 0.9,
  "path_split": false,
  "query": ["gift", "partner"],
  "query_threshold": 0.9,
  "query_split": false
}
```

## What's next

Now that I have a basic bot, the main thing I can do is to sit back and monitor for any changes in the scene and false positive/negatives.
However, there are also various other things that I have planned.

### Machine learning revisited

While using ML to detect if a URL is fraudulent is something I determined not to be too effective for my case, there is still the option to train the thresholds in the config with ML.
However, I can potentially build something with the values that Levenshtein spits out.
If I have the normalized Levenshtein distance for the main domain, the (max value of) the path, querystring, etc. then I could possibly train a classifier.
I doubt this will perform better than what I have working right now, but in the future if I need to compromise on false positive and false negatives ML starts to look like a much more viable solution.

### Making the bot accessible

Currently, installing the bot requires at least some sort of technical know-how.
I want to look into making it easier for server owners with little programming knowledge to install and configure the bot, through a more comprehensive guide, YouTube tutorials, etc.

What would be even nicer is some form of public bot that users can invite.
Currently I lack the time to do this, unless this is definitely something that will work and will have widespread usage.
Honestly though, maintaining public bots is kind of tedious.
If nobody is interested in this option, then there is no point.

## Final remarks

I hope this post was somewhat interesting to you, and you learned something/got inspired by it.
If you have any feedback, feel free to message me on Discord (`Arraying#7363`, I accept pretty much all friend requests).
If you have any ideas on how to improve the bot/develop it further, feel free to message me on Discord or leave a GitHub issue ([link for issues regarding the bot](https://github.com/Arraying/AndyInAction/issues), and [link for issues regarding the algorithm](https://github.com/Arraying/Andy/issues)).

Stay safe out there.
