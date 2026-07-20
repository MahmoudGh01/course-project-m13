# Course Project: Performance Audit Report

*   **[Executive Summary](executive-summary.md)** (For C-level and Directors)
*   **[Baseline Findings](baseline-findings.md)**
*   **[Bundle Analysis](bundle-analysis.md)**
*   **[Prioritization](prioritization.md)**

## 1. Target Website
*   **Name:** AP News
*   **URL:** [https://apnews.com/](https://apnews.com/)

## 2. Why This is a Good Candidate
AP News is an excellent candidate for a performance audit because it sits at the intersection of high global traffic and heavy, dynamic content. As a leading news wire, the site must constantly load high-resolution photojournalism, auto-playing video embeds, and live tickers. Balancing this rich media with the necessity for near-instant load times (crucial for breaking news and mobile SEO) presents significant performance engineering challenges. An audit will reveal how they manage ad-tech scripts, image optimization, and Cumulative Layout Shift (CLS) on a highly mutable layout.

## 3. Main PageSpeed Insights Scores & Testing Methodology
*(Scores retrieved from PageSpeed Insights for the Homepage)*
*   **Mobile Performance Score:** 22/100
*   **Desktop Performance Score:** 54/100

**Testing Methodology:** 
This audit places a primary emphasis on the **Mobile** user experience. To ensure accurate representation of real-world mobile browsing, all baseline metrics and findings were measured using the following throttling profile:
*   **Device Simulation:** Mobile (Moto G Power)
*   **Network Throttling:** Slow 4G / Fast 3G
*   **CPU Throttling:** 4x slowdown

## 4. Focus Pages for Audit
This audit will focus on the following core page templates to evaluate different performance bottlenecks:

1. **The Homepage** (`https://apnews.com/`)
   * *Why to include:* The primary entry point. It is highly dynamic, aggregating the latest stories with hero image carousels and sticky headers. We will audit Largest Contentful Paint (LCP) and how third-party ad scripts affect the initial load.
2. **Standard Article Page** (`https://apnews.com/article/example-news-story`)
   * *Why to include:* This represents the bulk of user engagement and reading time. It is vital to measure text rendering, web font loading, and Cumulative Layout Shift (CLS) as ads or related-article widgets pop in during reading.
3. **Category "Hub" Page** (`https://apnews.com/hub/politics`)
   * *Why to include:* A dense, grid-based layout. This will test their image lazy-loading implementation and how quickly the server responds with categorized database queries compared to a cached static page.
4. **Video Content Page** (`https://apnews.com/video`)
   * *Why to include:* Video players bring heavy JavaScript payloads. Auditing this will highlight their script execution time, Total Blocking Time (TBT), and how efficiently the streaming media is delivered without blocking the main thread.
5. **Live Blog / Breaking Event Page** (`https://apnews.com/live-updates/example-event`)
   * *Why to include:* These pages constantly poll for new data or rely on WebSockets to push updates. This tests the site's ongoing runtime performance, client-side rendering capabilities, and memory management over a long session.

---
*Prepared for M13 Course Project*