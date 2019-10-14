Terraform configuration for deploying OpenShift on AWS

# Usage
To deploy infrastructure, run:
```
terraform init
AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY> AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY> terraform apply
```

To install OpenShift 3.9, run:
```
ansible-playbook -i ansible/files/inventory ansible/prepare_openshift_hosts.yml --private-key <Path to your private SSH key>
```

To destroy infrastructure, run:
```
AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY> AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_KEY> terraform destroy
```

# Variables
|Name                |Description                                  |Default Value|
|------------------- |---------------------------------------------|----------|
|var.key_name        |Name of your SSH keypair (for example, jdoe) |-         |
|var.private_key     |Path to your private SSH key                 |-         |
|var.zone_id         |AWS Route53 DNS zone ID                      |-         |
|provider.aws.region |AWS region to create instances in            |us-east-1 |

# Note
You may need to run destroy operation twice to get rid of all instances, or delete them manually after destroy operation times out for the 1st time or returns error about systems not being registered after 2nd run. The reason is when EIPs are deleted, instances become inaccessible because public_ip attribute contains EIPs, and during the 2nd run it contains default public IPs. I'm yet to figure this out, but for now, be aware of this.
