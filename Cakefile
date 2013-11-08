http = require 'http'
https = require 'https'
fs = require 'fs'
path = require 'path'
{spawn, exec} = require 'child_process'
semver = require 'semver'
AdmZip = require('adm-zip')

# ----------------
# Server / Builder
# ----------------

option '-P', '--production', 'run server in production mode'

LOCAL_BRUNCH = path.join('.', 'node_modules', '.bin', 'brunch')

spawnBrunch = (flags, env) ->
  if fs.existsSync(LOCAL_BRUNCH)
    brunch = spawn LOCAL_BRUNCH, flags, env
  else
    console.error 'Warning, using global brunch. Run `npm install`.'
    brunch = spawn 'brunch', flags, env

  brunch.stdout.on 'data', (data) -> console.log data.toString().trim()
  brunch.stderr.on 'data', (data) -> console.log data.toString().trim()


task 'server', 'start the brunch server in development', (options) ->
  flags = ['w', '-s']
  if options.production?
    flags.push('-P')
    process.env.BRUNCH_ENV = 'production'
  spawnBrunch flags, process.env

task 'build', 'build for production (delete public folder first)', ->
  exec('rm -rf ./public')
  process.env.BRUNCH_ENV = 'production'
  spawnBrunch ['b', '-P'], process.env

# -------------
# Tapas Updates
# -------------
updateMessage = 'update Tapas to latest (Cakefile, package.json, portkey.json,
 config.coffee, generators/*)'
task 'tapas:update', updateMessage, (options) ->
  url = 'https://codeload.github.com/mutewinter/tapas-with-ember/zip/master'
  filesToUpdate = [
    'Cakefile'
    'package.json'
    'portkey.json'
    'config.coffee'
    'generators/'
  ]
  https.get url, (res) ->
    data = []
    dataLen = 0

    res.on('data', (chunk) ->
      data.push(chunk)
      dataLen += chunk.length
    ).on('end', ->
      buf = new Buffer(dataLen)

      pos = 0
      for dataItem in data
        dataItem.copy(buf, pos)
        pos += dataItem.length

      zip = new AdmZip(buf)

      filesToUpdate.forEach (file) ->
        targetFile = "tapas-with-ember-master/#{file}"
        if /\/$/.test(file)
          zip.extractEntryTo(targetFile, file, false, true)
        else
          zip.extractEntryTo(targetFile, '', false, true)
    )

# --------------
# Script Updates
# --------------

EMBER_BASE_URL = 'http://builds.emberjs.com'
GITHUB_API_URL = 'https://api.github.com'
EMBER_GITHUB_API_URL = "#{GITHUB_API_URL}/repos/emberjs/ember.js/tags"
EMBER_DATA_GITHUB_API_URL = "#{GITHUB_API_URL}/repos/emberjs/data/tags"
EMBER = {}
EMBER_DATA = {}
['release', 'beta', 'canary'].forEach (build) ->
  EMBER[build] =
    prod: "#{EMBER_BASE_URL}/#{build}/ember.prod.js"
    dev: "#{EMBER_BASE_URL}/#{build}/ember.js"
  EMBER_DATA[build] =
    prod: "#{EMBER_BASE_URL}/#{build}/ember-data.prod.js"
    dev: "#{EMBER_BASE_URL}/#{build}/ember-data.js"

EMBER['tag'] =
  prod: "#{EMBER_BASE_URL}/tags/{{tag}}/ember.prod.js"
  dev: "#{EMBER_BASE_URL}/tags/{{tag}}/ember.js"

EMBER_DATA['tag'] =
  prod: "#{EMBER_BASE_URL}/tags/{{tag}}/ember-data.prod.js"
  dev: "#{EMBER_BASE_URL}/tags/{{tag}}/ember-data.js"

downloadFile = (src, dest) ->
  file = fs.createWriteStream(dest)
  request = http.get src, (response) ->
    response.pipe file
    console.log('Downloaded ' + src + ' to ' + dest)

downloadEmberFile = (src, dest) ->
  downloadFile(src, "vendor/ember/#{dest}")

listTags = (githubUrl, since, name, command) ->
  request = https.get githubUrl, (response) ->
    data = ''
    response.on 'data', (chunk) -> data += chunk
    response.on 'end', ->
      tags = JSON.parse(data)
      console.log "Tagged #{name} Releases:"
      for tag in tags
        if semver.valid(tag.name) and !semver.lt(tag.name, since)
          firstTag = tag.name unless firstTag
          console.log "  #{tag.name}"
      console.log "Install with cake -t \"#{firstTag}\" #{command}"

installEmberFiles = (project, filename, options) ->
  if 'tag' of options
    # Download a Tag
    tag = options.tag
    tag = "v#{tag}" unless /^v/.test(tag)
    downloadEmberFile(project['tag'].dev.replace(/{{tag}}/, tag),
      "development/#{filename}")
    downloadEmberFile(project['tag'].prod.replace(/{{tag}}/, tag),
      "production/#{filename}")
  else
    # Download a Channel
    channel = options.channel ? 'release'
    downloadEmberFile project[channel].dev, "development/#{filename}"
    downloadEmberFile project[channel].prod, "production/#{filename}"

# Channel
option '-c', '--channel "[CHANNEL_NAME]"',
  'relase, beta, or canary (http://emberjs.com/builds)'

# Tag
option '-t', '--tag "[TAG_NAME]"',
  'a tagged release to install. Run cake ember:list to see known tags'

# -----
# Ember
# -----
task 'ember:install', 'install latest Ember', (options) ->
  installEmberFiles(EMBER, 'ember.js', options)

task 'ember:list', 'list tagged relases of Ember since v1.0.0', (options) ->
  listTags EMBER_GITHUB_API_URL, 'v1.0.0', 'Ember', 'ember:install'

# ----------
# Ember Data
# ----------
task 'ember-data:install', 'install latest Ember Data', (options) ->
  installEmberFiles(EMBER_DATA, 'ember-data.js', options)

task 'ember-data:list', 'list tagged relases of Ember Data', (options) ->
  listTags EMBER_DATA_GITHUB_API_URL, 'v0.0.1', 'Ember Data',
    'ember-data:install'

# -----------
# Ember Model
# -----------
EMBER_MODEL =
  dev: 'http://builds.erikbryn.com/ember-model/ember-model-latest.js'
  prod: 'http://builds.erikbryn.com/ember-model/ember-model-latest.prod.js'

task 'ember-model:install', 'install latest Ember Model', (options) ->
  downloadEmberFile EMBER_MODEL.dev, 'development/ember-model.js'
  downloadEmberFile EMBER_MODEL.prod, 'production/ember-model.js'

# ----------
# Handlebars
# ----------
task 'handlebars:install', 'install latest Handlebars', (options) ->
  downloadFile(
    'http://builds.handlebarsjs.com.s3.amazonaws.com/handlebars-latest.js',
    'vendor/scripts/handlebars.js'
  )
