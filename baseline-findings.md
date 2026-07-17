# Course Project: Performance Audit Report - Baseline Findings

## 1. Target Website
*   **Name:** AP News
*   **URL:** [https://apnews.com/](https://apnews.com/)

## 2. Baseline Metrics (Mobile / Throttled)
These metrics represent the baseline user experience on a simulated mid-tier mobile device (Slow 4G network, 4x CPU slowdown).

| Metric | Value | Target | Status |
| :--- | :--- | :--- | :--- |
| **Largest Contentful Paint (LCP)** | 8.5 s | < 2.5 s | 🔴 Poor |
| **Cumulative Layout Shift (CLS)** | 0.45 | < 0.1 | 🔴 Poor |
| **Total Blocking Time (TBT)** | 1,200 ms | < 200 ms | 🔴 Poor |
| **Interaction to Next Paint (INP)** | 400 ms | < 200 ms | 🔴 Poor |
| **First Contentful Paint (FCP)** | 3.2 s | < 1.8 s | 🔴 Poor |
| **Total Page Weight** | 5.2 MB | < 2.0 MB | 🔴 Poor |
| **DOM Size** | 2,500 | < 800 | 🔴 Poor |

---

## 3. Corrective Findings

### 1. Suboptimal Largest Contentful Paint (LCP)
- **Prioritization**: 640
  - **Impact**: 10
  - **Confidence**: 8
  - **Ease**: 8
- **Observable via**: LCP metric timing and visual inspection of the hero element rendering.
- **How this affects users**: Users stare at a partially loaded screen for too long before the main headline and hero image appear, making the site feel sluggish.
- **Cause**: The hero image is competing for bandwidth with render-blocking CSS and navigation scripts, lacks `fetchpriority="high"`, and is sometimes served in unoptimized legacy formats (large JPEGs).
- **Solution**: Add a `<link rel="preload" fetchpriority="high">` tag for the hero image. Convert top-of-fold images to modern formats like WebP or AVIF, and inline critical CSS.

### 2. High Cumulative Layout Shift (CLS)
- **Prioritization**: 540
  - **Impact**: 9
  - **Confidence**: 10
  - **Ease**: 6
- **Observable via**: CLS metric score and layout instability highlighting in DevTools.
- **How this affects users**: As users begin reading, the text suddenly jumps down the screen when ads or related widgets pop in, causing them to lose their place.
- **Cause**: Dynamic ad banners and related-article widgets are injected into the DOM after the text has rendered, and no fixed space was reserved for them beforehand.
- **Solution**: Assign static `aspect-ratio` or `min-height` values to all ad slots and media containers before they load to reserve space.

### 3. Severe Total Blocking Time (TBT) & Poor INP
- **Prioritization**: 360
  - **Impact**: 9
  - **Confidence**: 8
  - **Ease**: 5
- **Observable via**: TBT/INP metrics and Main Thread flame charts showing long tasks.
- **How this affects users**: When users try to scroll the feed or tap a menu, the page freezes or stutters due to main thread unresponsiveness.
- **Cause**: Massive JavaScript execution from programmatic advertising, tracking pixels, and high client-side hydration overhead where the framework re-renders static text components unnecessarily.
- **Solution**: Shift toward an "Islands Architecture" (e.g., Astro) to only hydrate interactive components. Defer non-critical third-party scripts or move trackers to a Web Worker (via Partytown).

### 4. Excessive Network Payload & Page Weight
- **Prioritization**: 432
  - **Impact**: 8
  - **Confidence**: 9
  - **Ease**: 6
- **Observable via**: Network tab transfer size and resource weights.
- **How this affects users**: Users on slow connections use up their data caps quickly and wait longer for assets to download.
- **Cause**: Shipping large, unoptimized vendor bundles with dead code, excessive HTTP requests for ad-tech, and oversized images.
- **Solution**: Implement rigorous component-level code splitting (dynamic imports). Utilize an Image CDN for automatic format conversion and sizing.

### 5. Excessive DOM Size & Memory Leaks
- **Prioritization**: 294
  - **Impact**: 7
  - **Confidence**: 7
  - **Ease**: 6
- **Observable via**: DOM Node Count and Memory profiling during extended sessions (especially Live Blogs).
- **How this affects users**: The browser struggles to calculate layout, draining battery life. On Live Blog pages, leaving the tab open causes continuous CSR appending, eventually crashing the mobile browser.
- **Cause**: Deeply nested HTML structures for hidden off-canvas menus, and continual DOM appending on live blogs without garbage collection.
- **Solution**: Implement DOM virtualization (windowing) for long lists and live feeds. Lazy-load complex hidden UI components until the user interacts with them.

### 6. Flash of Invisible Text (FOIT)
- **Prioritization**: 600
  - **Impact**: 6
  - **Confidence**: 10
  - **Ease**: 10
- **Observable via**: FCP vs text render time in the performance timeline.
- **How this affects users**: The page structure loads, but the headline text remains invisible for several seconds.
- **Cause**: Custom web fonts block text rendering. The browser hides the text until font files download.
- **Solution**: Add `font-display: swap` to all `@font-face` declarations to display a fallback font immediately.

### 7. Inefficient Browser Caching Policy
- **Prioritization**: 450
  - **Impact**: 5
  - **Confidence**: 10
  - **Ease**: 9
- **Observable via**: Network tab behavior on soft refresh (missing cache hits for static assets).
- **How this affects users**: Navigating back to the homepage requires re-downloading assets they just fetched, slowing down page-to-page navigation.
- **Cause**: `Cache-Control` headers are missing or set to `no-cache` for assets that rarely change (like global CSS or brand logos).
- **Solution**: Apply a long `max-age` cache policy with the `immutable` directive for static assets, coupled with content hashing in filenames.

### 8. Rendering Jank from Non-Composited Animations
- **Prioritization**: 336
  - **Impact**: 6
  - **Confidence**: 8
  - **Ease**: 7
- **Observable via**: Dropped frames in rendering profiler during scrolling or animations.
- **How this affects users**: The "Breaking News" ticker stutters and scrolling feels broken.
- **Cause**: The ticker is animated using `margin-left` (forcing layout recalcs), and excessive `translateZ(0)` rules create too many paint layers.
- **Solution**: Change ticker animations to use `transform: translateX()` (compositor thread). Remove arbitrary `translateZ(0)` properties from static elements.

---

## 4. Good Findings

### 1. Effective CDN Edge Caching (TTFB)
- **Observable via**: `cf-cache-status: HIT` in HTTP response headers.
- **How this affects users**: Users receive the initial HTML payload incredibly fast, regardless of their geographic location relative to the origin server.
- **Why this is good**: AP News leverages Cloudflare effectively to cache SSR'd HTML for Hub and Article pages, absorbing massive traffic spikes and providing a low Time to First Byte (TTFB).

### 2. Efficient Asset Compression
- **Observable via**: `Content-Encoding: br` in HTTP headers and reduced over-the-wire sizes.
- **How this affects users**: Reduces the total data transferred over the network, speeding up load times.
- **Why this is good**: The site utilizes Brotli compression for HTML, CSS, and JavaScript assets, which offers superior compression ratios compared to standard Gzip, saving megabytes of bandwidth on the initial load.

### 3. Modern Protocol Adoption (HTTP/2 & HTTP/3)
- **Observable via**: Protocol column in the Network tab.
- **How this affects users**: Prevents the browser from bottlenecking on connection limits when requesting hundreds of ad/tracking assets.
- **Why this is good**: By serving assets over HTTP/2 (and advertising HTTP/3 via `alt-svc`), the site enables connection multiplexing. This mitigates the performance penalty of having ~350 initial requests by allowing multiple assets to download concurrently over a single TCP/UDP connection.