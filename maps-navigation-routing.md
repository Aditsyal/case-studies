# Case Study: 60 FPS Navigation — Optimizing Real-Time EV Routing for High Performance & Low Battery Drain

> **Disclosure:** This case study focuses on frontend architecture, UI/UX challenges, and real-time state management. It omits proprietary backend logic, API keys, and exact internal metrics.

---

## TL;DR

I led the frontend engineering for an in-app EV navigation system, transforming a stuttering map experience into a high-performance driving companion.

**The Challenge:** Managing dynamic multi-stop routes and real-time rerouting on mid-range devices without sacrificing battery life or 60fps responsiveness.

**The Win:**

- **Performance:** Achieved consistent 60fps and <200ms route updates.
- **Efficiency:** Slashed background battery drain from ~8% to <2% per hour.
- **UX:** Eliminated "navigation whiplash" with smooth, cross-faded rerouting transitions.
- **Reliability:** Reduced navigation-related support tickets by moving heavy processing to Dart isolates.

---

## The Context: Building for the Driver

The product is an EV management app. While the backend computes complex multi-stop routes, the **frontend** is the driver's primary interface. My responsibility was to ensure that map rendering, turn-by-turn instructions, and rerouting logic remained fluid and responsive, even under heavy load or poor network conditions.

I owned the navigation flow end-to-end: from UI/UX design patterns to state management architecture and map rendering optimization.

---

## The Challenge: Eliminating "Navigation Whiplash"

Navigation is high-stakes: a 500ms stutter or a jarring map jump can cause a driver to miss a turn. We faced several friction points that compromised safety and performance:

### Core Friction Points

1. **Jittery Route Updates** — Re-rendering 1000+ point polylines caused visible frame drops on mid-range devices.
2. **Disorienting Reroutes** — Instant map redraws caused "context loss," where the driver momentarily lost their sense of direction.
3. **Main-Thread Bottlenecks** — Processing turn-by-turn instructions every 2 seconds made the UI feel sluggish to touch.
4. **State Ambiguity** — Fragmented state logic led to race conditions (e.g., showing the "Arrived" UI while still 200m away).

### The Baseline (Pre-Optimization)

- **Performance:** Rerouting dropped frames to ~45fps.
- **Responsiveness:** 150ms tap latency during route processing.
- **Battery:** 5–8% hourly drain (unacceptable for long trips).
- **Latency:** Large routes took >500ms to render.

---

## Goals and Constraints

### Goals

- **Smooth UX:** Eliminate jarring transitions during rerouting.
- **Main-Thread Freedom:** Keep instructions updating without blocking user input.
- **Zero Jitter:** Maintain a locked 60fps on mid-range hardware.
- **Efficiency:** Limit background tracking to <2% battery drain per hour.

### Constraints

- **Device Fragmentation:** Must perform on Android 8+ and iOS 12+ devices with limited RAM.
- **Network Gaps:** Gracefully handle 5–30s delays in backend route updates.
- **Driver Context:** Touch targets must be large and text readable at a glance.

---

## Approach: Engineering for Performance

### 1. Polyline Rendering: Choosing SDK Power over Custom Logic

**The Insight:** High-performance map rendering is a solved problem within the SDK. Our goal was to feed it data efficiently rather than reinventing the wheel.

| Option                               | Pros                                                | Cons                              |
| ------------------------------------ | --------------------------------------------------- | --------------------------------- |
| Draw full polyline every frame       | Simplest                                            | Frame drops on large routes       |
| Draw only visible segment + buffer   | Better performance                                  | More complex; reinventing culling |
| **Use map SDK polyline rendering** ✓ | Native optimization; benefits from platform updates | Minimal                           |

**The Decision:** We used the SDK’s native polyline API but optimized the _data delivery_. Instead of full redraws, we decoded and diffed segments on a background thread (Dart Isolate), supplying only the delta to the SDK. This reduced route render time from **500ms to <50ms**.

---

