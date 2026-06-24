# Aurora MySQL Internals: A Production Engineer's Deep Guide

This repository contains the source for an mdBook covering Aurora MySQL internals, from storage architecture and the buffer pool through replication, query optimization, and production operations.

## Build locally

You need [mdBook](https://github.com/rust-lang/mdBook) installed:

```bash
# With Homebrew on macOS
brew install mdbook

# Or via cargo
 cargo install mdbook --version 0.4.40
```

Then build the book:

```bash
mdbook build
```

The generated site is written to the `book/` directory.

## Serve locally

```bash
mdbook serve
```

Then open <http://localhost:3000>.

## Deploy

The book is automatically built and deployed to GitHub Pages on every push to `main` via the workflow in `.github/workflows/deploy.yml`.

To enable Pages for this repository:

1. Go to **Settings → Pages**.
2. Under **Build and deployment**, set **Source** to **GitHub Actions**.
3. Push to `main` or trigger the workflow manually.

## Structure

- `book.toml` — mdBook configuration.
- `src/SUMMARY.md` — table of contents.
- `src/ch*.md` — chapter content.
- `src/part*.md` — short part introductions.

## License

See the repository's license file for details.
