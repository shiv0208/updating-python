---
# Base Amazon Linux 2 tweaks

- name: Install basic packages
  yum: name={{ packages }} state=latest
  vars:
    packages:
    - vim-enhanced
    - net-tools
    - bash-completion
    - elinks
    - git
    - wget
    - unzip
    - telnet
    - nmap-ncat
    - screen
    - deltarpm
    - lsof
    - vim
    - iptraf
    - bind-utils
    - mlocate
    - python3
    - python3-pip
    - "https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm"

- name: Installing epel in Amazon Linux
  command: amazon-linux-extras install epel

- name: Updating pip
  command: "pip install --upgrade pip"

- name: Install pip modules
  pip:
    name: ['boto3', 'botocore', 'awscli', 'pymysql', 'pyOpenSSL']
    extra_args: --upgrade --ignore-installed six
    executable: pip
    
- name: Update python3.8
  command: ansible-playbook python-install.yml


- name: Set the timezone
  timezone:
    name: Australia/Melbourne


- name: Create default root directories
  file:
    path: /root/{{item}}
    state: directory
  with_items:
    - apps
    - backups
    - bin
    - temp
    - scripts

- name: Enable scrolling in screen
  lineinfile:
    dest: /etc/screenrc
    line: "termcapinfo xterm* ti@:te@"
  notify: Restart sshd

- name: Gather EC2 instance metadata
  action: ec2_metadata_facts

- name: Obtain EC2 tags for this instance
  ec2_tag:
    region: "{{ ansible_ec2_placement_region }}"
    resource: "{{ ansible_ec2_instance_id }}"
    state: list
  register: ec2_tags

- name: Set hostname to match EC2 Name
  hostname: name={{ ec2_tags.tags.Name }}

- name: Copy Route53 dns update playbook
  copy: src=update_dns.yml dest="/root/"

- name: Install dns update systemd service
  copy: src=update_dns.service dest="/etc/systemd/system"

- name: Start dns update playbook at boot
  service:
    name: update_dns
    enabled: yes
    state: started
    daemon_reload: yes

- name: Include command logging role
  include_role:
    name: command-logging

- name: Start Amazon SSM agent at boot
  service:
    name: amazon-ssm-agent
    enabled: yes
    state: started
