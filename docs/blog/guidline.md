---
sidebar_position: 1
---

# Deploy Docusaurus into Github page

1. Install nodejs (https://nodejs.org/en/download)
2. Open terminal and type `node -v` to check node version
3. Create empty repo on Github
4. Create empty folder and run `npx create-docusaurus@latest blog classic`
5. Run `npm run start` to check
6. Edit file `docusaurus.config.js` with following content
   - `url`: https://your-github-name.github.io
   - `baseUrl`: `/{repo-name}/` (ex: `/test-blog/`)
   - `organizationName`: your GitHub user name
   - `projectName`: repo name created in <u>step 3</u>
   - `trailingSlash`: false
7. Create `.github/workflow` folders in project
8. Create `deploy.yml` file in `workflow` with content

```yaml title="deploy.yml"
name: Deploy to GitHub Pages

on:
  push:
    branches:
      - main
    # Review gh actions docs if you want to further define triggers, paths, etc
    # https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#on

jobs:
  build:
    name: Build Docusaurus
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci
      - name: Build website
        run: npm run build

      - name: Upload Build Artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: build

  deploy:
    name: Deploy to GitHub Pages
    needs: build

    # Grant GITHUB_TOKEN the permissions required to make a Pages deployment
    permissions:
      pages: write # to deploy to Pages
      id-token: write # to verify the deployment originates from an appropriate source

    # Deploy to the github-pages environment
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

9. Create `test-deploy.yml` file in `workflow` with content

```yaml title="test-deploy.yml"
name: Test deployment

on:
  pull_request:
    branches:
      - main
    # Review gh actions docs if you want to further define triggers, paths, etc
    # https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#on

jobs:
  test-deploy:
    name: Test deployment
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 18
          cache: npm

      - name: Install dependencies
        run: npm ci
      - name: Test build website
        run: npm run build
```

10. Push to github
11. Go to repo setting on github
    ![github-page](./img/github-page.png)

:::tip
Prompt to create icon

> sticker design, icon, flat color, 2 color only black and white, coder, transparent background

:::
