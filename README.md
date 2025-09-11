provider "aws" {
  region = "ap-south-1"
}

resource "aws_vpc" "monarch" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "monarch" }
}

resource "aws_subnet" "public_a" {
  vpc_id     = aws_vpc.monarch.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = { Name = "public-a" }
}

resource "aws_subnet" "public_b" {
  vpc_id     = aws_vpc.monarch.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "ap-south-1b"
  map_public_ip_on_launch = true
  tags = { Name = "public-b" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.monarch.id
  tags = { Name = "nova" }
}

resource "aws_route_table" "soul" {
  vpc_id = aws_vpc.monarch.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "soul" }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.soul.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.soul.id
}

resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "Allow HTTP"
  vpc_id      = aws_vpc.monarch.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "alb-sg" }
}

resource "aws_security_group" "ec2_sg" {
  name        = "ec2-sg"
  vpc_id      = aws_vpc.monarch.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "ec2-sg" }
}

resource "aws_launch_template" "web_lt" {
  name_prefix   = "web-lt"
  image_id      = "ami-0a7cf821b91bcccbc" # Amazon Linux 2 (verify latest)
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.ec2_sg.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum install -y httpd
              systemctl enable httpd
              systemctl start httpd
              echo "<h1>Hello from Auto Scaling Group</h1>" > /var/www/html/index.html
              EOF
            )
}

resource "aws_autoscaling_group" "web_asg" {
  name                      = "web-asg"
  min_size                  = 1
  max_size                  = 3
  desired_capacity          = 1
  vpc_zone_identifier       = [aws_subnet.public_a.id, aws_subnet.public_b.id]
  launch_template {
    id      = aws_launch_template.web_lt.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.web_tg.arn]
  health_check_type = "EC2"
  health_check_grace_period = 300

  tag {
    key                 = "Name"
    value               = "web-instance"
    propagate_at_launch = true
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb" "web_alb" {
  name               = "web-alb"
  load_balancer_type = "application"
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]
  security_groups    = [aws_security_group.alb_sg.id]
  tags = { Name = "web-alb" }
}

resource "aws_lb_target_group" "web_tg" {
  name     = "web-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.monarch.id
  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 3
    unhealthy_threshold = 3
    matcher             = "200"
  }
}

resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.web_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_tg.arn
  }
}