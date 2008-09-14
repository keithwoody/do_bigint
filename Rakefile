#!/usr/bin/env ruby

# do rake file

require 'pathname'
require 'rubygems'
require 'rake'
require 'rake/clean'
require Pathname('spec/rake/spectask')
require Pathname('rake/rdoctask')

ROOT = Pathname(__FILE__).dirname.expand_path

CLEAN.include '**/{pkg,log,coverage}'

WINDOWS = (RUBY_PLATFORM =~ /mswin|mingw|cygwin/) rescue nil
SUDO    = WINDOWS ? '' : ('sudo' unless ENV['SUDOLESS'])

# projects = %w[data_objects do_jdbc do_mysql do_postgres do_sqlite3]
# Took out do_jdbc since it doesn't build yet.
projects = %w[data_objects do_mysql do_postgres do_sqlite3]

desc 'Install the do gems'
task :install => [ 'ci:install_all' ]

desc 'Release all do gems'
task :release_all do
  projects.each do |dir|
    Dir.chdir(dir){ sh "rake release VERSION=#{ENV["VERSION"]}" }
  end
end

desc 'Run specifications'
Spec::Rake::SpecTask.new(:spec) do |t|
  t.spec_opts << '--format specdoc' << '--color'
  t.spec_files = Pathname.glob((ROOT + '**/spec/**/*_spec.rb').to_s)
end

task 'do:spec' => [ 'spec' ]

namespace :ci do

  projects.each do |gem_name|
    task gem_name do
      ENV['gem_name'] = gem_name

      Rake::Task["ci:run_all"].invoke
    end
  end

  task :install_all do
    projects.each do |gem_name|
      cd(File.join(File.dirname(__FILE__), gem_name))
      sh("rake install")
    end
  end

  task :run_all => [:spec, :install, :doc, :publish]

  task :spec => :define_tasks do
    Rake::Task["#{ENV['gem_name']}:spec"].invoke
  end

  task :doc => :define_tasks do
    Rake::Task["#{ENV['gem_name']}:doc"].invoke
  end

  task :install => :uninstall do
    sh %{cd #{ENV['gem_name']} && rake install}
  end

  task :uninstall do
    sh %{#{SUDO} gem uninstall #{ENV['gem_name']} --ignore-dependencies} rescue nil
  end

  task :publish do
    out = ENV['CC_BUILD_ARTIFACTS'] || "out"
    mkdir_p out unless File.directory? out if out

    mv "rdoc", "#{out}/rdoc" if out
    mv "coverage", "#{out}/coverage_report" if out && File.exists?("coverage")
    mv "rspec_report.html", "#{out}/rspec_report.html" if out
  end

  task :define_tasks do
    gem_name = ENV['gem_name']

    unless FileList[ROOT + "#{gem_name}/ext/**/extconf.rb"].empty?
      file "#{gem_name}/Makefile" => FileList[ROOT + "#{gem_name}/ext/**/extconf.rb", ROOT + "#{gem_name}/ext/**/*.c", ROOT + "#{gem_name}/ext/**/*.h"] do
        system("cd #{gem_name} && ruby ext/extconf.rb")
        system("cd #{gem_name} && make all") || system("cd #{gem_name} && nmake all")
      end
      task "#{gem_name}:spec" => "#{gem_name}/Makefile"
    end

    Spec::Rake::SpecTask.new("#{gem_name}:spec") do |t|
      t.spec_opts = ["--format", "specdoc", "--format", "html:rspec_report.html", "--diff"]
      t.spec_files = Pathname.glob(ENV['FILES'] || (ROOT + "#{gem_name}/spec/**/*_spec.rb").to_s)
      unless ENV['NO_RCOV']
        t.rcov = true
        t.rcov_opts << '--exclude' << "spec,gems,#{(projects - [gem_name]).join(',')}"
        t.rcov_opts << '--text-summary'
        t.rcov_opts << '--sort' << 'coverage' << '--sort-reverse'
        t.rcov_opts << '--only-uncovered'
      end
    end

    Rake::RDocTask.new("#{gem_name}:doc") do |t|
      t.rdoc_dir = 'rdoc'
      t.title    = gem_name
      t.options  = ['--line-numbers', '--inline-source', '--all']
      t.rdoc_files.include("#{gem_name}/lib/**/*.rb", "#{gem_name}/ext/**/*.c")
    end
  end
end