### 2. Rerouting UX: Speed vs. Spatial Awareness

**The Insight:** While "fast" is usually better, instant updates in navigation can be disorienting. Drivers need a bridge between the old path and the new one.

| Option                             | Pros                                  | Cons                   |
| ---------------------------------- | ------------------------------------- | ---------------------- |
| Instant redraw                     | Fast                                  | Disorienting "snap"    |
| Fade transition                    | Smooth                                | Adds perceived latency |
| **Notification + silent update** ✓ | Signals change without breaking focus | Brief data-to-UI lag   |

**The Decision:** We implemented a "Signal-then-Shift" pattern. When a reroute is detected, we show a brief "Recalculating..." status. We then apply a 300ms cross-fade to the new polyline, giving the driver's eyes time to adjust to the new path without breaking their flow.

---

### 3. Threading: Protecting the Main Thread

**The Issue:** Instruction updates (nearest turn, distance formatting) arrive every ~2s. Processing this on the main thread made map pan/zoom feel "heavy."

**The Approach:** We utilized Dart Isolates to handle all non-UI logic. A background isolate processes raw coordinates and formats the instruction strings; the main thread only handles the final UI update.

**The Outcome:** Tap detection latency dropped from **150ms to ~80ms**, keeping the map responsive even during complex route recalculations.

---

### 4. State Management: The Navigation Lifecycle

**The Issue:** Navigation involves complex states (Planning → Navigating → Rerouting → Arrived). Ad-hoc boolean flags led to race conditions and inconsistent UI states.

**The Approach:** We implemented an explicit state machine using a `NavigationViewModel` (Provider). All UI components derive their state from a single `NavigationState` enum. Transitions are centralized, ensuring the UI never enters an "impossible" state.

---

### 5. Battery Efficiency: Native Location Services

**The Issue:** Continuous location polling in pure Flutter was draining 5–8% battery per hour.

**The Approach:** We bypassed generic Flutter packages in favor of custom platform channels to native iOS (CoreLocation) and Android (FusedLocationProvider) services. We implemented distance-based throttling (>50m) and automated "coarse" mode when the app is backgrounded.

**The Outcome:** Battery drain dropped to **~1.5% per hour**, making the app viable for all-day road trips.

---

## The Outcome: High Performance, High Trust

| Metric                | Before         | After             |
| --------------------- | -------------- | ----------------- |
| **Route Rendering**   | ~500ms         | **<50ms**         |
| **UI Responsiveness** | ~150ms latency | **~80ms latency** |
| **Frame Rate**        | ~45fps dips    | **Steady 60fps**  |
| **Memory Footprint**  | ~45MB          | **~15MB**         |
| **Battery Impact**    | 5–8% / hour    | **<2% / hour**    |

### Business & User Impact

- **Trust:** "Map is lagging" feedback decreased by **85%**.
- **Safety:** The cross-fade reroute pattern reduced user reports of "route confusion" during navigation.
- **Stability:** Moving heavy logic to isolates resulted in a **<0.1% crash rate** for the navigation module.

---

## Engineering Principles for Mission-Critical UI

1. **Don't Fight the SDK:** Leverage native platform optimizations for heavy rendering (LOD/Culling) rather than building custom abstractions.
2. **Perceived Performance is King:** In a driving context, a smooth 300ms transition is more valuable than a "perfectly fast" instant snap.
3. **Isolate Heavy Work:** Treat the main thread as sacred. Any data processing (decoding, formatting) belongs in a background isolate.
4. **State Machines are Mandatory:** Replacing boolean flags with a formal `NavigationState` enum eliminated 90% of UI race conditions.
5. **Native for Battery:** For background location, skip the wrappers. Direct platform channels to native services are the only way to meet strict battery budgets.

---

**Tech stack:** Flutter, Provider, Google Navigation & Maps SDK, Dart Isolates, Native Location (iOS/Android)  
**Focus:** Performance Engineering, UI/UX, State Management  
**Date:** February 2026
