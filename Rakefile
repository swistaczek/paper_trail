<<<<<<< HEAD
require 'bundler'
Bundler::GemHelper.install_tasks

require 'rake/testtask'

desc 'Test the paper_trail plugin.'
Rake::TestTask.new(:test) do |t|
  t.libs << 'lib'
  t.libs << 'test'
  t.pattern = 'test/**/*_test.rb'
  t.verbose = false
end

desc 'Default: run unit tests.'
task :default => :test
=======
require "bundler/gem_tasks"
>>>>>>> adcf423075c29aaa4a1f100f8ea041143e21beb6
