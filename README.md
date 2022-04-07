DGX-A100
===

  This repository is a tiny tutorial to install deepops and slurm-cluster from NVIDIA (https://github.com/NVIDIA/deepops) in a ``DGX OS``.

  ### Pre-requisites

  * Operation System: DGX OS 5.2.0
  * Machine: DGX-A100
  * Installation: slurm-cluster on single machine using **local** install

  ### Getting Started

  * Install DGX OS 5.2.0
  * Add a user called ``ubuntu``
  * Set hostname to ``dgx-a100``

  **NOTE:** be sure to update on commands below, information such as ip address, user, hostname, etc

  ### Steps

  1. Clone deepops repository and checkout to current release

```bash
git clone https://github.com/NVIDIA/deepops.git
cd deepops
git checkout tags/22.01
```

  2. (Optional) I don't know why, but it seems like grafana dashboard do not show memory panel, so I've made a tiny fix

```bash
curl https://raw.githubusercontent.com/p89fmathias/dgx-a100/main/dashboard-gpu-nodes.json -o src/dashboards/gpu-dashboard.json
```

  3. Execute the setup script

```bash
./scripts/setup.sh
```

  4. At this point, the script will fail in an error message of ansible galaxy modules. I just figured out that MarkupSafe 2.1.1 do not work, so we're downgrading this version of module. Its important that we evoke pip under python virtual environment, you should execute this command having an "(env)" in your bash terminal:

```bash
pip install markupsafe==2.0.1
exit
```

  5. After downgrading the module, quit the python virtual environment and run setup again:

```bash
./scripts/setup.sh
```

6. The script should work as intended. Now we going to configure the inventory. Im using sed to automate the commands but you can manually edit the ``config/inventory`` file:

  * Config hostname on **[all]** region of inventory
  * Add the hostname to both **[slurm-master]** and **[slurm-node]** region
  * Uncomment lines on region **[all:vars]** wich tells ansible to use user ubuntu and our private key to log in

```bash
sed -i 's|^\[all\]|[all]\ndgx-a100    ansible_host=192.168.1.245|gi' config/inventory
sed -i 's|^\[slurm-master\]|[slurm-master]\ndgx-a100|gi' config/inventory
sed -i 's|^\[slurm-node\]|[slurm-node]\ndgx-a100|gi' config/inventory
sed -i 's|^#ansible_user=ubuntu|ansible_user=ubuntu|gi' config/inventory
sed -i 's|^#ansible_ssh_private_key_file|ansible_ssh_private_key_file|gi' config/inventory
```


  * Disable NFS server and NFS client config options in ``config/group_vars/slurm-cluster.yml``
  * Configure to install singularity package in ``config/group_vars/slurm-cluster.yml``
  * Configure the environment to allow ssh for our user ``ubuntu`` in ``config/group_vars/slurm-cluster.yml``

```bash
sed -i 's|^slurm_enable_nfs_server: true|slurm_enable_nfs_server: false|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_enable_nfs_client_nodes: true|slurm_enable_nfs_client_nodes: false|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_cluster_install_singularity: no|slurm_cluster_install_singularity: yes|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_login_on_compute: false|slurm_login_on_compute: true|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_allow_ssh_user: \[\]|slurm_allow_ssh_user: \[ "ubuntu" \]|gi' config/group_vars/slurm-cluster.yml
sed -i 's|^slurm_pyxis_version: 0.11.1|slurm_pyxis_version: 0.12.0|gi' config/group_vars/slurm-cluster.yml
```

7. I have discovered that you need to add another region in ``config/invetory`` called **[slurm-metric]** right before **[slurm-cluster:children]** informing ansible to configure prometheus, grafana and node-exporter. Despite having an ansible condition ``hostlist: "{{ slurm_monitoring_group | default('slurm-metric') }}"`` in ``playbooks/slurm-cluster.yml`` and having ``slurm_monitoring_group`` seted up, the playbook didn't installed monitoring tools. So, to fix that, I manually added the region in ``config/inventory``:


```bash
sed -i 's|\[slurm-cluster:children\]|[slurm-metric]\ndgx-a100\n\n[slurm-cluster:children]|gi' config/inventory
```

8. Now we run the playbook:
```bash
ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
```

9. The playbook will fail installing docker. It asks to downgrade some packages, so we do it manually:

```bash
sudo apt-get -y -o "Dpkg::Options::=--force-confdef" -o "Dpkg::Options::=--force-confold" install 'containerd.io=1.4.4-1' 'docker-ce-cli=5:20.10.5~3-0~ubuntu-focal' 'docker-ce=5:20.10.5~3-0~ubuntu-focal' --allow-downgrades
```

10. After completing, we should run the recipe again:

```bash
ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
```

11. Now, the recipe asks us to reboot the machine, because ansible is unable to reboot since we're running local

```bash
sudo shutdown -r now
```

12. After rebooting, we log again using ``ubuntu`` user (we've just created on installation process) and run the recipe again:

```bash
ansible-playbook -K --forks=1 --connection=local -l slurm-cluster playbooks/slurm-cluster.yml
```

13. Now, before start using, we should run a command to update pyxis and config enroot properly:

```bash
ansible-playbook -K --connection=local -i config/inventory playbooks/container/pyxis.yml -v
```

14. After the finishing process, we have now a working slurm cluster and grafana monitoring.
