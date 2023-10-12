Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  probe_script = <<-SCRIPT
#!/bin/bash
while true; do
  if ping -c 1 8.8.8.8 > /dev/null 2>&1; then
      echo "ISP is up" | nc -l -p 1500 -q 1
  else
      echo "ISP is down" | nc -l -p 1500 -q 1
  fi
done
SCRIPT

  probe_service = <<-SERVICE
[Unit]
Description=Probe Service

[Service]
ExecStart=/home/vagrant/probe.sh
Restart=always

[Install]
WantedBy=multi-user.target
SERVICE

  # ISP 1 VM
  config.vm.define "isp1" do |isp1|
      isp1.vm.network "private_network", ip: "192.168.56.101"
      isp1.vm.network "public_network"
      isp1.vm.provision "shell", inline: <<-SHELL
        sudo ip route del default via 10.0.2.2 dev enp0s3
        echo 1 > /proc/sys/net/ipv4/ip_forward
        iptables -t nat -A POSTROUTING -o enp0s9 -j MASQUERADE
        echo '#{probe_script}' > /home/vagrant/probe.sh
        chmod +x /home/vagrant/probe.sh
        echo '#{probe_service}' | sudo tee /etc/systemd/system/probe.service
        sudo systemctl daemon-reload
        sudo systemctl enable probe.service
        sudo systemctl start probe.service
      SHELL
    end

  # ISP 2 VM
  config.vm.define "isp2" do |isp2|
      isp2.vm.network "private_network", ip: "192.168.56.102"
      isp2.vm.network "public_network"
      isp2.vm.provision "shell", inline: <<-SHELL
        sudo ip route del default via 10.0.2.2 dev enp0s3
        echo 1 > /proc/sys/net/ipv4/ip_forward
        iptables -t nat -A POSTROUTING -o enp0s9 -j MASQUERADE
        echo '#{probe_script}' > /home/vagrant/probe.sh
        chmod +x /home/vagrant/probe.sh
        echo '#{probe_service}' | sudo tee /etc/systemd/system/probe.service
        sudo systemctl daemon-reload
        sudo systemctl enable probe.service
        sudo systemctl start probe.service
      SHELL
    end
  
    # Client VM
    config.vm.define "client" do |client|
      client.vm.network "private_network", ip: "192.168.56.200"
      client.vm.provision "shell", inline: <<-SHELL
        sudo ip route del default via 10.0.2.2
        sudo ip route add default via 192.168.56.101 metric 100
        sudo ip route add default via 192.168.56.102 metric 200
  
        cat <<EOF > /home/vagrant/failover.sh
  #!/bin/bash
  while true; do
    isp1_status=\\$(echo -n | nc 192.168.56.101 1500)
    isp2_status=\\$(echo -n | nc 192.168.56.102 1500)
  
    if [[ \\$isp1_status == "ISP is up" ]]; then
      sudo ip route replace default via 192.168.56.101 metric 100
      sudo ip route replace default via 192.168.56.102 metric 200
    elif [[ \\$isp2_status == "ISP is up" ]]; then
      sudo ip route replace default via 192.168.56.101 metric 200
      sudo ip route replace default via 192.168.56.102 metric 100
    else
      echo "Neither ISP is available, maintaining current settings"
    fi
  
    sleep 1
  done
  EOF
  
        chmod +x /home/vagrant/failover.sh
  
        cat <<EOF | sudo tee /etc/systemd/system/failover.service
  [Unit]
  Description=Failover Script
  
  [Service]
  ExecStart=/home/vagrant/failover.sh
  Restart=always
  User=vagrant
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  sudo systemctl daemon-reload
  sudo systemctl enable failover.service
  sudo systemctl start failover.service
  
  sudo apt update && sudo apt install -y traceroute
  SHELL
    end
  end
  