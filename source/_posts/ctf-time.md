---
title: 'Trellix - Google AndroidCTF Team Challenge 2025'
date: 2025-10-22 04:00:00
tags:
  - android_re
thumbnail: https://imgur.com/qKqrowJ.png
---

<h1 id="Introduction"><a href="#Introduction" class="headerlink" title="Introduction"></a>Introduction</h1>Hello and welcome to this post, I will discuss my adventures on the Google AndroidCTF Team Challenge 2025, which I tackled as part of the process for a Trellix Android Reverse Engineer position! This challenge was really profound and insightful, and as you may know already, Android reversing can be quite a ride.

<h1 id="Initial-Analysis-Deobfuscation"><a href="#Initial-Analysis-Deobfuscation" class="headerlink" title="Initial Analysis &amp; Deobfuscation"></a>Initial Analysis &amp; Deobfuscation</h1>

![]\(https://imgur.com/qKqrowJ.png\)

So, I downloaded the APK and quickly went into my good ol’ bud JADX-GUI, which is the one I commonly use for my Android RE projects… Started analyzing it and… Wait what? The package was heavily obfuscated, so a little deobfuscation would come in handy. After cleaning it up, I checked the <code>AndroidManifest</code> and <code>MainActivity</code>, but they were pretty bland.

I started digging deeper into the <code>Dashboard</code> and <code>Home</code> components, but the real fun started when I hit the <code>Notifications</code> section. I started seeing a bounch of weird things, eg. imports for <code>ByteArrays</code>, <code>IO Stream</code> and <code>Cipher</code>. So that was quite interesting to go through.

I kept digging further and lo’ and behold there it was!

![]\(https://imgur.com/7qwSfrX.png\)

I noticed a native function declaration <code>Fragment(String str)</code> and some asset loading logic. It seemed like it was loading an asset into a byte array and doing some XOR operations with decimal key 69.

<h1 id="Native-Library-Analysis"><a href="#Native-Library-Analysis" class="headerlink" title="Native Library Analysis"></a>Native Library Analysis</h1>So as I hit a wall with just Java analysis, I noticed the APK had a native library embedded called <code>libfragment.so</code>. Since <code>Fragment(&quot;...&quot;)</code> was being called to prepare keys, I called in IDA, and started working on it.

So once I started working on it, I found the <code>Java_com_google_ctf_ui_notifications_NotificationsFragment_Fragment</code> function. This caught my attention since it would basically be the call that was going on in the Java function.

![]\(https://imgur.com/VN4mDm0.png\)

So I love the renaming feature of IDA so why not use it? 😜

I noticed that the Java side was loading <code>google.png</code>, chopping off 71612 bytes, and treating the rest as an encrypted payload. It was using AES&#x2F;ECB&#x2F;PKCS5Padding. But the key? It was passing “15539863” to that native function. If I just used that string, it would be junk. The native library was actually doing the heavy lifting to generate the real key.

<h1 id="Key-Generation-Decryption"><a href="#Key-Generation-Decryption" class="headerlink" title="Key Generation &amp; Decryption"></a>Key Generation &amp; Decryption</h1>So this Challenge, (skipping ahead some prior reversing) was relatively simpe once I looked inside the native code. Not minding any of the junk stuff, I just had to focus on this part where the key was generated.

![]\(https://imgur.com/reaLFYA.png\)

This is where the meat is, the native function takes the input string and XORs it byte-by-byte with a secondary key string “FragmentsControl”.

and it’s as simple as, getting that derived key, and using it to decrypt the payload inside <code>google.png</code>, viola!

so in python the key generation would be someting like this:

<figure class="highlight python"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br></pre></td><td class="code"><pre><span class="line"><span class="keyword">def</span> <span class="title function_">generate_key</span>(<span class="params">input_str</span>):</span><br><span class="line">    strFragmentCtrl = <span class="string">&quot;FragmentsControl&quot;</span></span><br><span class="line">    strLen = <span class="built_in">len</span>(input_str)</span><br><span class="line">    </span><br><span class="line">    <span class="comment"># First byte logic</span></span><br><span class="line">    prepped_key = [<span class="built_in">ord</span>(input_str[<span class="number">0</span>]) ^ <span class="number">0x46</span>]</span><br><span class="line">    </span><br><span class="line">    <span class="comment"># Loop for the rest</span></span><br><span class="line">    <span class="keyword">for</span> i <span class="keyword">in</span> <span class="built_in">range</span>(<span class="number">15</span>):</span><br><span class="line">        val = <span class="built_in">ord</span>(input_str[(i + <span class="number">1</span>) % strLen]) ^ <span class="built_in">ord</span>(strFragmentCtrl[i + <span class="number">1</span>])</span><br><span class="line">        prepped_key.append(val)</span><br><span class="line">        </span><br><span class="line">    <span class="keyword">return</span> <span class="built_in">bytes</span>(prepped_key).<span class="built_in">hex</span>().upper()</span><br><span class="line"></span><br><span class="line"><span class="built_in">print</span>(generate_key(<span class="string">&quot;15539863&quot;</span>)) </span><br><span class="line"><span class="comment"># Output: 77475454545D584742765A5D4D4A595F</span></span><br></pre></td></tr></table></figure>

<h1 id="Conclusion-Flag"><a href="#Conclusion-Flag" class="headerlink" title="Conclusion &amp; Flag"></a>Conclusion &amp; Flag</h1>

![]\(https://imgur.com/rJzk1Fx.png\)

<br>Once I had the key, I decrypted the payload and found a hidden DEX file. I dropped that back into JADX and found a method called <code>TheFlagReal</code> containing:

<code>Android_CTF{Peel1ng_L4y3rs_0ne_by_0ne}</code>

This was a really a really cool excercise!

So, first, I’m sorry if I didn’t provided every single step of the dynamic reflection analysis, but as the title suggested it was just a write-up, and the challenge encouraged strict static analysis anyway.

Second, shout out and thanks to the Trellix team for the opportunity.

And that’s it, I’m out!