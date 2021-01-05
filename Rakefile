require 'rspec/core'
require 'rspec/core/rake_task'
require 'json'
require 'yaml'
require 'fileutils'
require 'sequel'

# system database configuration
require_relative 'lib/system/database'
include Intrigue::Core::System::Database

# Config files
$intrigue_basedir = File.dirname(__FILE__)
$intrigue_environment = ENV["INTRIGUE_ENV"] || "development"
$intrigueio_api_key = ENV["INTRIGUEIO_API_KEY"]

# Configuration and scripts
procfile_file = "#{$intrigue_basedir}/Procfile"
config_directory = "#{$intrigue_basedir}/config"
puma_config_file = "#{$intrigue_basedir}/config/puma.rb"
system_config_file = "#{$intrigue_basedir}/config/config.json"
database_config_file = "#{$intrigue_basedir}/config/database.yml"
sidekiq_config_file = "#{$intrigue_basedir}/config/sidekiq.yml"
redis_config_file = "#{$intrigue_basedir}/config/redis.yml"
control_script = "#{$intrigue_basedir}/util/control.sh"

all_config_files = [
  procfile_file,
  puma_config_file,
  system_config_file,
  database_config_file,
  sidekiq_config_file,
  redis_config_file
]

desc "Clean"
task :clean do
  puts "[+] Cleaning up!"
  all_config_files.each do |c|
    FileUtils.mv c, "#{c}.backup"
  end
end

desc "System Cleanup"
task :cleanup do
  FileUtils.rm procfile_file, :force => true
  FileUtils.rm puma_config_file, :force => true
  FileUtils.rm system_config_file, :force => true
  FileUtils.rm database_config_file, :force => true
  FileUtils.rm sidekiq_config_file, :force => true
  FileUtils.rm redis_config_file, :force => true
  FileUtils.rm "#{config_directory}/server.key", :force => true
  FileUtils.rm "#{config_directory}/server.crt", :force => true
end

desc "System Update"
task :update do
  puts "[+] Downloading latest data files..."
  Dir.chdir("#{$intrigue_basedir}/data/"){ puts %x["./get_latest.sh"] }
end

def _get_global_entities
  uri = "https://app.intrigue.io/api/system/entities/global/entities/?key=#{$intrigueio_api_key}"
  begin
    puts "[+] Making request for global entities!"
    response = RestClient.get(uri)

    # handle missing data
    return -1 unless response && response.length > 0

    j = JSON.parse(response.body)
  rescue JSON::ParserError => e
    puts "[+] Unable to parse json: #{e}"
    return -1
  end
j
end

desc "Load Global Namespace"
task :load_global_namespace do
  require_relative 'core'

 # First, always pull, and load in the global entities
 puts "[+] Getting global entities intel from Intrigue.io API"
 global_entities = _get_global_entities
 # wait a while and try again if we didnt get an bootstrap list
 until global_entities && global_entities.kind_of?(Hash)
   wait_time = rand(100)
   puts "[ ] Trying again after #{wait_time} to get global entities"
   sleep wait_time
   global_entities = _get_global_entities
   # check it first
   if global_entities == -1
      puts "[-] unable to get global entities intel, retrying..."
      next
   end
 end

 # LOAD IT IN  
 puts "[+] Loading in entities intel from Intrigue.io API"
 Intrigue::Core::Model::GlobalEntity.load_global_namespace(global_entities)
 global_entities = nil

  puts "[+] Done pulling global namespace"
end

