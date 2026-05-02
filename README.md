# abhishekbansal.dev — Blog

Personal blog built with [Jekyll](https://jekyllrb.com/) and hosted on GitHub Pages.

## Prerequisites

- **Ruby 3.1.4** (managed via [rbenv](https://github.com/rbenv/rbenv))
- **Bundler** — install with `gem install bundler`

### Install rbenv and Ruby 3.1.4

1. Install rbenv via Homebrew:

   ```bash
   brew install rbenv ruby-build
   ```

2. Add rbenv to your shell profile (`~/.zshrc` or `~/.bash_profile`) and restart your shell:

   ```bash
   echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
   source ~/.zshrc
   ```

3. Install Ruby 3.1.4 and set it as the local version:

   ```bash
   rbenv install 3.1.4
   rbenv local 3.1.4   # creates .ruby-version in the repo root
   ```

4. Verify:

   ```bash
   ruby -v  # should print ruby 3.1.4
   ```

5. Install Bundler:

   ```bash
   gem install bundler
   ```

## Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/abhishekBansal/abhishekBansal.github.io.git
   cd abhishekBansal.github.io
   ```

2. **Install dependencies**

   ```bash
   bundle install
   ```

## Running Locally

```bash
bundle exec jekyll serve
```

The site will be available at [http://localhost:4000](http://localhost:4000).

To enable live reload while editing:

```bash
bundle exec jekyll serve --livereload
```

## Writing a New Post

Create a new Markdown file in the `_posts/` directory following the naming convention:

```
YYYY-DD-MM-your-post-title.md
```

Add the required front matter at the top of the file:

```yaml
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
tags: [tag1, tag2]
---
```

## Building for Production

```bash
bundle exec jekyll build
```

The output is generated in the `_site/` directory.

## Project Structure
```
_config.yml       # Site configuration
_posts/           # Blog post Markdown files
_layouts/         # Page layout templates
_includes/        # Reusable HTML partials (analytics, head, etc.)
assets/           # CSS, fonts, images
index.html        # Home page
about.md          # About page
tags.html         # Tags listing page
```
