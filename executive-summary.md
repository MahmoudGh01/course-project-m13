# AP News Performance Audit: Executive Briefing & Investment Strategy

## The Bottom Line
We analyzed the AP News mobile web experience to answer one question: is our frontend performance costing us money and reader engagement, and if so, where should we invest to fix it? 

The short answer is **yes**. While our backend infrastructure is rock-solid, our frontend delivery—specifically how we load ads, fonts, and images—is creating a sluggish experience that drives away readers and puts our Google search rankings at risk. 

By strategically funding frontend performance work, we can significantly improve reader retention and safeguard our ad inventory without needing to re-architect our backend systems.

## What is Working Well
Your infrastructure and DevOps teams have built a highly resilient foundation. This report is not a tear-down; it is a roadmap to bring the frontend up to the standard of your backend.
* **Traffic Spike Resilience:** Our CDN (Cloudflare) edge caching is configured exceptionally well. When major news breaks, our servers instantly absorb massive traffic spikes without going down.
* **Modern Delivery Protocols:** The site uses top-tier compression and modern network protocols. Data is moving as efficiently as possible from our servers to the user's device. 

---

## Ranked Investment Roadmap
Not every issue costs the same to fix, nor do they yield the same return. Below is the ranked order of where we should invest engineering time, categorized by business risk, verifiable evidence, and cost to fix.

### Priority 1: Stop Immediate Reader Abandonment (Fund This Sprint)
These are targeted configuration changes that cost almost nothing to implement but will immediately stop readers from leaving in the first three seconds of their visit.

* **Fix the "Invisible Headline" Problem**
  * **The Risk:** Readers click a link, the page structure loads quickly, but custom fonts block the actual news text from rendering. Users stare at a blank screen and bounce.
  * **The Evidence:** Open AP News on your phone while on a slow cellular connection (or toggle airplane mode on and off). You will see the layout appear, but the headlines will be completely invisible for a frustrating 2-4 seconds.
  * **The Cost:** Trivial (< 1 week). A simple CSS configuration change will show standard text immediately until the brand font loads.
* **Prioritize the Hero Image**
  * **The Risk:** The main story image competes for bandwidth with invisible background scripts, delaying the visual impact of top stories.
  * **The Cost:** Trivial (< 1 week). Adding a single priority tag forces the browser to load the hero image first.

### Priority 2: SEO & Ad Revenue Protection (Fund This Quarter)
These items require moderate engineering effort but are critical for protecting our SEO rankings (Google heavily penalizes slow sites via Core Web Vitals) and ensuring our ad inventory remains high quality.

* **Stop the Page from Jumping (Cumulative Layout Shift)**
  * **The Risk:** Readers start an article, and 3 seconds later, an ad pops in and violently pushes the text down. This causes readers to lose their place and leads to accidental ad clicks. Advertisers heavily discount inventory with high accidental click rates, hurting long-term revenue.
  * **The Evidence:** Open any article on your phone and start reading immediately. Watch the text jump away from your eyes as ads and widgets load.
  * **The Cost:** Moderate (1–2 Sprints). Engineers need to reserve blank geometric space for ads *before* they load, so the text never moves.
* **Reduce the Payload (Page Weight)**
  * **The Risk:** We are sending massive bundles of unused code and oversized images to mobile devices. This eats user data caps and delays ad loading.
  * **The Cost:** Moderate (2 Sprints). Implement code-splitting (only sending the code needed for the specific page) and automated image resizing. 

### Priority 3: Structural Debt (Strategic Initiative for Q3/Q4)
These are expensive, structural problems tied to how we currently monetize the site. They require cross-functional alignment between Engineering, Product, and Ad-Ops.

* **Unblock the Browser (Total Blocking Time)**
  * **The Risk:** The site currently loads over 350 background requests for tracking, ad-bidding, and analytics. This completely locks up mobile devices. When users try to tap a menu or scroll the feed, the site freezes. 
  * **The Evidence:** Scroll down the homepage quickly on a mid-tier Android phone. Notice the stuttering, unresponsiveness, and "frozen" feeling.
  * **The Cost:** High (Architectural shift). We need to move ad-tech and analytics off the main browser thread and adopt an "Islands Architecture" to only load interactive code where necessary. 
* **Fix Live Blog Browser Crashes (Memory Leaks)**
  * **The Risk:** During major, continuous news events (like Election Night), our Live Blogs append new updates forever without cleaning up old data. This drains user battery life and eventually crashes their browser outright, capping our maximum possible engagement.
  * **The Evidence:** Leave a busy live blog open on your phone for 30 minutes. The device will grow noticeably warm, and scrolling will become impossibly slow before the tab crashes.
  * **The Cost:** High (1-2 Months). Requires rebuilding the live blog feed to use "DOM Virtualization"—a technique that unloads older stories from the phone's memory as the user scrolls past them, keeping the app fast and crash-free.

## Next Steps
I recommend we approve **Priority 1** for the immediate upcoming sprint, as the ROI is massive relative to the trivial effort. We should schedule **Priority 2** for the current quarter to stabilize our search rankings. Finally, I propose we commission a joint working group between Engineering and Ad-Ops to scope the cost of **Priority 3**, as our current ad-tech implementation is fundamentally bottlenecking our growth on mobile.
