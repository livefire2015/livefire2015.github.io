# Repository Guidelines

## Project Structure & Module Organization
Core configuration lives in `_config.yml`; keep site metadata, URLs, and theme settings there with two-space YAML indentation. Markdown entry points (`index.md`, `about.md`, `404.html`) reside at the repo root, while long-form content belongs in `_posts/` using dated filenames. `_site/` is generated output—never edit it manually nor commit temporary experiments there. Dependencies and Ruby versions are pinned in `Gemfile` and `Gemfile.lock`; update them through Bundler rather than editing by hand.

## Build, Test, and Development Commands
Run `bundle install` to sync gems before any work session. Use `bundle exec jekyll serve --livereload --port 5000` for local authoring; it mirrors production routing and reflects commit `change port to 5000`. `bundle exec jekyll build` performs a production build in `_site/`, and `bundle exec jekyll doctor` surfaces configuration regressions or unused keys.

## Coding Style & Naming Conventions
Every Markdown page starts with YAML front matter specifying `layout`, `title`, and optional metadata; keep keys in lowercase and align values with two spaces. Blog posts follow `YYYY-MM-DD-kebab-title.md` naming so Jekyll orders them correctly. Favor Markdown syntax over embedded HTML, wrap inline code with backticks, and keep paragraphs concise (≤100 characters per line) to minimize diff noise.

## Testing Guidelines
Treat the build pipeline as the test suite: run `bundle exec jekyll build` on each change set and inspect the output for warnings. For content that impacts navigation or feeds, follow up with `bundle exec jekyll serve` and spot-check the rendered pages plus `_site/feed.xml`. If you touch configuration, reload the server to confirm permalinks, tags, and excerpts still resolve.

## Commit & Pull Request Guidelines
Recent history favors short, imperative, mostly lowercase subjects (e.g., `change port to 5000`); keep summaries ≤60 characters and group related changes logically. Reference issues in the body (`Refs #42`) when applicable. Pull requests must describe the motivation, list verification commands, and include screenshots for visual tweaks. Call out any follow-up work or manual steps reviewers must perform before deployment.
