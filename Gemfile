source "https://rubygems.org"

# Bespoke Kami theme — no remote/gem theme. Layouts/includes/sass live in-repo.
# GitHub Pages builds this with its own pinned toolchain; this Gemfile is for
# local `bundle exec jekyll serve` only.

gem "jekyll", "~> 3.9"

group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.15"
  gem "jekyll-paginate", "~> 1.1"
end

gem "kramdown-parser-gfm"

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]
gem "wdm", "~> 0.1.0" if Gem.win_platform?
