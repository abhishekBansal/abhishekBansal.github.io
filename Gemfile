source "https://rubygems.org"

# Use the latest stable version of Jekyll
gem "jekyll", "~> 4.3.2"

# Plugins
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.17.0"
  gem "jekyll-sitemap", "~> 1.4.0"
  gem "jekyll-paginate", "~> 1.1.0"
  gem "jemoji", "~> 0.13.0"
  gem "jekyll-seo-tag", "~> 2.8.0"
end

# Required for Ruby 3+
gem "webrick", "~> 1.8"

# Additional dependencies
gem "csv", "~> 3.2"
gem "bigdecimal", "~> 3.1"

# Windows and JRuby does not include zoneinfo files
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
