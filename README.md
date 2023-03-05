# Kubernetes_Terraform_cluster_deploy_on_EKS_AWS
[ Kubernetes ] Deploy a Kubernetes cluster on EC2 instances using Terraform


``` tf
provider "aws" {
  region                    = "us-east-1"
  shared_config_files       = ["/home/ec2-user/.aws/config"]
  shared_credentials_files  = ["/home/ec2-user/.aws/credentials"]
}

data "aws_availability_zones" "available" {
  state = "available"
}
data "aws_ssm_parameter" "current-ami" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

resource "aws_default_vpc" "default" {
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "Default VPC"
  }
}

resource "aws_subnet" "public_subnet" {
  vpc_id     = aws_default_vpc.default.id
  # cidr_block = "10.0.1.0/24"
  cidr_block = "172.31.98.128/25"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "ansible-public-subnet"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_default_vpc.default.id
  # cidr_block = "10.0.2.0/24"
  cidr_block = "172.31.99.128/25"
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name     = "ansible-private-subnet"
  }
}

resource "aws_security_group" "ec2_master_security_group" {
  name        = "ec2-master-security-group"
  description = "Kunernetes control plane access"
  vpc_id      = aws_default_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 6443
    to_port     = 6443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 2379
    to_port     = 2380
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 10259
    to_port     = 10259
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 10257
    to_port     = 10257
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "ec2_security_group" {
  name        = "ec2-slave-security-group"
  description = "Kunernetes slaves access"
  vpc_id      = aws_default_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }

  ingress {
    from_port   = 30000
    to_port     = 32767
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for the security reasons
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "master_instance" {
  ami           = data.aws_ssm_parameter.current-ami.value
  instance_type = "t2.medium"
  subnet_id     = aws_subnet.private_subnet.id
  vpc_security_group_ids = [aws_security_group.ec2_master_security_group.id]
  associate_public_ip_address = true
  key_name      = "test_delete"
  
  user_data     = <<-EOF-SCRIPT
  #!/bin/bash
  sudo yum install docker -y
  sudo systemctl start docker
  sudo systemctl enable docker
  sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF
  sudo yum install -y kubelet kubeadm kubectl
  sudo systemctl start kubelet.service
  sudo systemctl enable kubelet.service
  sudo kubeadm init --pod-network-cidr=192.168.0.0/16
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  EOF-SCRIPT
  
  
  # provisioner "remote-exec" {
  #   inline = [
  #     "kubeadm token create --print-join-command > /tmp/k8s_join_cmd.sh",
  #     "sudo chmod +x /tmp/k8s_join_cmd.sh"
  #   ]
  # }
  
  tags = {
    Name = "master_instance"
  }
}

resource "aws_instance" "ansible_slave" {
  count = 1
  ami           = data.aws_ssm_parameter.current-ami.value
  instance_type = "t2.medium"
  subnet_id     = aws_subnet.public_subnet.id
  vpc_security_group_ids = [aws_security_group.ec2_security_group.id]
  associate_public_ip_address = true
  key_name      = "test_delete"
  
  user_data     = <<-EOF-SCRIPT
  #!/bin/bash
  sudo yum install docker -y
  sudo systemctl start docker
  sudo systemctl enable docker
  sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
  [kubernetes]
  name=Kubernetes
  baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF
  sudo yum install -y kubelet kubeadm kubectl
  sudo systemctl start kubelet.service
  sudo systemctl enable kubelet.service
  EOF-SCRIPT
  
  # provisioner "remote-exec" {
  #   inline = [
  #     "sudo $(cat /tmp/k8s_join_cmd.sh)"
  #   ]
  # }
  
  tags = {
    Name = "slave_instance${count.index + 1}"
  }
}

output "master_instance_public_ip" {
   value = aws_instance.master_instance.public_ip
 }

output "slaves_ips" {
   value = ["${aws_instance.ansible_slave.*.private_ip}"]
 }

```

<br><br>

this is for troubleshooting using logs:
```
aws ec2 get-console-output --instance-id i-04d6a246a5f4c08ed --region us-east-1

vi /var/log/cloud-init-output.log
```

<br><br>
on a MASTER INSTANCE get a join token:
```
sudo kubeadm token create --print-join-command
```

<br><br>
on each SLAVE INSTANCE join to the cluster (replace with the actual token got on the master):

```
kubeadm join 171.00.00.163:6443 --token 8zfltv.2rrtauvvlczzwlrj --discovery-token-ca-cert-hash sha256:00000000000000000000000000000000000000000000000000
```
