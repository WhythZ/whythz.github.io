# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.0", ">= 7.0.1"

# gem "html-proofer", "~> 5.0", group: :test
group :test do
  gem "html-proofer", "~> 5.0"
end

# 若无此项则在Windows端执行jekyll server会产生异常
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]