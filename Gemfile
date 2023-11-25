source "https://rubygems.org"

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Add webrick as a dependency for Ruby 3.0, as it's no longer a bundled gem
gem "webrick", "~> 1.7"


gem 'jekyll', '~> 4.3', '>= 4.3.2'
gem 'jekyll-feed', '~> 0.17.0'
gem 'jekyll-gist', '~> 1.5'
gem 'jekyll-paginate-v2', '~> 3.0'
gem 'jekyll-seo-tag', '~> 2.8'
gem 'jekyll-sitemap', '~> 1.4'
gem 'faraday-retry'
gem 'jekyll-toc'
gem 'coderay'
gem 'kramdown-syntax-coderay'