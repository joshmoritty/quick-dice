# Quick Dice

A lightweight Vue 3 web app for rolling dice by typing formulas.

## Features

- Supports standard dice notation:
  `1d20 + 5`, `2d6 + 1d8 - 3`, etc.
- Instant results with breakdown display
- Save commonly-used formulas, optionally with names

## Example Usage

Type any formula and press **Enter** or click **Roll**:

```
1d20 + 2d4 + 3
```

Produces something like:

```
Result: 17
Breakdown: [12] + [3, 2] + 3
```

## Development

Install dependencies:

```bash
pnpm install
```

Run locally:

```bash
pnpm dev
```

Build for deployment:

```bash
pnpm build
```

Preview production build:

```bash
pnpm preview
```

## Deployment

This project is configured for GitHub Pages.
After building, deploy the `dist` folder using:

```bash
pnpm gh-pages -d dist
```