desc "System Setup"
task :setup do

  puts "[+] Setup initiated!"

  ## Copy puma config into place
  puts "[+] Copying Procfile...."
  unless File.exist? procfile_file
    puts "[+] Creating.... #{procfile_file}"
    FileUtils.cp "#{procfile_file}.default", procfile_file
  end

  ## Copy puma config into place
  puts "[+] Copying puma config...."
  unless File.exist? puma_config_file
    puts "[+] Creating.... #{puma_config_file}"
    FileUtils.cp "#{puma_config_file}.default", puma_config_file
  end

  ## Copy system config into place
  puts "[+] Copying system config...."

  if File.exist? system_config_file

    puts "[ ] File already exists, skipping: #{system_config_file}"
    config = JSON.parse(File.read(system_config_file))
    system_password = config["credentials"]["password"]

  else

    puts "[+] Creating system config: #{system_config_file}"
    FileUtils.cp "#{system_config_file}.default", system_config_file

    # Set up our password
    config = JSON.parse(File.read(system_config_file))
    if config["credentials"]["password"] == "AUTOGENERATED" 
      if ENV["INTRIGUE_PASS"] # if we were passed on the CLI
        config["credentials"]["username"] = ENV["INTRIGUE_USER"] || "intrigue"
        config["credentials"]["password"] = ENV["INTRIGUE_PASS"]
      else # generate a new password
        system_password = "#{(0...11).map { ('a'..'z').to_a[rand(26)] }.join}"
        puts "[+] GENERATING NEW SYSTEM PASSWORD!" 
        config["credentials"]["password"] = system_password # Save it
      end
      system_password = config["credentials"]["password"]
    end

    # Set up our api_key
    if config["credentials"]["api_key"] == "AUTOGENERATED" 
      if ENV["INTRIGUE_API_KEY"] # if we were passed on the CLI
        config["credentials"]["api_key"] = ENV["INTRIGUE_API_KEY"]
      else
        # generate a new api_key
        api_key = "#{(0...11).map { ('a'..'z').to_a[rand(26)] }.join}"
        puts "[+] GENERATING NEW SYSTEM API KEY!"
        config["credentials"]["api_key"] = api_key # Save it
      end
    else
      api_key = config["credentials"]["api_key"]
    end

    # write the password / api key s
    File.open(system_config_file,"w").puts JSON.pretty_generate(config)

  end

  # Create SSL Cert  
  unless (File.exist?("#{$intrigue_basedir}/config/server.key") || File.exist?("#{$intrigue_basedir}/config/server.crt"))
    puts "[+] Generating A new self-signed SSL Certificate..."
    Dir.chdir("#{$intrigue_basedir}/config/"){ 
      subject_name = "/C=GB/ST=London/L=London/O=Global Security/OU=IT Department/CN=intrigue.local"
      command = "openssl req -subj '#{subject_name}' -new -newkey rsa:2048 -sha256 -days 365 -nodes -x509 -addext 'extendedKeyUsage = serverAuth' -keyout server.key -out server.crt > /dev/null 2>&1"
      puts `#{command}`
    }
  else
    puts "[+] SSL Certificate exists..."
  end

  ## Copy database config into place
  puts "[+] Copying database config."
  unless File.exist? database_config_file
    puts "[+] Creating.... #{database_config_file}"
    FileUtils.cp "#{database_config_file}.default", database_config_file

    # Set up our password
    config = YAML.load_file(database_config_file)
    config["development"]["password"] = system_password
    config["production"]["password"] = system_password
    File.open(database_config_file,"w").puts YAML.dump config
  end

  ## Copy redis config into place
  puts "[+] Copying redis config."
  unless File.exist? redis_config_file
    puts "[+] Creating.... #{redis_config_file}"
    FileUtils.cp "#{redis_config_file}.default", redis_config_file
  end

  ## Place sidekiq task worker configs into place
  puts "[+] Setting up sidekiq config...."
  worker_configs = [
    sidekiq_config_file
  ]

  worker_configs.each do |wc|
    unless File.exist?(wc)
      puts "[+] Copying: #{wc}.default"
      FileUtils.cp "#{wc}.default", wc
    end
  end
  # end worker config placement

  puts "[+] Downloading latest data files..."
  Dir.chdir("#{$intrigue_basedir}/data/"){ puts %x["./get_latest.sh"] }

  # Print it
  puts 
  puts "[+] ==================================="
  puts "[+] SYSTEM USERNAME: intrigue"
  puts "[+] SYSTEM PASSWORD: #{system_password}"
  puts "[+] ==================================="
  puts 

  puts "[+] Complete!"

end

namespace :db do
  desc "Prints current schema version"
  task :version do
    setup_database
    version = if $db.tables.include?(:schema_info)
      $db[:schema_info].first[:version]
    end || 0
    puts "[+] Schema Version: #{version}"
  end

  desc "Perform migration up to latest migration available"
  task :migrate do
    setup_database
    ::Sequel::Migrator.run($db, "db")
    Rake::Task['db:version'].execute
  end

  desc "Perform rollback to specified target or full rollback as default"
  task :rollback, :target do |t, args|
    setup_database
    args.with_defaults(:target => 0)
    ::Sequel::Migrator.run($db, "db", :target => args[:target].to_i)
    Rake::Task['db:version'].execute
  end

  desc "Perform migration reset (full rollback and migration)"
  task :reset do
    setup_database
    ::Sequel::Migrator.run($db, "db", :target => 0)
    ::Sequel::Migrator.run($db, "db")
    Rake::Task['db:version'].execute
  end
end
