source "https://rubygems.org"

# Jekyll 4 — built and deployed via GitHub Actions (see .github/workflows/jekyll.yml),
# not the legacy GitHub Pages auto-build, so we are free of the github-pages gem's
# old Jekyll 3.x pin.
gem "jekyll", "~> 4.3"

# Site plugins.
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17"     # generates /feed.xml
  gem "jekyll-sitemap", "~> 1.4"   # generates /sitemap.xml
  gem "jemoji", "~> 0.13"          # :emoji: support
end

# webrick is no longer bundled with Ruby 3+, needed for `jekyll serve`.
gem "webrick", "~> 1.8"
