resource "aws_vpc" "demovpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"
tags = {
  Name = "Demo VPC"
}
}

resource "aws_subnet" "public-subnet-2" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "10.0.0.16/28"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1b"
tags = {
  Name = "public-Sub2"
}
}

resource "aws_subnet" "public-subnet-3" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "10.0.0.32/28"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1a"
tags = {
  Name = "pri-Sub3"
}
}
resource "aws_subnet" "public-subnet-4" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "10.0.0.48/28"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1b"
tags = {
  Name = "pri-Sub4"
}
}
resource "aws_subnet" "public-subnet-5" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "10.0.0.64/28"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1a"
tags = {
  Name = "pri-Sub5"
}
}
resource "aws_subnet" "public-subnet-6" {
  vpc_id                  = "${aws_vpc.demovpc.id}"
  cidr_block             = "10.0.0.80/28"
  map_public_ip_on_launch = true
  availability_zone = "us-east-1b"
tags = {
  Name = "pri-Sub6"
}
}
resource "aws_internet_gateway" "demogateway" {
  vpc_id = "${aws_vpc.demovpc.id}"
  tags = {
    name = "demo igw"
  }
}

resource "aws_route_table" "route" {
  vpc_id = "${aws_vpc.demovpc.id}"
route {
      cidr_block = "0.0.0.0/0"
      gateway_id = "${aws_internet_gateway.demogateway.id}"
  }
tags = {
      Name = "Route to internet"
  }
}
# Associating Route Table
resource "aws_route_table_association" "rt1" {
  subnet_id = "${aws_subnet.public-subnet-1.id}"
  route_table_id = "${aws_route_table.route.id}"
}
# Associating Route Table
resource "aws_route_table_association" "rt2" {
  subnet_id = "${aws_subnet.public-subnet-2.id}"
  route_table_id = "${aws_route_table.route.id}"
}

resource "aws_security_group" "demosg" {
  vpc_id = "${aws_vpc.demovpc.id}"
# Inbound Rules
# HTTP access from anywhere
ingress {
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
# HTTPS access from anywhere
ingress {
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
# SSH access from anywhere
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
# Outbound Rules
# Internet access to anywhere
egress {
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]
}
tags = {
  Name = "Web SG"
}
}

resource "aws_instance" "demoinstance" {
  ami                         = "ami-087c17d1fe0178315"
  instance_type               = "t2.micro"
  key_name                    = "terraform"
  vpc_security_group_ids      = ["${aws_security_group.demosg.id}"]
  subnet_id                   = "${aws_subnet.public-subnet-1.id}"
  associate_public_ip_address = true
  user_data                   = "${file("data.sh")}"
tags = {
  Name = "My Public Instance"
}
}
resource "aws_instance" "demoinstance1" {
  ami                         = "ami-087c17d1fe0178315"
  instance_type               = "t2.micro"
  key_name                    = "terraform"
  vpc_security_group_ids      = ["${aws_security_group.demosg.id}"]
  subnet_id                   = "${aws_subnet.public-subnet-1.id}"
  associate_public_ip_address = true
  user_data                   = "${file("data.sh")}"
tags = {
  Name = "My Public Instance 1"
}
}

resource "aws_security_group" "database-sg" {
  name        = "Database SG"
  description = "Allow inbound traffic from application layer"
  vpc_id      = aws_vpc.demovpc.id
ingress {
  description     = "Allow traffic from application layer"
  from_port       = 3306
  to_port         = 3306
  protocol        = "tcp"
  security_groups = [aws_security_group.demosg.id]
}
egress {
  from_port   = 32768
  to_port     = 65535
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
tags = {
  Name = "Database SG"
}
}

resource "aws_lb" "external-alb" {
  name               = "External-LB"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.demosg.id]
  subnets            = [aws_subnet.public-subnet-1.id, aws_subnet.public-subnet-2.id]
}
resource "aws_lb_target_group" "target-elb" {
  name     = "ALB-TG"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.demovpc.id
}
resource "aws_lb_target_group_attachment" "attachment" {
  target_group_arn = aws_lb_target_group.target-elb.arn
  target_id        = aws_instance.demoinstance.id
  port             = 80
depends_on = [
  aws_instance.demoinstance,
]
}
resource "aws_lb_target_group_attachment" "attachment5" {
  target_group_arn = aws_lb_target_group.target-elb.arn
  target_id        = aws_instance.demoinstance1.id
  port             = 80
depends_on = [
  aws_instance.demoinstance1,
]
}
resource "aws_lb_listener" "external-elb" {
  load_balancer_arn = aws_lb.external-alb.arn
  port              = "80"
  protocol          = "HTTP"
default_action {
  type             = "forward"
  target_group_arn = aws_lb_target_group.target-elb.arn
}
}


resource "aws_db_subnet_group" "default" {
  name       = "main"
  subnet_ids = [aws_subnet.public-subnet-5.id, aws_subnet.public-subnet-6.id]
tags = {
  Name = "My DB subnet group"
}
}
resource "aws_db_instance" "default" {
  allocated_storage      = 10
  db_subnet_group_name   = aws_db_subnet_group.default.id
  engine                 = "mysql"
  engine_version         = "8.0.35"
  instance_class         = "db.m5d.large"
  multi_az               = true
  identifier             = "mydb"
  username               = "username"
  password               = "password"
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.database-sg.id]
}
