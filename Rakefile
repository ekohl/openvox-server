require 'open3'
RED = "\033[31m"
GREEN = "\033[32m"
RESET = "\033[0m"

def run_command(cmd, silent: true, print_command: false, report_status: false)
  puts "#{GREEN}Running #{cmd}#{RESET}" if print_command
  output = ''
  Open3.popen2e(cmd) do |_stdin, stdout_stderr, thread|
    stdout_stderr.each do |line|
      puts line unless silent
      output += line
    end
    exitcode = thread.value.exitstatus
    unless exitcode.zero?
      err = "#{RED}Command failed! Command: #{cmd}, Exit code: #{exitcode}"
      # Print details if we were running silent
      err += "\nOutput:\n#{output}" if silent
      err += RESET
      abort err
    end
    puts "#{GREEN}Command finished with status #{exitcode}#{RESET}" if report_status
  end
  output.chomp
end

Dir.glob(File.join('tasks/**/*.rake')).each { |file| load file }

### puppetlabs stuff ###
require 'open-uri'
require 'json'
require 'pp'

PROJECT_ROOT = File.dirname(__FILE__)
ACCEPTANCE_ROOT = ENV['ACCEPTANCE_ROOT'] ||
  File.join(PROJECT_ROOT, 'acceptance')
BEAKER_OPTIONS_FILE = File.join(ACCEPTANCE_ROOT, 'config', 'beaker', 'options.rb')
PUPPET_SRC = File.join(PROJECT_ROOT, 'ruby', 'puppet')
PUPPET_LIB = File.join(PROJECT_ROOT, 'ruby', 'puppet', 'lib')
FACTER_LIB = File.join(PROJECT_ROOT, 'ruby', 'facter', 'lib')
PUPPET_SERVER_RUBY_SRC = File.join(PROJECT_ROOT, 'src', 'ruby', 'puppetserver-lib')
PUPPET_SERVER_RUBY_SPEC = File.join(PROJECT_ROOT, 'spec')
PUPPET_SUBMODULE_PATH = File.join('ruby','puppet')
FACTER_SUBMODULE_PATH = File.join('ruby','facter')
# Branch of puppetserver for which to update submodule pins
PUPPETSERVER_BRANCH = ENV['PUPPETSERVER_BRANCH'] || 'main'
# Branch of puppet-agent to track for passing puppet SHA
PUPPET_AGENT_BRANCH = ENV['PUPPET_AGENT_BRANCH'] || 'main'
# Branch of puppet-agent to track for Facter's passing SHA.
# This needs to be separate because for 6.x, we want to use facter#main
# instead of Facter 3.
FACTER_BRANCH = ENV['FACTER_BRANCH'] || 'main'

TEST_GEMS_DIR = File.join(PROJECT_ROOT, 'vendor', 'test_gems')
TEST_BUNDLE_DIR = File.join(PROJECT_ROOT, 'vendor', 'test_bundle')

GEM_SOURCE = ENV['GEM_SOURCE'] || 'https://rubygems.org'

# Jenkins uses `$LEIN` configure the lein version to use
LEIN_PATH = ENV['LEIN'] || 'lein'

def assemble_default_beaker_config
  if ENV["BEAKER_CONFIG"]
    return ENV["BEAKER_CONFIG"]
  end

  platform = ENV['PLATFORM']
  layout = ENV['LAYOUT']

  if platform and layout
    beaker_config = "#{ACCEPTANCE_ROOT}/config/beaker/jenkins/"
    beaker_config += "#{platform}-#{layout}.cfg"
  else
    abort "Must specify an appropriate value for BEAKER_CONFIG. See acceptance/README.md"
  end

  return beaker_config
end

def setup_smoke_hosts_config
  sh "bundle exec beaker-hostgenerator centos7-64m-64a > acceptance/scripts/hosts.cfg"
end

def basic_smoke_test(package_version)
  beaker = "PACKAGE_BUILD_VERSION=#{package_version}"
  beaker += " bundle exec beaker --debug --root-keys --repo-proxy"
  beaker += " --preserve-hosts always"
  beaker += " --type aio"
  beaker += " --helper acceptance/lib/helper.rb"
  beaker += " --options-file #{BEAKER_OPTIONS_FILE}"
  beaker += " --load-path acceptance/lib"
  beaker += " --config acceptance/scripts/hosts.cfg"
  beaker += " --keyfile ~/.ssh/id_rsa-acceptance"
  beaker += " --pre-suite acceptance/suites/pre_suite/foss"
  beaker += " --post-suite acceptance/suites/post_suite"
  beaker += " --tests acceptance/suites/tests/00_smoke"

  sh beaker
