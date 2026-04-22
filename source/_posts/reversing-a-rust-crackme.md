---
title: 'Reversing a Rust CrackMe!'
date: 2018-04-03 04:00:00
tags:
  - reversing
thumbnail: https://i.imgur.com/uu0OGdf.png
---

<h1 id="Dear-Reader"><a href="#Dear-Reader" class="headerlink" title="Dear Reader,"></a>Dear Reader,</h1>Hello and welcome to this post, I will discuss my adventures on a Rust CrackMe I saw in a Reverse Engineering Discord server, this CrackMe as you may know already is powered by the Rust Lang, This language is AMAZING and it’s really close to C++.

<h1 id="Hello-Rust-Nice-to-Meet-You"><a href="#Hello-Rust-Nice-to-Meet-You" class="headerlink" title="Hello Rust! Nice to Meet You!"></a>Hello Rust! Nice to Meet You!</h1>

![]\(https://i.imgur.com/uu0OGdf.png\)

So, I downloaded the CrackMe and quickly went into my good ol’ bud x64DBG, which is the one I commonly use for my RE projects… Started analyzing it and… Wait what? as soon as I hit the <code>OEP</code> I start seeing a bounch of weird things (being new to a Rust binary, that’s quite understandable), eg. instead of having a normal control flow, I started seeing how I had to keep on going further deep in between calls and things like that. So that was quite painful to go through, Also I couldn’t trace step out because it would obivously would skip the calls and then I would be out of luck, and also couldn’t trace call stack because it showed no hints what so ever of what was being called, it would always show calls from kernel and stuff, but not from the internal binary.

<h1 id="IDA-to-the-rescue"><a href="#IDA-to-the-rescue" class="headerlink" title="IDA to the rescue!"></a>IDA to the rescue!</h1>So as I called it a defeat with x64DBG, I called in IDA, and started working on it, IDA has cool graphs and plugins that will help you setup your pace back on tracks and will allow you to have a more broader approach when analyzing something…

So once I started working on it, I noticed where the binary had some interesting strings showing up, so I went there and lo’ and behold there they were!

![]\(https://i.imgur.com/9yPrps6.png\)

So I went to where it was being used, and here we are, finally!

![]\(https://i.imgur.com/Sv0wOkt.png\)

So I quickly assumed this was the main function where everything was going down, so I love the renaming feature of IDA so why not use it? 😜

So, not only I noticed that Rust does all those weird things with the control flow as I mentioned before, but also the compiler adds a bunch of other junk stuff, however thankfully, now that I got the main function, I can call it a day and skip all the junk stuff, and just focus on this part only.

<h1 id="CrackMe-What’s-your-sorcery"><a href="#CrackMe-What’s-your-sorcery" class="headerlink" title="CrackMe, What’s your sorcery?"></a>CrackMe, What’s your sorcery?</h1>So this CrackMe, (skipping ahead some prior reversing) was relatively simpe as the name suggests, not minding any of the junk stuff compiler adds, I just had to focus on this part:

![]\(https://i.imgur.com/MU0CGfY.png\)

This is where the meat is, more importantly inside <code>checkPwdHash</code> at <code>0xE11411</code> function that you see there, is where the actual algorithm that checks password is.

and it’s as simple as, getting all the bytes in a byte arry, and sum them all together, the output will be the checksum for the string, viola!

so in python it would be someting like this:

<pre><code class="language-python">def calc_checksum(the_bytes):
    return b&#39;%02X&#39; % (sum(the_bytes))

bytes = [0x70, 0x65, 0x70, 0x65] # bytes = &#39;pepe&#39;

print(calc_checksum(bytes)) # output: b&#39;1AA&#39;
</code></pre>