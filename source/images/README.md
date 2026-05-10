# Images

This source tree intentionally does not duplicate the existing binary image files from the generated GitHub Pages output.

The current published site already keeps those assets in the repository root under `images/` and `uploads/`. During `npm run build`, `scripts/copy-existing-assets.js` copies those existing root assets into `public/` after Hexo finishes generating pages.
