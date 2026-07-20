# AP News Performance Audit: Developer Implementation Guide

This document contains the technical mechanisms, reproduction steps, and required fixes for the frontend performance bottlenecks identified on AP News. 

**Global Reproduction Environment**
Unless otherwise specified, all issues below were identified and should be verified using Chrome DevTools with the following settings:
*   **Device:** Mobile (Moto G Power or equivalent narrow viewport)
*   **Network Throttling:** Slow 4G
*   **CPU Throttling:** 4x slowdown
*   **Cache:** Disabled (for initial load metrics)

---

## 1. Local Fixes (Component/Template Level)
*These items are isolated to specific CSS/HTML templates and can be picked up in a standard sprint with minimal regression risk.*

### A. FOIT (Flash of Invisible Text)
*   **The Mechanism:** The site uses custom web fonts but does not specify a fallback rendering strategy. The browser hides all text nodes relying on these fonts until the `.woff2` files finish downloading.
*   **Reproduction:** 
    1. Open the Homepage.
    2. In DevTools, go to the **Performance** tab and record a reload.
    3. Look at the "Screenshots" track. You will see the layout paint (FCP), but the text remains completely invisible for 2+ seconds until the font network request completes.
*   **The Fix:** Add `font-display: swap` to all `@font-face` declarations in the global CSS. This instructs the browser to immediately render the text using a system font and swap it once the custom font is ready.
*   **Validation:** Rerun the Performance profile. You should see text rendered in the exact same frame as FCP.

### B. Hero Image LCP Delay
*   **The Mechanism:** The main top-of-fold image is treated like any other image. It competes for bandwidth with bottom-of-fold tracking scripts and CSS. Additionally, it is often served as a heavy JPEG instead of WebP/AVIF.
*   **Reproduction:**
    1. Open the Homepage.
    2. In the **Network** tab, filter by `Img`. Notice the hero image begins downloading late in the waterfall.
    3. In the **Performance** tab, check the LCP marker. It triggers extremely late, directly tied to this image's load time.
*   **The Fix:** 
    1. Inject `<link rel="preload" as="image" href="[hero-image-url]" fetchpriority="high">` into the `<head>` of the server-rendered HTML for the homepage and article templates.
    2. Ensure the `src` being preloaded points to the Next-Gen format (WebP/AVIF) provided by the image CDN.
*   **Validation:** In the Network waterfall, the hero image request should now appear in the first block of requests, immediately after the HTML document.

### C. Layout Shift (CLS) from Dynamic Ad Slots
*   **The Mechanism:** Ad slots (via GPT/Prebid) and related-article widgets are injected into the DOM asynchronously. Because no geometric space is reserved for them beforehand, they push the text down when they render, triggering massive Recalculate Style and Layout tasks.
*   **Reproduction:**
    1. Open any standard Article page.
    2. In DevTools, open the **Rendering** drawer and enable "Layout Shift Regions".
    3. Scroll down slowly. As ads load in, the screen will flash blue, indicating a layout shift.
*   **The Fix:** Identify the wrapper `<div>` for all ad slots (e.g., `.ad-slot-container`). Assign a CSS `min-height` or `aspect-ratio` that matches the expected dimensions of the ad (e.g., `min-height: 250px` for an MREC ad) *before* the ad JS executes. 
*   **Validation:** With "Layout Shift Regions" enabled, reloading and scrolling the page should produce zero blue flashes around ad containers.

### D. Animation Jank (Breaking News Ticker)
*   **The Mechanism:** The Breaking News ticker is animated using `margin-left`. This property triggers the browser's Layout pipeline on every single frame, dominating the main thread and causing dropped frames (stuttering).
*   **Reproduction:**
    1. Open the Homepage.
    2. Go to the **Performance** tab and record 5 seconds while the ticker is moving.
    3. Look at the Main Thread track. It will be a solid block of purple ("Layout").
*   **The Fix:** Rewrite the CSS keyframe animation to use `transform: translateX()`. This offloads the animation to the GPU's compositor thread, bypassing layout calculations. Remove any unnecessary `translateZ(0)` hacks attached to static article cards, which are currently causing memory bloat via excessive layer creation.
*   **Validation:** Rerun the Performance profile. The purple layout blocks should disappear, and the animation will run at a smooth 60fps.

---

## 2. Structural Fixes (Architecture/System Level)
*These require cross-team coordination (Engineering, DevOps, Ad-Ops) and structural changes to the application architecture.*

### A. Main Thread Blocking (TBT & INP)
*   **The Mechanism:** The site is heavily penalized by excessive JavaScript execution. Two things are happening: 
    1. Third-party ad-tech/analytics (350+ requests) are executing synchronously on the main thread.
    2. The frontend framework (React) is executing a massive hydration pass over the entire DOM, including static text that never changes.
*   **Reproduction:**
    1. Open the Homepage.
    2. In the **Performance** tab, record a reload.
    3. Observe the "Main" track. It is covered in red-flagged "Long Tasks" (tasks taking > 50ms). During this time, the page is entirely unresponsive to user taps or scrolls.
*   **The Fix:**
    1. **Ad-Tech:** Implement Partytown (or similar Web Worker proxy) to execute third-party tracking and analytics scripts off the main thread. 
    2. **Hydration:** Shift toward an Islands Architecture (e.g., Astro or Next.js React Server Components). Static text (article bodies) should be shipped as raw HTML without hydration. Only interactive components (carousels, comment sections) should be hydrated.
*   **Validation:** TBT should drop below 200ms in Lighthouse/PageSpeed Insights. In the Performance tab, Long Tasks should be sparse.

### B. DOM Size & Memory Leaks (Live Blogs)
*   **The Mechanism:** Live Blog pages continually poll for updates and append new HTML nodes to the DOM indefinitely. They do not unmount old DOM nodes that have scrolled out of the viewport. On lower-end mobile devices, this bloated DOM eventually exhausts available memory and crashes the browser tab.
*   **Reproduction:**
    1. Open a Live Blog page.
    2. Open the **Memory** tab in DevTools. Take a Heap Snapshot.
    3. Scroll down continuously for 2 minutes to load dozens of updates. Take a second Heap Snapshot. 
    4. Note the massive increase in JS heap size and detached DOM nodes.
*   **The Fix:** Implement DOM Virtualization (e.g., `react-window` or `react-virtuoso`). As the user scrolls, the application must unmount story nodes that are far outside the viewport and replace them with empty placeholder `div`s of the exact same height, keeping the active DOM node count below 1,500.
*   **Validation:** Take continuous Heap Snapshots while scrolling a live blog. The memory footprint and DOM node count should plateau, rather than growing linearly forever.

---

## 3. Domains Evaluated But Not Actionable
*The following areas were analyzed but have been intentionally excluded from this roadmap because they are either already optimized or not relevant to our business model.*

*   **Service Workers / Offline Support (PWA):** We do not currently cache full HTML articles for offline reading via Service Workers. While technically possible, AP News prioritizes breaking, real-time information. Serving stale news from a Service Worker cache risks displaying outdated information during developing events. This is intentionally skipped.
*   **Backend TTFB / Edge Delivery:** Server response times were audited and found to be excellent. Cloudflare is successfully serving cache HITs for HTML documents globally. No backend database tuning or infrastructure changes are required for this performance push.
*   **WebAssembly (WASM):** There are no heavy client-side computational tasks (like image manipulation or video encoding) that would benefit from WASM on the frontend.