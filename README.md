# FlightLab

Source for [jasonscottsf.github.io](https://jasonscottsf.github.io) - a technical blog by Jason Scott on networking, edge AI, and autonomous flight, documenting an AI-piloted drone built from the ground up.

Built with [Astro](https://astro.build), deployed to GitHub Pages via GitHub Actions on every push to `main`.

## Local development

```sh
npm install
npm run dev      # local dev server at localhost:4321
npm run build    # production build to ./dist/
npm run preview  # preview the production build locally
```

## Structure

- `src/content/blog/` - posts (Markdown/MDX)
- `src/pages/` - home, blog index, about
- `src/components/`, `src/layouts/` - site chrome
- `.github/workflows/deploy.yml` - build + deploy to Pages

Post frontmatter: `title`, `description`, `pubDate`, optional `updatedDate` and `heroImage`.
