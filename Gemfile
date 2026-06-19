# frozen_string_literal: true

source "https://rubygems.org"

gemspec

group :test do
  gem "html-proofer", "~> 4.4"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows.
# wdm 0.1.1 does not build with Ruby 3.4+ on mingw-ucrt, and it is not
# required for non-watch builds such as `jekyll build`.
if Gem::Version.new(RUBY_VERSION) < Gem::Version.new("3.4")
  gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
end

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

gem "webrick", "~> 1.8"
