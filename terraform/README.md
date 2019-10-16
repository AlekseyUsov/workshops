# Task 0

```
mkdir terraform
cd terraform
cp ~/workshop/terraform/* .
chmod 400 workshop.pem
```

```
cat .gitignore

terraform.tfstate*
.terraform
*.retry
.ssh
```

# Task 1


```
cat ec2.tf

provider "aws" {
  region     = var.region
}

resource "aws_instance" "ansible" {
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "us-east-2a"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "ansible"
  }
}

resource "aws_instance" "managed_node" {
  count             = 3
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "${var.region}${element(var.zones, count.index)}"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "node${count.index+1}"
  }
}

```

```
cat variables.tf

ariable "amis" {
  type = map(string)
  default = {
    "us-east-2" = "ami-03291866"
  }
}

variable "instance_type" {
  default = "t2.medium"
}

variable "region" {
  default = "us-east-2"
}

variable "zones" {
  default = ["a", "b", "c"]
}

variable "username" {
  default = "ec2-user"
}

variable "key_name" {
  default = "workshop"
}
```

```
cat versions.tf

terraform {
  required_version = ">= 0.12"
}
```

```
terraform init
terraform validate
terraform plan
terraform apply
```

Create GitHub repo, initialize local working copy, commit, and push changes:
```
git init
git add .
git commit -m 'first commit'
git remote add origin <repo_url>
git push -u origin master
```

# Task 2

```
cat ec2.tf

provider "aws" {
  region     = var.region
}

resource "aws_instance" "ansible" {
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "us-east-2a"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags = {
    Name                                                     = "ansible"
  }
  provisioner "file" {
    content = tls_private_key.ansible.private_key_pem
    destination = ".ssh/id_ecdsa"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file("workshop.pem")
    }
  }
  provisioner "remote-exec" {
    inline = [
      "chmod 0400 .ssh/id_ecdsa"
    ]
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file("workshop.pem")
    }
  }
}

resource "aws_instance" "managed_node" {
  count             = 3
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "${var.region}${element(var.zones, count.index)}"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags = {
    Name                                                     = "node${count.index+1}"
  }
  provisioner "file" {
    content = tls_private_key.ansible.public_key_openssh
    destination = ".ssh/authorized_keys"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file("workshop.pem")
    }
  }
}

resource "tls_private_key" "ansible" {
  algorithm = "RSA"
}
```

```
terraform init
terraform validate
terraform plan
terraform apply
terraform taint aws_instance.ansible
terraform taint aws_instance.managed_node[0]
terraform taint aws_instance.managed_node[1]
terraform taint aws_instance.managed_node[2]
terraform apply
```

```
git commit -am 'Added private key resource'
git push
```

# Task 3

```
cat ec2.tf

provider "aws" {
  region     = var.region
}

resource "aws_instance" "ansible" {
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "us-east-2a"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "ansible"
  }
  provisioner "file" {
    content = tls_private_key.ansible.private_key_pem
    destination = ".ssh/id_ecdsa"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
  provisioner "remote-exec" {
    inline = [
      "chmod 0400 .ssh/id_ecdsa"
    ]
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}

resource "aws_instance" "managed_node" {
  count             = 3
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "${var.region}${element(var.zones, count.index)}"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "node${count.index+1}"
  }
  provisioner "file" {
    content = tls_private_key.ansible.public_key_openssh
    destination = ".ssh/authorized_keys"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}

resource "tls_private_key" "ansible" {
  algorithm = "RSA"
}

locals {
  private_key = join(".",[var.key_name,"pem"])
  inventory   = "/tmp/inventory"
}

resource "local_file" "ansible_private_key" {
  sensitive_content  = tls_private_key.ansible.private_key_pem
  filename           = ".ssh/ansible.pem"
  file_permission    = "0400"
}
```

```
terraform init
terraform validate
terraform plan
terraform apply
```

```
git commit -am 'Added local_file resource for private key'
git push
```

# Task 4

```
cat inventory.tf

data "template_file" "inventory" {
  template = file("${path.module}/inventory.tpl")
  vars = {
    node_ips    = join(",",aws_instance.managed_node.*.public_ip)
    control_ip  = aws_instance.ansible.public_ip
  }
}

resource "null_resource" "inventory" {
  provisioner "file" {
    content     = data.template_file.inventory.rendered
    destination = local.inventory
    connection {
      type = "ssh"
      host = aws_instance.ansible.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}
```

```
cat inventory.tpl

[web]
%{ for ip in split(",", node_ips) ~}
node${tonumber(index(split(",", node_ips), ip))+1}    ansible_host=${ip}
%{ endfor ~}

[control]
ansible ansible_host=${control_ip}
```

```
terraform init
terraform validate
terraform plan
terraform apply
```

```
git add .
git commit -m 'Added inventory resource'
git push
```

# Task 5

```
cat workshop.yml

---
- name: Provision workshop
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
  - name: Run terraform
    terraform:
      project_path: "{{playbook_dir}}"
      force_init: yes
      state: "{{state}}"
```

