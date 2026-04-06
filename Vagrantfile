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

  config.vm.provision "shell", inline: <<-SHELL
    echo "NEXT_PUBLIC_THINKING_EXTRACTOR_URL=http://localhost:28001" >> /etc/environment
    echo "NEXT_PUBLIC_CORPUS_INDEXER_URL=http://localhost:28002" >> /etc/environment
    echo "NEXT_PUBLIC_RELATIONSHIP_ENGINE_URL=http://localhost:28003" >> /etc/environment
  SHELL
end
