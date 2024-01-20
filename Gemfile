source "https://rubygems.org"

# Specify Jekyll version
gem "github-pages", group: :jekyll_plugins

# Use Jekyll native version (uncomment the line below if needed)
# gem "jekyll"

# Specify wdm version based on the Windows platform
gem 'wdm', Gem.win_platform? ? '~> 0.1.1' : '~> 0.1.0'

# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-feed"
  gem 'jekyll-sitemap'
  gem 'hawkins'
  gem 'jekyll-octicons'
end

gem 'webrick'

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem 'tzinfo', '>= 1', '< 3'
  gem 'tzinfo-data'
end
