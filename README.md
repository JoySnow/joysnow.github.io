# JoySnow's Blog

[![GitHub Pages](https://img.shields.io/github/deployments/joy/joysnow.github.io/github-pages)](https://joysnow.github.io)

Joy's personal blog, built with [Jekyll](https://jekyllrb.com/) and using the [**NexT**](https://github.com/Simpleyyt/jekyll-theme-next) theme style.

## Theme

This site uses the [jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next) theme, a Jekyll port of the popular [NexT](https://github.com/theme-next/hexo-theme-next) theme originally built for Hexo.

> **Note:** This repository is **not a fork** of [jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next). The theme files were manually imported and customized to fit this blog's needs. The original fork ancestry is from [barryclark/jekyll-now](https://github.com/barryclark/jekyll-now), but the codebase has since diverged significantly.

### Syncing with jekyll-theme-next

Since this repo is not a fork of jekyll-theme-next, you can sync upstream changes by adding it as an additional remote:

```bash
# Add the upstream repo as a remote (one-time setup)
git remote add upstream https://github.com/Simpleyyt/jekyll-theme-next.git

# Fetch upstream changes
git fetch upstream

# Cherry-pick or merge specific commits/tags
git checkout master
git merge upstream/master --allow-unrelated-histories
```

After merging, resolve any conflicts between the upstream changes and your local customizations. You can also cherry-pick individual commits:

```bash
git cherry-pick <commit-hash>
```

## Local Development

```bash
# Install dependencies
bundle install

# Start the Jekyll dev server
bundle exec jekyll serve

# Build only
bundle exec jekyll build
```

## License

This project is open source under the [MIT License](LICENSE).
