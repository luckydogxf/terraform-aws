# terraform-aws
Privision AWS by terraform
 
创建AWS/openstack/Azure等，一个开源的云平台管理工具，假如你用了Azure/openstack等混杂云环境，可以考虑用这个，官方文档非常的简洁。下边例子是创建VPC/subnet/若干个EC2，后端的subnet的ec2通过nat-box上网并且安装puppet/zabbix的yum repo。
 
  注意provisioner有个问题：创建EIP和remote-exec, 由于remote-exec需要EIP,但terraform会等remote-exec执行完才得到eip，这显然是不可能的，需要depends_on来指定先后顺序和用null_resource, 这这里也有说明。
  
```
provider "aws" {
  access_key = "${var.access_key}"
  secret_key = "${var.secret_key}"
  region = "${var.region}"
}
resource "aws_vpc" "demo" {
  cidr_block = "10.0.0.0/16"
  tags {
    Name = "demo"
  }
}
resource "aws_subnet" "public" {
  vpc_id = "${aws_vpc.demo.id}"
  cidr_block = "10.0.10.0/24"
  tags {
    Name = "demo-public"
  }
}
resource "aws_subnet" "private" {
  vpc_id = "${aws_vpc.demo.id}"
  cidr_block = "10.0.80.0/24"
  tags {
    Name = "demo-private"
  }
}
resource "aws_internet_gateway" "default" {
  vpc_id = "${aws_vpc.demo.id}"
  tags {
    Name = "default-IGW"
  }
}
resource "aws_route_table" "internet_access" {
  vpc_id = "${aws_vpc.demo.id}"
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.default.id}"
  }
  tags {
    Name = "igw"
  }
}
resource "aws_route_table_association" "public" {
  subnet_id = "${aws_subnet.public.id}"
  route_table_id = "${aws_route_table.internet_access.id}"
}
resource "aws_route_table" "private" {
  vpc_id = "${aws_vpc.demo.id}"
  route {
    cidr_block = "0.0.0.0/0"
    instance_id = "${aws_instance.nat.id}"
  }
}
resource "aws_route_table_association" "private" {
  subnet_id = "${aws_subnet.private.id}"
  route_table_id = "${aws_route_table.private.id}"
}
resource "aws_elb" "web" {
  name = "pcf-elb"
  subnets = ["${aws_subnet.private.id}"]
  security_groups = ["${aws_security_group.elb.id}"]
  listener {
    instance_port = 80
    instance_protocol = "http"
    lb_port = 80
    lb_protocol = "http"
  }
  health_check {
    healthy_threshold = 2
    unhealthy_threshold = 2
    timeout = 3
    target = "HTTP:80/"
    interval = 30
  }
  # The instance is registered automatically
  # instances = ["${aws_instance.web.id}"]
  cross_zone_load_balancing = true
  idle_timeout = 400
  connection_draining = true
  connection_draining_timeout = 400
  tags {
    Name = "test-pcf"
  }
}
/*
   resource "aws_lb_cookie_stickiness_policy" "default" {
   name = "lbpolicy"
   load_balancer = "${aws_elb.web.id}"
   lb_port = 80
   cookie_expiration_period = 600
   } */
resource "aws_security_group" "elb" {
  name = "example_elb"
  description = "Nothing more"
  vpc_id = "${aws_vpc.demo.id}"
  # HTTP access from anywhere
  ingress {
    from_port = 80
    to_port = 80
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  # outbound internet access
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
resource "aws_eip" "demo" {
  instance = "${aws_instance.zabbix-server.id}"
  depends_on = ["aws_instance.zabbix-server"]
  vpc = true
}
resource "aws_instance" "zabbix-server" {
  ami = "${lookup(var.aws_opsman_ami,var.region)}"
  instance_type = "m3.medium"
  key_name = "${var.key_name}"
  subnet_id = "${aws_subnet.public.id}"
  vpc_security_group_ids = ["${aws_security_group.all_pass.id}"]
  tags {
    Name = "zabbix-server"
  }
}
resource "null_resource" "server-provisioning" {
  triggers {
    instance = "${aws_instance.zabbix-server.id}"
  }
  connection {
    host = "${aws_eip.demo.public_ip}"
    user = "ec2-user"
    timeout = "5m"
    private_key = "${file("./hfu.pem")}"
  }
  provisioner "file" {
    # puppet.conf
    content = <<EOF
[main]
certname = ${var.puppetname}
server = ${var.puppetname}
strict_variables = true
masterport = 8140
[master]
node_terminus = exec
external_nodes = /etc/puppetlabs/enc.rb
autosign = true
vardir = /opt/puppetlabs/server/data/puppetserver
logdir = /var/log/puppetlabs/puppetserver
rundir = /var/run/puppetlabs/puppetserver
pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
codedir = /etc/puppetlabs/code
[agent]
puppetmaster = ${var.puppetname}
environment = production
runinterval = 1h
EOF
    destination = "/tmp/puppetmaster"
  }
  provisioner "file" {
    # external node classifier
    content =<<EOF
#!/usr/bin/env ruby
require 'yaml'
node = ARGV[0]
# agent FQDN : ${app}.${env}.${Location || IDC}, e.g web01.prod.bjcnc
if node =~ /(^[a-zA-Z]+)(\d+)\.(\S+)\.(\S+)$/
    hostname = $1
    env = $3
    config = YAML.load_file('defaults.yaml')[env]
    print YAML.dump(config.fetch(hostname))
    exit 0
else
    config = YAML.load_file('defaults.yaml')
    print YAML.dump(config.fetch('base'))
    exit 0
end
EOF
    destination = "/tmp/enc"
  }
  provisioner "remote-exec" {
    inline = [
      # reserve EC2 for future usage.
      "sudo echo ${join(",", aws_instance.trial.*.private_ip)} >>/tmp/ec2_list" ,
      "sudo bash -c 'echo puppet \"${aws_instance.zabbix-server.private_ip} puppet.xxx.com\" >> /etc/hosts'" ,
      "sudo rpm -ivh ",
      "sudo rpm -Uvh ",
      "sudo sed -i 's/enabled=0/enabled=1/g' /etc/yum.repos.d/redhat-rhui.repo",
      "sudo yum -y install ruby mariadb mariadb-server vim-enhanced telnet zabbix-server-mysql zabbix-web-mysql puppetserver 1 >/dev/null ",
      "sudo mv /tmp/puppetmaster /etc/puppetlabs/puppet/puppet.conf",
      "sudo mv /tmp/enc /etc/puppetlabs/enc.rb",
      "sudo chmod a+x /etc/puppetlabs/enc.rb",
      # start puppetmaster
# "sudo systemctl start puppetserver",
    ]
  }
}
resource "aws_instance" "trial" {
  count = 1
  ami = "${var.other_ami}"
  instance_type = "t2.micro"
  key_name = "${var.key_name}"
  subnet_id = "${aws_subnet.private.id}"
  vpc_security_group_ids = ["${aws_security_group.all_pass.id}"]
  tags {
    # generate sequential tag , Name = "${format("web-%03d", count.index)}"
    Name = "${lookup(var.instance_tag, count.index)}"
  }
  provisioner "local-exec" {
    /*
    To reference attributes of your own resource, the syntax is self.ATTRIBUTE. For example ${self.private_ip_address} will interpolate that resource's private IP address. Note that this is only allowed/valid within provisioners.
    */
    command = "echo ${self.private_ip} > private_ips"
  }
  connection {
    host = "${self.private_ip}"
    bastion_host = "${aws_eip.demo.public_ip}"
    user = "ec2-user"
    timeout = "5m"
    bastion_port = "22"
    bastion_private_key = "${file("./hfu.pem")}"
  }
  provisioner "file" {
    # puppet.conf of agent
    content = <
[agent]
puppetmaster = ${var.puppetname}
environment = production
runinterval = 1h
EOF
    destination = "/tmp/init_env.sh"
  }
  provisioner "remote-exec" {
    inline = [
      "sudo bash -c 'echo puppet \"${aws_instance.zabbix-server.private_ip} puppet.xxx.com\" >> /etc/hosts

```

#variables.tf
```
variable "access_key" {
  default = "xxxx"
}
variable "secret_key" {
  default = "xxxxxxx"
}
variable "key_name" {
  default = "hfu"
}
variable "region" {
  default = "cn-north-1"
}
variable "aws_opsman_ami" {
  type = "map"
  default = {
    cn-north-1 = "ami-52d1183f"
    us-east-1 = "ami-xx"
  }
}
variable "other_ami" {
    default = "ami-52d1183f"
}
variable "aws_nat_ami" {
  type = "map"
  default = {
    cn-north-1 = "ami-1848da21"
    us-east-1 = "ami-xxx"
  }
}
variable "puppetname" {
    default = "puppet.xxx.com"
}
variable "instance_tag" {
  default = {
    "0" = "client"
  }
}
```
需要设置的变量都在variables.tf里，然后直接terrafrom apply, 详细的输出可以告诉你哪些资源被创建了。
安装完毕，puppetmaster，zabbix-server以及agent设置好了。

至于层次结构跟`cloudformation`差不多，只不过它不是json文件，这个要注意。其他什么input/output/provider/provisioner也大同小异。
