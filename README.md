#cloud-config
fqdn: site.example.com
hostname: site.example.com
locale: en_GB.UTF-8
timezone: Europe/London

# Package management
package_update: true
package_upgrade: true
package_reboot_if_required: true

packages:
  - wireguard
  - ufw
  - nano
  # K3s dependencies and useful tools
  - curl
  - open-iscsi
  - nfs-common
  - jq
  - apt-transport-https
  - ca-certificates
  - software-properties-common
  - python3-pip
  - gnupg
  - lsb-release
  - net-tools
  - iptables
  - linux-modules-extra-$(uname -r)

# User configuration
users:
  - name: patrick
    groups: sudo, www-data
    lock_passwd: false
    shell: /bin/bash
    # Generated using: perl -e 'print crypt("your-password-here","\$6\$UnXq642da9EfkQfH\$")'
    passwd: $6$UnXq642da9EfkQfH$WyjGklLI/RQy6/ss7/v5MAmrm0MKMnJaRLmx0FYCuLfBvcetHp6Rb.DL9djJ50C56293m2cA11E3PhnTRsa3B.
    ssh-authorized-keys:
      # Replace with your actual public SSH key
      - ssh-ed25519 PLACEHOLDER_SSH_KEY your-comment-here

# SSH Configuration
write_files:
  - path: /etc/ssh/sshd_config
    content: |
      Match Group sftp-only
          ForceCommand internal-sftp
          ChrootDirectory %h/www
          PermitTunnel no
          AllowAgentForwarding no
          AllowTcpForwarding no
          X11Forwarding no
    append: true
    
  # K3s system configurations
  - path: /etc/sysctl.d/99-kubernetes.conf
    content: |
      net.bridge.bridge-nf-call-iptables = 1
      net.ipv4.ip_forward = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      fs.inotify.max_user_instances = 512
      fs.inotify.max_user_watches = 524288
      vm.max_map_count = 262144

  - path: /etc/modules-load.d/k8s.conf
    content: |
      br_netfilter
      overlay
      
  # Script to install k3s
  - path: /home/patrick/scripts/install-k3s.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      # This script will be ready to install k3s when needed
      # Uncomment and modify the following lines when ready to install
      
      # export K3S_TOKEN=your-token
      # export INSTALL_K3S_EXEC="server --cluster-init --disable traefik --disable servicelb --node-name=$(hostname) --tls-san $(hostname -I | cut -d' ' -f1)"
      # curl -sfL https://get.k3s.io | sh -

# System configurations and security
runcmd:
  # Configure nano
  - sed -i 's/[# ]*set tabsize 8/set tabsize 4/g' /etc/nanorc

  # Configure SSH
  - sed -i 's/[#]*Port 22/Port 5822/g' /etc/ssh/sshd_config
  - sed -i 's/#HostKey \/etc\/ssh\/ssh_host_ed25519_key/HostKey \/etc\/ssh\/ssh_host_ed25519_key/g' /etc/ssh/sshd_config
  - sed -i 's/[#]*PermitRootLogin yes/PermitRootLogin prohibit-password/g' /etc/ssh/sshd_config
  - sed -i 's/[#]*PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
  - /etc/init.d/ssh restart

  # Configure Firewall
  - ufw default deny incoming
  - ufw default allow outgoing
  - ufw allow 80/tcp
  - ufw allow 443/tcp
  - ufw limit 5822/tcp
  # K3s required ports
  - ufw allow 6443/tcp  # Kubernetes API Server
  - ufw allow 8472/udp  # Flannel VXLAN
  - ufw allow 10250/tcp # Kubelet metrics
  - ufw allow 2379:2380/tcp # etcd
  - ufw allow 51820/udp # WireGuard
  - ufw enable

  # Load kernel modules for k3s
  - modprobe br_netfilter
  - modprobe overlay
  - sysctl --system

  # Configure .bashrc for patrick
  - sed -i -E 's/secure_path="(.*)"/secure_path="\1:\/home\/patrick\/scripts"/g' /etc/sudoers
  - sed -i 's/($debian_chroot)}\\u\@\\h:\\/($debian_chroot)}\\u\@$(hostname -I | cut -d " " -f 1):\\/g' /home/patrick/.bashrc
  - sed -i 's/($debian_chroot)}\\\[\\033\[01;32m\\\]\\u\@\\h/($debian_chroot)}\\\[\\033\[01;34m\\\]\\u\@$(hostname -I | cut -d " " -f 1)/g' /home/patrick/.bashrc
  - echo "alias update='sudo -- sh -c \"apt update; apt upgrade -y; apt dist-upgrade -y; apt autoremove -y; apt autoclean -y\"'" >> /home/patrick/.bashrc
  - echo "alias shutdown-r='sudo shutdown -r now'" >> /home/patrick/.bashrc
  - echo "export PATH=$PATH:/home/patrick/scripts" >> /home/patrick/.bashrc
  
  # K3s convenience aliases
  - echo "alias k='kubectl'" >> /home/patrick/.bashrc
  - echo "source <(kubectl completion bash)" >> /home/patrick/.bashrc
  - echo "complete -o default -F __start_kubectl k" >> /home/patrick/.bashrc

  # Setup WireGuard placeholder configuration
  - mkdir -p /etc/wireguard
  - |
    cat > /etc/wireguard/wg0.conf << EOF
    # Placeholder WireGuard configuration
    # Replace with actual configuration
    [Interface]
    PrivateKey = YOUR_PRIVATE_KEY
    Address = 10.0.0.1/24
    ListenPort = 51820
    
    [Peer]
    PublicKey = PEER_PUBLIC_KEY
    AllowedIPs = 10.0.0.2/32
    EOF
  - chmod 600 /etc/wireguard/wg0.conf

  # Create scripts directory
  - mkdir -p /home/patrick/scripts
  - chown patrick:patrick /home/patrick/scripts

  # Force password change for patrick on initial login
  - chage -d 0 patrick

  # Configure container runtime settings
  - mkdir -p /etc/systemd/system/docker.service.d
  - |
    cat > /etc/sysctl.d/99-kubernetes-cri.conf << EOF
    net.bridge.bridge-nf-call-iptables  = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF
  - sysctl --system

# Server host keys
ssh_keys:
  ed25519_private: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    # Replace with your actual host key
    -----END OPENSSH PRIVATE KEY-----
  ed25519_public: ssh-ed25519 PLACEHOLDER_HOST_KEY your-host-comment
