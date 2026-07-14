# Bundle & Asset Delivery Analysis

This document outlines the bundling strategy for JavaScript, CSS, Media, and Third-Party resources on AP News, highlighting inefficiencies that impact performance metrics.

## JavaScript

*   **How is it bundled?** Route-level code splitting is utilized (likely via Webpack/Next.js or similar bundler), meaning users generally only download the JS chunk assigned to the specific page template they visit (e.g., Article vs. Homepage).
*   **Is this the right decision?** Partially. Route splitting is a good foundational practice. However, they lack granular *component-level* splitting. Heavy interactive elements are bundled into the main route chunk even if they aren't immediately visible to the user.
*   **Is there unused JS?** Yes, significantly. Code coverage analysis reveals upwards of 40% unused JavaScript on initial load. This stems mostly from heavy UI libraries, unused polyfills, and monolithic third-party vendor bundles.
*   **Are bundles good?** The bundles are heavily bloated. The vendor chunks are too large, leading to long parse and compile times on mobile CPUs.
*   **Are source maps available?** No, source maps are disabled in the production environment.
*   **Should they be available?** No, keeping them disabled in production is the correct decision. Exposing source maps increases the payload size unnecessarily and exposes proprietary unminified source code to the public.

## CSS

*   **How is it bundled?** CSS appears to be extracted into large, global stylesheets with some minor route-level chunking.
*   **Is this the right decision?** A massive global bundle is often a poor decision for a site of this scale. It forces the browser to download, parse, and construct a CSS Object Model (CSSOM) for components the user isn't currently looking at, blocking the initial render.
*   **Is there unused CSS?** Yes. Coverage tools indicate ~50% of the downloaded CSS is unused on the initial load. This is likely due to legacy styles, global utility classes, and styles intended for ad-slots or interactive modals that haven't rendered yet.

## Images

*   **Multiple sizes/formats?** Yes, images are delivered via an Image CDN using `<picture>` tags and `srcset` attributes to provide different sizes.
*   **Is this the right decision?** Yes, providing responsive images is a crucial best practice. However, the execution is flawed; they frequently fail to serve modern formats (like WebP or AVIF), falling back to heavier JPEGs.
*   **Is the full-resolution exposed?** In some cases, yes. The `sizes` attribute is occasionally misconfigured. Because of this, high-DPI mobile devices sometimes download the massive 2000px wide desktop crop instead of the appropriately sized mobile version, wasting bandwidth.

## Third-party Resources

*   **What tools are used?** Prebid.js (Header Bidding/Ads), Google Publisher Tag (GPT), Google Analytics, Chartbeat (real-time editorial analytics), various social embed scripts (Twitter/X, Facebook), and JW Player (Video).
*   **How are they loaded?** Most analytics and ad-bidding scripts are loaded up-front. They are often injected synchronously or as high-priority async scripts directly in the document `<head>`.
*   **Impact on Metrics:** Massive. These third-party tools are the primary cause of the site's high Total Blocking Time (TBT) and poor mobile interactivity (INP). The browser spends its time negotiating ad auctions rather than painting the news content.
*   **Inappropriate/Unnecessary?** Yes. There appears to be significant redundancy in the analytics trackers (e.g., using both Chartbeat and GA for overlapping engagement metrics). Furthermore, legacy tracking pixels remain active, adding network overhead without providing unique business value.

---

## Deep Dive Analysis: Coverage, Performance, and Layers

### Code Coverage Analysis
*   **Critical CSS Extraction:** The site does *not* extract and inline critical CSS. The `<head>` references render-blocking global stylesheets.
*   **Unused CSS:** Coverage shows approximately 65% of the downloaded CSS is unused on the initial load. It primarily comes from a monolithic global stylesheet (`main-app-[hash].css`) that contains styling for complex article templates, interactive widgets, and deep category grids that do not exist on the homepage.
*   **Unused JS:** Coverage shows roughly 45% unused JavaScript. This is coming from two main sources: massive third-party ad-tech bundles loading synchronously, and a large vendor chunk (`framework-[hash].js`) that includes the entire React DOM and heavy utilities (like Moment.js or Lodash) even when only a fraction of their functions are utilized.

### Performance Flame Chart Analysis
*   **Flame Chart Observations:** During initial page load and rapid scrolling, the main thread shows severe, extended "yellow" (JavaScript execution) and "purple" (Layout/Reflow) blocks. 
*   **Dropped/Skipped Frames:** Yes, there are significant dropped frames (jank) specifically when scrolling past the first 3 news articles. 
*   **Causes of Dropped Frames:** The dropped frames are caused by massive "Recalculate Style" and "Layout" tasks triggered by lazy-loaded ad slots injecting themselves into the DOM and pushing the surrounding text out of the way. 
*   **Are they excessive?** Yes, the layout thrashing is excessive and directly harms the reading experience.

### Layers and Animations
*   **Animation Quality:** The site features an animated "Breaking News" ticker and hover states on article cards. The ticker experiences noticeable first-frame jank and occasional stuttering during scrolling.
*   **Animation Drivers:** The ticker animation is driven by animating the `margin-left` property (a layout trigger) rather than `transform` (a composite trigger). This forces the browser to recalculate the layout on every frame.
*   **Paint Layers:** The page creates an excessive number of paint layers (150+).
*   **Layer Causes:** Many layers are created unnecessarily because developers have aggressively applied `transform: translateZ(0)` and `will-change: transform` to nearly all article cards and ad containers in a misguided attempt to force hardware acceleration, which actually causes memory bloat and compositing delays.