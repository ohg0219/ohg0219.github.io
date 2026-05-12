source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.5"

# Chirpy 가 의존하는 Jekyll 플러그인을 :jekyll_plugins 그룹에 명시해야
# Jekyll 의 plugin_manager 가 _config.yml 의 plugins 목록을 로드할 수 있다.
group :jekyll_plugins do
  gem "jekyll-paginate", "~> 1.1"
  gem "jekyll-redirect-from", "~> 0.16"
  gem "jekyll-seo-tag", "~> 2.8"
  gem "jekyll-archives", "~> 2.3"
  gem "jekyll-sitemap", "~> 1.4"
end

gem "html-proofer", "~> 5.0", group: :test

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]

gem "webrick", "~> 1.8"
