# Lexxy: A new rich text editor for Rails

**Author:** Jorge Manrubia, Principal Programmer, Product
**Published:** September 4, 2025
**Source:** <https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/>

*A better Action Text.*

---

Today, we are introducing [Lexxy](https://github.com/basecamp/lexxy/), a new rich text editor for Action Text. It’s based on [Lexical](https://lexical.dev/) — Meta’s text editing framework — and it brings a much better text editing experience to Rails:
- Good HTML semantics. Paragraphs are real `<p>` tags, as they should be.
- Markdown support: shortcuts, auto-formatting on paste.
- Real-time code syntax highlighting.
- Create links by pasting URLs on selected text.
- Configurable prompts. Support for mentions and other interactive prompts with multiple loading and filtering strategies.
- Preview attachments like PDFs and Videos in the editor.
- Works seamlessly with Action Text and Active Storage.

[📺 Watch video](https://videos.37signals.com/dev/assets/videos/announcing-lexxy-a-new-rich-text-editor-for-rails/lexxy.mp4)

We created Lexxy because Trix was falling short of expectations in certain areas, and we encountered technical barriers when attempting to offer the experience we wanted. Lexxy comes with a bunch of juicy improvements, but more than that, we now have a fantastic foundation to build on top of.

Lexxy will also bring a great improvement to [Action Text](https://guides.rubyonrails.org/action_text_overview.html): we will let you configure the editor in Action Text just like you configure the database in Active Record. This will open the door to integrating other editors in Rails.

Text editing is central to our products. We believe Lexxy will let us deliver the editing experience we want. We are launching an early beta today, [give it a try](https://github.com/basecamp/lexxy/)!
