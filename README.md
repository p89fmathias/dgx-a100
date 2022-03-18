git clone https://github.com/NVIDIA/deepops.git
cd deepops
git checkout tags/22.01

### fmathias - start - corrigindo a dashboard (memoria)
curl https://raw.githubusercontent.com/p89fmathias/dgx-a100/main/dashboard-gpu-nodes.json -o src/dashboards/gpu-dashboard.json
### fmathias - end

./scripts/setup.sh
pip install markupsafe==2.0.1
exit
./scripts/setup.sh

sed -i 's|^\[all\]|[all]\ndgx-a100    ansible_host=192.168.1.245|gi' config/inventory
sed -i 's|^\[slurm-master\]|[slurm-master]\ndgx-a100|gi' config/inventory
sed -i 's|^\[slurm-node\]|[slurm-node]\ndgx-a100|gi' config/inventory
sed -i 's|^#ansible_user=ubuntu|ansible_user=ubuntu|gi' config/inventory
sed -i 's|^#ansible_ssh_private_key_file|ansible_ssh_private_key_file|gi' config/inventory

sed -i 's|^slurm_enable_nfs_server: true|slurm_enable_nfs_server: false|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_enable_nfs_client_nodes: true|slurm_enable_nfs_client_nodes: false|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_cluster_install_singularity: no|slurm_cluster_install_singularity: yes|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_login_on_compute: false|slurm_login_on_compute: true|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_allow_ssh_user: \[\]|slurm_allow_ssh_user: \[ "ubuntu" \]|gi' config/group_vars/slurm-cluster.yml

### fmathias - start - adicionando o host no grupo slurm-metrics
###                    para o ansible identificar e instalar o prometheus/grafana

sed -i 's|\[slurm-cluster:children\]|[slurm-metric]\ndgx-a100\n\n[slurm-cluster:children]|gi' config/inventory

### fmathias - end

ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
sudo apt-get -y -o "Dpkg::Options::=--force-confdef" -o "Dpkg::Options::=--force-confold" install 'containerd.io=1.4.4-1' 'docker-ce-cli=5:20.10.5~3-0~ubuntu-focal' 'docker-ce=5:20.10.5~3-0~ubuntu-focal' --allow-downgrades
ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
sudo shutdown -r now
ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