end

# TODO: this could be DRY'd up with the method above, but it seemed like it
# might make it a little harder to read and didn't seem worth the effort yet
def re_run_basic_smoke_test
  beaker = "bundle exec beaker --debug --root-keys --repo-proxy"
  beaker += " --preserve-hosts always"
  beaker += " --type aio"
  beaker += " --helper acceptance/lib/helper.rb"
  beaker += " --options-file #{BEAKER_OPTIONS_FILE}"
  beaker += " --load-path acceptance/lib"
  beaker += " --config acceptance/scripts/hosts.cfg"
  beaker += " --keyfile ~/.ssh/id_rsa-acceptance"
  beaker += " --tests acceptance/suites/tests/00_smoke"

  sh beaker
end

namespace :spec do
  task :init do
    if ! Dir.exist? TEST_GEMS_DIR
      ## Install bundler
      ## Line 1 launches the JRuby that we depend on via leiningen
      ## Line 2 programmatically runs 'gem install bundler' via the gem command that comes with JRuby
      gem_install_bundler = <<-CMD
      GEM_HOME='#{TEST_GEMS_DIR}' GEM_PATH='#{TEST_GEMS_DIR}' \
      #{LEIN_PATH} gem install -i '#{TEST_GEMS_DIR}' bundler --version 2.5.10 --no-document --source '#{GEM_SOURCE}'
      CMD
      sh gem_install_bundler

      path = ENV['PATH']
      ## Install gems via bundler
      ## Line 1 makes sure that our local bundler script is on the path first
      ## Line 2 tells bundler to use puppet's Gemfile
      ## Line 3 tells JRuby where to look for gems
      ## Line 4 launches the JRuby that we depend on via leiningen
      ## Line 5 runs our bundle install script
      bundle_install = <<-CMD
      PATH='#{TEST_GEMS_DIR}/bin:#{path}' \
      BUNDLE_GEMFILE='#{PUPPET_SRC}/Gemfile' \
      GEM_HOME='#{TEST_GEMS_DIR}' GEM_PATH='#{TEST_GEMS_DIR}' \
      #{LEIN_PATH} run -m org.jruby.Main \
        -S bundle install --without extra development packaging --path='#{TEST_BUNDLE_DIR}' --retry=3
      CMD
      sh bundle_install
    end
  end
end

desc "Run rspec tests"
task :spec, [:rspec_opts] => ["spec:init"] do |t, args|
  rspec_opts = args[:rspec_opts] || './spec'
  ## Run RSpec via our JRuby dependency
  ## Line 1 tells bundler to use puppet's Gemfile
  ## Line 2 tells JRuby where to look for gems
  ## Line 3 launches the JRuby that we depend on via leiningen
  ## Line 4 adds all our Ruby source to the JRuby LOAD_PATH
  ## Line 5 runs our rspec wrapper script
  ## <sarcasm-font>dang ole real easy man</sarcasm-font>
  run_rspec_with_jruby = <<-CMD
    BUNDLE_GEMFILE='#{PUPPET_SRC}/Gemfile' \
    GEM_HOME='#{TEST_GEMS_DIR}' GEM_PATH='#{TEST_GEMS_DIR}' \
    #{LEIN_PATH} run -m org.jruby.Main \
      -I'#{PUPPET_SERVER_RUBY_SPEC}' -I'#{PUPPET_LIB}' -I'#{FACTER_LIB}' -I'#{PUPPET_SERVER_RUBY_SRC}' \
      ./spec/run_specs.rb #{rspec_opts}
  CMD
  sh run_rspec_with_jruby
end