Add the following parameter to Ansible configuration file to disable retry files in case you make a mistake and your playbook fails:
```
cat /etc/ansible/ansible.cfg

[defaults]
retry_files_enabled = false
```

As a case-by-case solution, you can prepend Ansible commands with `ANSIBLE_RETRY_FILES_ENABLED=false` instead.

Explore the following commands:
```
ansible-doc
ansible-config
ansible-playbook
```

Run the following commands to check the syntax, run in check mode, and run the playbook, respectivelly:
```
ansible-playbook --syntax-check workshop.yml
ansible-playbook workshop.yml -e state=present -CD
ansible-playbook workshop.yml -e state=present
```

Note that not all Ansible modules support check mode.

Commit the changes:
```
git add .
git commit -m 'Added playbook wrapper'
git push
```

# Task 6

```
cat ec2.tf

provider "aws" {
  region     = var.region
}

resource "aws_instance" "ansible" {
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "us-east-2a"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "ansible"
  }
  provisioner "file" {
    content = tls_private_key.ansible.private_key_pem
    destination = ".ssh/id_ecdsa"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
  provisioner "remote-exec" {
    inline = [
      "chmod 0400 .ssh/id_ecdsa"
    ]
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}

resource "aws_instance" "managed_node" {
  count             = 3
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "${var.region}${element(var.zones, count.index)}"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags   = {
    Name = "node${count.index+1}"
  }
  provisioner "file" {
    content = tls_private_key.ansible.public_key_openssh
    destination = ".ssh/authorized_keys"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}

resource "tls_private_key" "ansible" {
  algorithm = "RSA"
}

locals {
  private_key = join(".",[var.key_name,"pem"])
  inventory   = "/tmp/inventory"
}

resource "local_file" "ansible_private_key" {
  sensitive_content  = tls_private_key.ansible.private_key_pem
  filename           = ".ssh/ansible.pem"
  file_permission    = "0400"
}

output "ansible_control_node_connection_info" {
  value = "ssh -i ${local.private_key} ${var.username}@${aws_instance.ansible.public_ip}"
}

output "ansible_inventory" {
  value = local.inventory
}

```

```
terraform outputs
```

```
cat workshop.yml

---
- name: Provision workshop
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
  - name: Run terraform
    terraform:
      project_path: "{{playbook_dir}}"
      force_init: yes
      state: "{{state}}"
    register: tf

  - name: Display Ansible control node connection info
    debug:
      msg: "To connect to your ansible control node, use: {{tf.outputs.ansible_control_node_connection_info.value}}"
    when: state == "present"

  - name: Display Ansible inventory location
    debug:
      msg: "Ansible inventory was copied to ansible node control node at: {{tf.outputs.ansible_inventory.value}}"
    when: state == "present"
```

```
ansible-playbook workshop.yml -e state=present
```

```
git commit -am 'Added outputs'
git push
```

# Task 7

```
cat ec2.tf

provider "aws" {
  region     = var.region
}

resource "aws_instance" "ansible" {
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "us-east-2a"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags = {
    Name                                                     = "ansible"
  }
  provisioner "file" {
    content = tls_private_key.ansible.private_key_pem
    destination = ".ssh/id_ecdsa"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
  provisioner "remote-exec" {
    inline = [
      "chmod 0400 .ssh/id_ecdsa"
    ]
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
  provisioner "remote-exec" {
    when = "destroy"
    inline = ["sudo subscription-manager unregister"]
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
    on_failure = "continue"
  }
}

resource "aws_instance" "managed_node" {
  count             = 3
  ami               = var.amis[var.region]
  instance_type     = var.instance_type
  availability_zone = "${var.region}${element(var.zones, count.index)}"
  key_name          = var.key_name
  security_groups   = ["default"]
  tags = {
    Name                                                     = "node${count.index+1}"
  }
  provisioner "file" {
    content = tls_private_key.ansible.public_key_openssh
    destination = ".ssh/authorized_keys"
    connection {
      type = "ssh"
      host = self.public_ip
      user = var.username
      private_key = file(local.private_key)
    }
  }
}

resource "tls_private_key" "ansible" {
  algorithm = "RSA"
}

locals {
  private_key = join(".",[var.key_name,"pem"])
  inventory   = "/tmp/inventory"
}

resource "local_file" "ansible_private_key" {
  sensitive_content  = tls_private_key.ansible.private_key_pem
  filename           = ".ssh/ansible.pem"
  file_permission    = "0400"
}

output "ansible_control_node_connection_info" {
  value = "ssh -i ${local.private_key} ${var.username}@${aws_instance.ansible.public_ip}"
}

output "ansible_inventory" {
  value = local.inventory
}

output "ansible_control_node_ip" {
  value = aws_instance.ansible.public_ip
}
```

