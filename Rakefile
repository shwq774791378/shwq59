require "find"
require 'fileutils'
require 'json'
require 'open3'
require 'open-uri'

task :default => [:test]

task :clean_api do
  FileUtils.rm_rf('content/api')
end

desc "Remove the tmp dir"
task :remove_tmp_dir do
  FileUtils.rm_r('tmp') if File.exist?('tmp')
end

desc "Remove the output dir"
task :remove_output_dir do
  FileUtils.rm_r('output') if File.exist?('output')
end

desc 'Builds the site'
task :build do
  Bundler.with_clean_env do
    Open3.popen3('npm run gulp build') do |_, stdout, stderr, wait_thr|
      puts stdout.read
      status = wait_thr.value
      abort stderr.read unless status.success?
    end
  end
end

desc "Run the HTML-Proofer"
task :run_proofer do
  require 'html-proofer'

  # Ignore platform switcher hash URLs
  platform_hash_urls = ['#platform-mac', '#platform-windows', '#platform-linux', '#platform-all']
  HTMLProofer.check_directory("./output", {
    :file_ignore => [%r(/output/api/)],
    :url_ignore => platform_hash_urls,
    :typhoeus => { :ssl_verifypeer => false },
  }).run
end

# Detects instances of Issue #204
desc "Validate article titles match file name and header"
task :validate_article_titles do
  require "colorize"
  require_relative "./lib/tasks/validate_article_title"

  pass = true

  Find.find('./content') do |path|
    if path =~ /sections.*\.md$/
      validate = ValidateArticleTitle.new(path).all

      unless validate.errors.empty?
        pass = false
        validate.errors.each { |error| puts "#{error}".red }
      end
    end
  end

  raise "Validate article titles failed" unless pass
end

desc "Test the output"
task :test => [:validate_article_titles, :remove_tmp_dir, :remove_output_dir, :build, :run_proofer]

# Prompt user for a commit message; default: P U B L I S H :emoji:
def commit_message(no_commit_msg = false)
  publish_emojis = [':boom:', ':rocket:', ':metal:', ':bulb:', ':zap:',
    ':sailboat:', ':gift:', ':ship:', ':shipit:', ':sparkles:', ':rainbow:']
  default_message = "P U B L I S H #{publish_emojis.sample}"

  unless no_commit_msg
    print "Enter a commit message (default: '#{default_message}'): "
    STDOUT.flush
    mesg = STDIN.gets.chomp.strip
  end

  mesg = default_message if mesg.nil? || mesg == ''
  mesg << "\nGenerated from #{ENV['BUILD_SHA']}" if ENV['BUILD_SHA']
  mesg.delete("'") # Allow this to be handed off via -m '#{message}'
end

desc 'Publish to https://flight-manual.atom.io'
task :publish, [:no_commit_msg] => [:remove_tmp_dir, :remove_output_dir, :build] do |_, args|
  require "shellwords"

  message = commit_message(args[:no_commit_msg])

  system "git add -f output/"
  tree = `git write-tree --prefix=output`.chomp
  commit = `echo #{message.shellescape} | git commit-tree #{tree} -p gh-pages`.chomp
  system "git update-ref refs/heads/gh-pages #{commit}"
  system 'git push origin gh-pages --force'
  system 'git reset'
end
