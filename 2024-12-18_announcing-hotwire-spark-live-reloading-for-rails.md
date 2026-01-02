# Announcing Hotwire Spark: live reloading for Rails applications

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** December 18, 2024
**Source:** <https://dev.37signals.com/announcing-hotwire-spark-live-reloading-for-rails/>

*Improve your feedback loop with smooth automatic page updates.*

---

Today, we are releasing [Hotwire Spark](https://github.com/hotwired/spark), a live-reloading system for Rails Applications.

Reloading the browser automatically on source changes is a problem that has been well-solved for a long time. Here, we wanted to put an accent on smoothness. If the reload operation is very noticeable, the feedback loop is similar to just reloading the page yourself. But if it’s smooth enough — if you only perceive the intended change — the feedback loop becomes terrific.

To use, just install the gem in development:

```
group :development do
  gem "hotwire-spark"
end
```

It will update the current page on three types of change: HTML content, CSS, and Stimulus controllers. How do we achieve that desired smoothness with each?
- For **HTML content**, it morphs the `<body>` of the page into the new `<body>`. Also, it disconnects and reconnects all the Stimulus controllers on the page.
- For **CSS**, it reloads the changed stylesheet.
- For **Stimulus controllers**, it fetches the changed controller, replaces its module in Stimulus, and reconnects all the controllers.

We designed Hotwire Spark to shine with the [nobuild approach we use and recommend](https://dev.37signals.com/a-vanilla-rails-stack-is-plenty/). Serving CSS and JS assets as standalone files is ideal when you want to fetch and update only what has changed. There is no need to use bundling or any tooling. Hot Module Replacement for Stimulus controllers without any frontend building tool is pretty cool!

2024 has been a very special year for Rails. We’re thrilled to share Hotwire Spark before the year wraps up.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/announcing-hotwire-spark-live-reloading-for-rails/announcing-hotwire-spark.mp4)

Wishing you all a joyful holiday season and a fantastic start to 2025.
