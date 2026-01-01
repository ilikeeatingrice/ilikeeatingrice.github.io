# Project Overview

This is a personal website and blog for "Tiancheng", hosted on GitHub Pages. It is built using [Jekyll](https://jekyllrb.com/), a static site generator. The site design appears to be based on the "Kiko" theme, heavily customized with local layouts and styles.

## Key Technologies
*   **Jekyll:** Static site generator (v3.2.1 specified in `Gemfile`).
*   **Minima:** Listed as a theme gem, but the site uses custom layouts in `_layouts/` and a custom stylesheet `style.scss`.
*   **Jekyll SEO Tag:** Plugin for better SEO.
*   **SCSS:** CSS pre-processing.

## Architecture
*   **Configuration:** `_config.yml` controls site settings (URL, author, navigation, markdown parser).
*   **Content:**
    *   **Posts:** Blog posts are located in `_posts/` and follow the standard naming convention `YYYY-MM-DD-title.md`.
    *   **Pages:** Static pages like `index.html` and `about.html` live in the root directory.
*   **Layouts:** `_layouts/` contains the HTML templates (`default.html`, `post.html`, `page.html`).
*   **Styling:** `style.scss` is the main stylesheet. It contains custom CSS and is processed by Jekyll into `style.css`. Note that there is also a `css/main.scss` which appears to import `minima`, but the default layout explicitly links to `/style.css` (compiled from `style.scss`).

# Building and Running

## Prerequisites
*   Ruby
*   Bundler

## Commands

1.  **Install Dependencies:**
    ```bash
    bundle install
    ```

2.  **Run Locally:**
    Start a local development server with live reloading:
    ```bash
    bundle exec jekyll serve
    ```
    The site will be available at `http://localhost:4000` (or the URL displayed in the terminal).

# Development Conventions

*   **New Posts:** Create a new markdown file in `_posts/` with the filename format `YEAR-MONTH-DAY-title.md`. Ensure it has the correct YAML front matter (layout, title, etc.).
*   **Styles:** Modify `style.scss` for global style changes. Avoid editing `style.css` directly as it is a generated file.
*   **Layouts:** HTML structure changes should be made in the files within `_layouts/`. `default.html` is the base template wrapping other layouts.
*   **Static Assets:** Images and other static files are stored in `static/`.