namespace :test do

  namespace :acceptance do
    desc "Run beaker based acceptance tests"
    task :beaker do |t, args|

      # variables that take a limited set of acceptable strings
      type = ENV["BEAKER_TYPE"] || "pe"

      # variables that take pathnames
      beakeropts = ENV["BEAKER_OPTS"] || ""
      presuite = ENV["BEAKER_PRESUITE"] || "#{ACCEPTANCE_ROOT}/suites/pre_suite/#{type}"
      postsuite = ENV["BEAKER_POSTSUITE"] || ""
      helper = ENV["BEAKER_HELPER"] || "#{ACCEPTANCE_ROOT}/lib/helper.rb"
      testsuite = ENV["BEAKER_TESTSUITE"] || "#{ACCEPTANCE_ROOT}/suites/tests"
      loadpath = ENV["BEAKER_LOADPATH"] || ""
      options = ENV["BEAKER_OPTIONSFILE"] || "#{ACCEPTANCE_ROOT}/config/beaker/options.rb"

      # variables requiring some assembly
      config = assemble_default_beaker_config

      beaker = "beaker "

      beaker += " -c #{config}"
      beaker += " --helper #{helper}"
      beaker += " --type #{type}"

      beaker += " --options-file #{options}" if options != ''
      beaker += " --load-path #{loadpath}" if loadpath != ''
      beaker += " --pre-suite #{presuite}" if presuite != ''
      beaker += " --post-suite #{postsuite}" if postsuite != ''
      beaker += " --tests " + testsuite if testsuite != ''

      beaker += " " + beakeropts

      sh beaker
    end

    desc "Do an ezbake build, and then a beaker smoke test off of that build, preserving the vmpooler host"
    task :bakeNbeak do
      package_version = nil

      Open3.popen3("#{LEIN_PATH} with-profile ezbake ezbake build 2>&1") do |stdin, stdout, stderr, thread|
        # sleep 5
        # puts "STDOUT IS: #{stdout}"
        success = true
        stdout.each do |line|
          if match = line.match(%r|^Your packages will be available at http://builds.delivery.puppetlabs.net/puppetserver/(.*)$|)
            package_version = match[1]
          elsif line =~ /^Packaging FAILURE\s*$/
            success = false
          end
          puts line
        end
        exit_code = thread.value
        if success == true
          puts "PACKAGE VERSION IS #{package_version}"
        else
          puts "\n\nPACKAGING FAILED!  exit code is '#{exit_code}'.  STDERR IS:"
          puts stderr.read
          exit 1
        end
      end

      begin
        setup_smoke_hosts_config()
        basic_smoke_test(package_version)
      rescue => e
        puts "\n\nJOB FAILED; PACKAGE VERSION WAS: #{package_version}\n\n"
        raise e
      end
    end

    desc "Do a basic smoke test, using the package version specified by PACKAGE_BUILD_VERSION, preserving the vmpooler host"
    task :smoke do
      package_version = ENV["PACKAGE_BUILD_VERSION"]
      unless package_version
        STDERR.puts("'smoke' task requires PACKAGE_BUILD_VERSION environment variable")
        exit 1
      end
      setup_smoke_hosts_config()
      basic_smoke_test(package_version)
    end

    desc "Re-run the basic smoke test on the host preserved from a previous run of the 'smoke' task"
    task :resmoke do
      re_run_basic_smoke_test()
    end

  end
end

namespace :package do
  task :bootstrap do
    puts 'Bootstrap is no longer needed, using packaging-as-a-gem'
  end
  task :implode do
    puts 'Implode is no longer needed, using packaging-as-a-gem'
  end
end

#begin
#  require 'packaging'
#  Pkg::Util::RakeUtils.load_packaging_tasks
#rescue LoadError => e
#  puts "Error loading packaging rake tasks: #{e}"
#end

begin
  require 'github_changelog_generator/task'
rescue LoadError
  task :changelog do
    abort('Run `bundle install --with release` to install the `github_changelog_generator` gem.')
  end
else
  GitHubChangelogGenerator::RakeTask.new :changelog do |config|
    config.header = <<~HEADER.chomp
      # Changelog

      All notable changes to this project will be documented in this file.
    HEADER
    config.user = 'openvoxproject'
    config.project = 'openvox-server'
    config.exclude_labels = %w[dependencies duplicate question invalid wontfix wont-fix modulesync skip-changelog]
    # this is probably the worst way to do this
    # ideally there would be a VERSION file and clojure and Rake would read it
    config.future_release = File.readlines('project.clj').first.scan(/".*"/).first.gsub('"', '')
    # we limit the changelog to all new openvox releases, to skip perforce onces
    # otherwise the changelog generate takes a lot amount of time
    config.since_tag = '8.7.0'
    config.exclude_tags_regex = /\A7\./
  end
end