```
cat workshop.yml

---
- name: Provision workshop
  hosts: localhost
  connection: local
  gather_facts: no
  vars_files:
    - tower.yml
  tasks:
  - name: Run terraform
    terraform:
      project_path: "{{playbook_dir}}"
      force_init: yes
      state: "{{state}}"
    register: tf

  - name: Display Ansible control node connection info
    debug:
      msg: "To connect to your ansible control node, use: {{tf.outputs.ansible_control_node_connection_info.value}}"
    when: state == "present"

  - name: Display Ansible inventory location
    debug:
      msg: "Ansible inventory was copied to ansible node control node at: {{tf.outputs.ansible_inventory.value}}"
    when: state == "present"

  - name: Create Ansible Tower inventory
    tower_inventory:
      name: \<choose_unique\>
      organization: \<your_organization\>
      tower_host: "{{tower_host}}"
      tower_username: "{{tower_username}}"
      tower_password: "{{tower_password}}"
      tower_verify_ssl: no
      state: "{{state}}"

  - name: Add ansible control node to Ansible Tower inventory
    tower_host:
      name: "ansible"
      inventory: \<name_from_previous_task\>
      tower_host: "{{tower_host}}"
      tower_username: "{{tower_username}}"
      tower_password: "{{tower_password}}"
      tower_verify_ssl: no
      variables:
        ansible_host: "{{tf.outputs.ansible_control_node_ip.value}}"
    when: state == "present"

  - name: Create Ansible Tower project
    tower_project:
      name: \<choose_unique\>
      organization: \<your_organization\>
      scm_type: git
      scm_update_on_launch: yes
      scm_clean: yes
      scm_url: https://github.com/AlekseyUsov/workshop-test.git
      tower_host: "{{tower_host}}"
      tower_username: "{{tower_username}}"
      tower_password: "{{tower_password}}"
      tower_verify_ssl: no
      state: "{{state}}"

  - name: Give project SCM update some time to finish
    pause:
      seconds: 30
    when: state == "present"

  - name: Create Ansible Tower job template
    tower_job_template:
      become_enabled: yes
      credential: workshop-aws-ssh
      vault_credential: redhat
      inventory: \<created_inventory\>
      job_type: run
      name: \<choose_unique\>
      playbook: register.yml
      project: \<name_from_previous_task\>
      tower_host: "{{tower_host}}"
      tower_username: "{{tower_username}}"
      tower_password: "{{tower_password}}"
      tower_verify_ssl: no
    when: state == "present"

  - name: Launch job template to register provisioned nodes with Red Hat
    tower_job_launch:
      job_template: "<name_from_previous_task>"
      job_type: run
      tower_host: "{{tower_host}}"
      tower_username: "{{tower_username}}"
      tower_password: "{{tower_password}}"
      tower_verify_ssl: no
      credential: workshop-aws-ssh
      inventory: \<created_inventory\>
    when: state == "present"
```

Prepare `tower.yml`:

```
cat tower.yml

tower_host: tower.demo.li9.com
tower_username: <your_tower_username>
tower_password: <your_tower_password>
```

Encrypt the file, as it stores sensitive data:
```
ansible-vault encrypt tower.yml
```

Explore other subcommands of `ansible-vault`, like `view`, `decrypt`, and `rekey`.

Make sure to take note of the password, as you will use it later to create a credential in Ansible Tower.

Prepare the playbook for registering and subscribing Ansible control node:
```
cat register.yml

---
- name: Subscribe
  hosts: ansible
  gather_facts: no
  vars_files:
    - redhat.yml
  tasks:
  - name: Subscribe to Red Hat
    redhat_subscription:
      username: "{{username}}"
      password: "{{password}}"
      pool: "{{pool}}"

  - name: Enable ansible repository
    rhsm_repository:
      name: rhel-7-server-ansible-2.8-rpms

  - name: Install Ansible
    yum:
      name: ansible
    state: present
```

Note that `redhat.yml` file is already prepared for you, but you were not provided with a password to decrypt it. This is by design, as the password is stored encrypted in Ansible Tower available for your use.

In order to avoid using subscriptions unnecessarily, only control node is subscribed, as RHEL RPMs are available on managed nodes via Red Hat Cloud Infrastructure in AWS.

```
git add .
git commit -m 'Added register.yml, destroy provisioner, and tower_* tasks'
git push
```

Cleanup:
```
terraform destroy
```

# Task 8: Ansible Tower

It's recommended to use first and last name or any other personalized string in Ansible Tower names of objects to guarantee uniqueness. For example, you may choose to use `FirstLast_Provision` scheme for assets you create manually and `FirstLast_Register` for assets created by Ansible (`tower.yml`).

1. Create a Vault credential with an arbitrary name in your organization. The credential must store the password you used to encrypt `tower.yml` file.
2. Create a Git-managed project in your organization with options `Clean` and `Update Revision on Launch` enabled.
3. Create a job template with the following parameters:

    3.1. Project: <your_project_from_2>

    3.2. Inventory: Demo Inventory

    3.3. Playbook: workshop.yml

    3.4. Credential: tower (type Vault)
4. Create survey with the following parameters:

    4.1. Prompt: \<arbitrary\>

    4.2. Answer variable name: state (must match its counterpart from the playbook you selected in 3.3)

    4.3. Asnwer type: Text

    4.4. Minimum length: 6

    4.5. Maximum length: 7

    4.6. Default answer: present
