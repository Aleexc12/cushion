# Embeds are Core, Viewers are plugins

Anything rendered inside a note is Core: KaTeX math and image, PDF and note embeds.
A Viewer, which opens a file in its own tab, is a plugin, even for basic file types like PDF and images.
The plugin API therefore never exposes CodeMirror widgets, the hardest part of the editor to keep stable, while Viewers already have a contribution point in the view registry.

## Considered Options

- Make embeds plugins too: rejected because a plain markdown note looks broken without them, and widget code inside CodeMirror is costly to rewrite behind an API.
- Keep the PDF and image Viewers in Core: rejected because they ship as Built-in plugins anyway, and routing them through the plugin API means Cushion uses its own API from day one.

## Consequences

- The PDF embed and the PDF Viewer both need pdfjs, which Core already ships for the embed.
- Monaco becomes the Built-in fallback Viewer for text files with no other Viewer, so Viewers need a priority or default concept.
- Disabling a Viewer leaves its embeds working.
