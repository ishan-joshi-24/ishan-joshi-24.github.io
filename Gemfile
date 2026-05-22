source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "minima", "~> 2.5"
gem "jekyll-feed", "~> 0.12"
gem "jekyll-seo-tag", "~> 2.8"

# Pin to a working version on GitHub Actions runners
# (sass-embedded 1.100.0 has a broken native build)
gem "sass-embedded", "~> 1.97.0"

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated gem.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

