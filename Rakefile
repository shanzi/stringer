require "bundler"
Bundler.setup

require 'logger'
require "rubygems"
require "net/http"
require "active_record"
require "delayed_job"
require "delayed_job_active_record"

require "sinatra/activerecord/rake"

require "rufus-scheduler"

require "./app"
require_relative "./app/jobs/fetch_feed_job"
require_relative "./app/tasks/fetch_feeds"
require_relative "./app/tasks/change_password"
require_relative "./app/tasks/remove_old_stories.rb"

desc "Start runing scheduler"
task :run_scheduler do
  logger = Logger.new(STDOUT)
  logger.level = Logger::DEBUG

  logger.info 'Starting scheduler'
  scheduler = Rufus::Scheduler.new
  scheduler.every '20m', :first => :now do
    logger.info "** Start fetching feeds..."
    Rake::Task["lazy_fetch"].execute
    logger.info "** Finished fetching feeds..."
  end
  scheduler.join
end

desc "Fetch all feeds."
task :fetch_feeds do
  FetchFeeds.new(Feed.all).fetch_all
end

desc "Lazily fetch all feeds."
task :lazy_fetch do
  if ENV["APP_URL"]
    uri = URI(ENV["APP_URL"])
    Net::HTTP.get_response(uri)
  end

  FeedRepository.list.each do |feed|
    Delayed::Job.enqueue FetchFeedJob.new(feed.id)
  end
end

desc "Fetch single feed"
task :fetch_feed, :id do |_t, args|
  FetchFeed.new(Feed.find(args[:id])).fetch
end

desc "Clear the delayed_job queue."
task :clear_jobs do
  Delayed::Job.delete_all
end

desc "Work the delayed_job queue."
task :work_jobs do
  Delayed::Job.delete_all

  3.times do
    Delayed::Worker.new(
      min_priority: ENV["MIN_PRIORITY"],
      max_priority: ENV["MAX_PRIORITY"]
    ).start
  end
end

desc "Change your password"
task :change_password do
  ChangePassword.new.change_password
end

desc "Clean up old stories that are read and unstarred"
task :cleanup_old_stories, :number_of_days do |_t, args|
  args.with_defaults(number_of_days: 30)
  RemoveOldStories.remove!(args[:number_of_days].to_i)
end

desc "Start server and serve JavaScript test suite at /test"
task :test_js do
  require_relative "./spec/javascript/test_controller"
  Stringer.run!
end

begin
  require "rspec/core/rake_task"

  RSpec::Core::RakeTask.new(:speedy_tests) do |t|
    t.rspec_opts = "--tag ~speed:slow"
  end

  RSpec::Core::RakeTask.new(:spec)

  task default: [:speedy_tests]
rescue LoadError # rubocop:disable Lint/HandleExceptions
  # allow for bundle install --without development:test
end
