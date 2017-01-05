#!/usr/bin/env rake

require 'erb'
require 'tty/spinner'
require 'pastel'

#############
# Functions #
#############

# General Syntax

bind        = ->(k, v, context)     { context.tap { context.local_variable_set k, v } }.curry
bool        = ->(value)             { value ? true : false }
disabled    = ->(env)               { ['false', 'nil'].include? ENV[env] }
equal       = ->(a, b)              { a == b }.curry
hash_of     = ->(generator)         { Hash.new { |hash, key| hash[key] = generator.call } }
is          = ->(function)          { ->(*args) { bool.(function.(*args)) } }

# Files and Directories

basename    = ->(path)              { File.basename path }
directory   = ->(path)              { File.directory? path }
dirname     = ->(path)              { File.dirname path }
exists      = ->(path)              { File.exist? path }
glob        = ->(pattern)           { Dir.glob pattern }
hardlink    = ->(source, target)    { File.link source, target }.curry
mkdir       = ->(path)              { Dir.mkdir path unless exists.(path) }
read        = ->(path)              { File.read path }
unlink      = ->(path)              { File.unlink path }
write       = ->(path, content)     { File.write path, content }.curry

directories = ->(path)              { glob.("#{path}/*").select(&directory) }

# String Mangling

combine     = ->(*strings)          { strings.compact.join ' ' }
prefix      = ->(prior, string)     { [prior, string].join }.curry
replace     = ->(from, to, string)  { string.tr from, to }.curry
unprefix    = ->(pattern, string)   { string.gsub(/^#{pattern}/, '') }.curry
unsuffix    = ->(pattern, string)   { string.gsub(/#{pattern}$/, '') }.curry

unpath      = ->(parent, path)      { unprefix.("#{parent}/".squeeze('/'), path) }.curry

# Extract image names and tags from mixed-format references

tag         = ->(img)               { img =~ /:/ ? tag.(replace.(':', '/').(img)) : img =~ %r{/} ? basename.(img) : 'latest' }
name        = ->(img)               { img =~ /:/ ? name.(replace.(':', '/').(img)) : img =~ %r{/} ? basename.(dirname.(img)) : img }
image       = ->(img, *args)        { combine.("#{name.(img)}:#{tag.(img)}", *args) }

# Scope isolation for templates to make it clear what's being passed *to* ERB

sandbox     = ->(**vars)            { vars.reduce(TOPLEVEL_BINDING.dup) { |c, kv| k, v = *kv ; bind.(k, v, c) } }
template    = ->(string, **vars)    { ERB.new(string).result(sandbox.(vars)) }

# Marginally better user experience

spinner     = ->(message)           { TTY::Spinner.new "[:spinner] #{message} ...", format: :arrow_pulse }
pastel      = ->(*styles)           { [*styles, :detach].reduce Pastel.new, :public_send }
blue        = ->(message)           { pastel.(:blue).(message) }
green       = ->(message)           { pastel.(:green).(message) }

#################
# Configurables #
#################

OWNER       = ENV.fetch('DOCKER_OWNER') { ENV.fetch('USERNAME') { ENV.fetch('USER') { %w(whoami).chomp } } }
WORKDIR     = ENV.fetch('WORKDIR')    { Dir.pwd }
IMG_DIR     = ENV.fetch('IMG_DIR')    { %w(images image img).map(&prefix.(WORKDIR)).find(&is.(directory)) || "#{WORKDIR}/images" }
SRC_DIR     = ENV.fetch('SRC_DIR')    { %w(sources source src).map(&prefix.(WORKDIR)).find(&is.(directory)) || "#{WORKDIR}/source" }
PKG_DIR     = ENV.fetch('PKG_DIR')    { %w(packages package pkg).map(&prefix.(WORKDIR)).find(&is.(directory)) || "#{WORKDIR}/packages" }
BUILDER     = ENV.fetch('BUILDER')    { 'builder' }
PACKAGER    = ENV.fetch('PACKAGER')   { 'packager' }
RUNTIME     = ENV.fetch('RUNTIME')    { 'runtime' }

[WORKDIR, IMG_DIR, SRC_DIR, PKG_DIR].each(&mkdir) unless disabled.('MKDIR')

# Context-dependent definitions

SOURCES     = directories.(SRC_DIR).map(&basename)
DOCKERFILES = glob.("#{IMG_DIR}/**/Dockerfile{,.erb}").map(&unpath.(IMG_DIR))
IMAGES      = DOCKERFILES.map(&dirname).reduce(hash_of.(-> { Set.new })) { |h, img| h.tap { h[name.(img)] << tag.(img) } }

#############
# Functions #
#############

workpath    = ->(path)       { path =~ %r{^/} ? path : "#{WORKDIR}/#{path}" }
src_dir     = ->(src)        { "#{SRC_DIR}/#{unpath.(SRC_DIR, src)}" }
pkg_dir     = ->(pkg)        { "#{PKG_DIR}/#{unpath.(PKG_DIR, pkg)}" }
img_dir     = ->(img)        { "#{IMG_DIR}/#{unpath.(IMG_DIR).(unsuffix.('/latest').(replace.(':', '/').(img)))}" }
canonical   = ->(img)        { "#{OWNER}/#{img}" }
dockerfile  = ->(img)        { "#{img_dir.(img)}/Dockerfile" }

# Context Management

tags        = ->(img)        { IMAGES.fetch(img) { [] } }
runtime     = ->(img)        { tags.(img).find(&equal.(RUNTIME)) || "#{img}:latest" }
builder     = ->(img)        { tags.(img).find(&equal.(BUILDER))  || runtime.(img) }
packager    = ->(img)        { tags.(img).find(&equal.(PACKAGER)) || builder.(img) }

# Docker

docker     = ->(cmd, *args)  { combine.('docker', *args, cmd) }
run        = ->(img, *args)  { combine.('run --rm --tty', *args, img) }
build      = ->(img)         { combine.('build --tag', canonical.(image.(img)), img_dir.(img)) }
volume     = ->(path, mnt)   { "--volume #{workpath.(path)}:#{mnt}" }.curry

src_volume = ->(src)         { volume.(src_dir.(src), '/src') }
pkg_volume = ->(pkg)         { volume.(pkg_dir.(pkg), '/pkg') }

# Task Management

tasks      = ->(prefix)      { Rake.application.tasks.select { |task| task.name.start_with? prefix } }
max_depth  = ->(max, task)   { task.name.count(':').between? 0, max }.curry

#########
# Tasks #
#########

# namespace :fetch   { directories.(SRC_DIR).each(&fetch_task) }  # (Clone || Pull) && Checkout

namespace :dockerfile do
  IMAGES.each do |img, tags|
    namespace(img) do
      tags.each do |tag|
        task tag do
          target = dockerfile.("#{img}:#{tag}")
          spinner.(blue.("Regenerating #{target}")).run(green.('done!')) do
            write.(target).(template.(read.("#{target}.erb"), namespace: OWNER))
          end if exists.("#{target}.erb")
        end
      end
    end

    task img => tags.map(&prefix.("dockerfile:#{img}:"))
  end
end

namespace :link do
  IMAGES.each do |img, tags|
    namespace(img) do
      tags.each do |tag|
        task tag do
          target = img_dir.("#{img}:#{tag}") + '/pkg'
          spinner.(blue.("Linking PKG_DIR into build context for #{img}:#{tag}")).run(green.('done!')) do
            unlink.(target) if exists.(target)
            hardlink.(PKG_DIR, target)
            sleep 0.12
          end
        end
      end
    end

    task img => tags.map(&prefix.("link:#{img}:"))
  end
end

namespace :image do
  IMAGES.each do |img, tags|
    namespace(img) do
      tags.each do |tag|
        dependencies = %W(dockerfile:#{img}:#{tag})

        case tag
        when PACKAGER
          dependencies << "image:#{img}:#{BUILDER}"
        when 'latest'
          dependencies << "package:#{img}"
          dependencies << "link:#{img}:#{tag}"
          dependencies << "image:#{img}:#{RUNTIME}"
        end

        task tag => dependencies do
          sh docker.(build.("#{img}:#{tag}")) # TODO: Add dependencies
        end
      end
    end
  end
end

namespace :build do
  directories.(SRC_DIR).map(&basename).each do |src|
    dependencies = tags.(src).any?(&equal.(BUILDER)) ? "image:#{src}:#{BUILDER}" : []
    task src => dependencies do
      sh docker.(run.(canonical.("#{src}:#{builder.(src)}"), src_volume.(src)))
    end
  end
end

namespace :package do
  directories.(SRC_DIR).map(&basename).each do |pkg|
    dependencies = [
      ("build:#{pkg}" if Rake.application.tasks.map(&:name).any?(&equal.("build:#{pkg}"))),
      ("image:#{pkg}:#{PACKAGER}" if tags.(pkg).any?(&equal.(PACKAGER)))
    ]

    task pkg => dependencies do
      sh docker.(run.(canonical.("#{pkg}:#{packager.(pkg)}"), src_volume.(pkg), pkg_volume.(pkg)))
    end
  end
end

# Aggregate Tasks

task build:       tasks.('build:')
task images:      tasks.('image:')
task packages:    tasks.('package:')
task dockerfiles: tasks.('dockerfile:').select(&max_depth.(1))
task links:       tasks.('link:').select(&max_depth.(1))

task all: %i(images build packages)
task default: :all

# Development Tasks

task(:tasks) { puts Rake.application.tasks.map(&name) }
task 'jack-in' do
  require 'pry'
  binding.pry
  abort
end
