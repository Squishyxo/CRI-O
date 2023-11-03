installing CRI-O for Kubernetes
Disable swap and firewall
{

sed -i '/swap/d' /etc/fstab
swapoff -a
systemctl disable --now ufw

}

{

cat >>/etc/modules-load.d/crio.conf<<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

}

export OS=xUbuntu_22.04
export VERSION=1.28
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | gpg --dearmor | tee /usr/share/keyrings/libcontainers-archive-keyring.gpg >/dev/null
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | gpg --dearmor | tee /usr/share/keyrings/cri-o-archive-keyring.gpg >/dev/null
# Add the repositories correctly with signed-by option
echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb [signed-by=/usr/share/keyrings/cri-o-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
# Update the package lists and install cri-o
sudo apt update && sudo apt install -y cri-o cri-o-runc cri-tools

# Check if the directory exists before adding the configuration, create if it does not exist
if [ ! -d /etc/crio/crio.conf.d ]; then
    mkdir -p /etc/crio/crio.conf.d
fi

# Add the CRI-O configuration
cat >/etc/crio/crio.conf.d/02-cgroup-manager.conf<<EOF
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "cgroupfs"
EOF

# Reload systemd manager configuration, enable and start cri-o service
sudo systemctl daemon-reload
sudo systemctl enable --now crio


crictl version
crictl ps
