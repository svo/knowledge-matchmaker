Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # thinking-extractor
  config.vm.network "forwarded_port", guest: 8001, host: 28001

  # corpus-indexer
  config.vm.network "forwarded_port", guest: 8002, host: 28002

  # relationship-engine
  config.vm.network "forwarded_port", guest: 8003, host: 28003

  # ui (Next.js frontend)
  config.vm.network "forwarded_port", guest: 3000, host: 23000
end
