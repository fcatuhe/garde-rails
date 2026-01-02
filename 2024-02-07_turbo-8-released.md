# Turbo 8 released

**Author:** Alberto Fernández-Capel, Programmer
**Published:** February 7, 2024
**Source:** <https://dev.37signals.com/turbo-8-released/>

---

![Turbo 8](https://dev.37signals.com/assets/images/turbo-8-released/turbo-8.webp)

We’re excited to announce the release of Turbo v8, a major update to the Turbo front-end framework. This release introduces a suite of innovative features designed to enhance web development and user experiences across the board.

Here are the key highlights of Turbo v8:

---

## [Morphing for smooth page refreshes](https://turbo.hotwired.dev/handbook/page_refreshes)

This new technique enables us to refresh pages selectively, replacing only the HTML elements that need to change, and ensures smoother updates. If you’re curious, Jorge Manrubia has written a [fantastic article](https://dev.37signals.com/page-refreshes-with-morphing-demo/) showcasing the power of this feature.

In our apps, we’re already leveraging morphing inside Basecamp, to refresh the [Card Table](https://basecamp.com/features/card-table) and throughout the [new HEY Calendar](https://www.hey.com/calendar/).

---

## [View Transitions](https://turbo.hotwired.dev/handbook/drive#view-transitions) with the [View Transition API](https://developer.mozilla.org/en-US/docs/Web/API/View_Transitions_API)

Turbo v8 introduces graceful animations between pages, enriching the user experience by providing visual continuity as users navigate through an application. This feature depends on browser support, and it’s limited to Chrome for now, but we’re already giving it good use in ONCE #1, [Campfire](https://once.com/campfire).

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/turbo-8-released/view-transition.mp4)

---

## [InstantClick](https://turbo.hotwired.dev/handbook/drive#instantclick)

We’ve further reduced the time it takes to load a new page. Turbo v8 pre-loads links before they’re even clicked, cutting off precious milliseconds and making our applications feel instantaneous.

In our tests, we’ve seen a very noticeable improvement in the typical click navigation. For example, with a slow connection, a page may take ~1.4 seconds to load, which feels sluggish.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/turbo-8-released/instantclick-off.mp4)

With InstantClick, however, the same page loads in ~380ms. That’s more than a 1 second improvement and a total game changer for the user experience!

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/turbo-8-released/instantclick-on.mp4)

This feature is now live in Basecamp and Campfire.

---

## [Migration away from TypeScript](https://world.hey.com/dhh/turbo-8-is-dropping-typescript-70165c01)

We’ve embraced the dynamic nature of JavaScript for Turbo’s development, moving away from TypeScript. Despite the initial [controversy within the TypeScript community](https://world.hey.com/dhh/open-source-hooliganism-and-the-typescript-meltdown-a474bfda), the Turbo project is now thriving more than ever with a stable and ever growing community of contributors.

In addition to these major features, Turbo v8 includes numerous smaller improvements and bug fixes. By the numbers, we’ve merged 125 pull requests and closed 102 issues across the Turbo, turbo-rails, and documentation repositories. You can find the full list of changes in the release notes for [Turbo](https://github.com/hotwired/turbo/releases/tag/v8.0.0) and [turbo-rails](https://github.com/hotwired/turbo-rails/releases/tag/v2.0.0).

We’re excited for you to experience the improved capabilities and enhancements of Turbo v8. Dive in and see how Turbo v8 can elevate your web development projects and user experiences.

Enjoy! ✌️
