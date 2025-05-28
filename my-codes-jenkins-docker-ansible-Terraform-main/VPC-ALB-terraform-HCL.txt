provider "aws" {

            access_key = ""
            secret_key = ""
            region = "ap-south-1"
}

#create VPC
resource "aws_vpc" "test" {

        cidr_block = "10.10.0.0/16"
        tags = {

           Name = "my-VPC"
          }
}

#Create subnet-1
resource "aws_subnet" "subnet-1" {

        vpc_id = "${aws_vpc.test.id}"
        cidr_block = "10.10.1.0/24"
        availability_zone = "ap-south-1a"
        tags = {

           Name = "subnet-1"
          }

}

#Create subnet-2
resource "aws_subnet" "subnet-2" {

        vpc_id = "${aws_vpc.test.id}"
        cidr_block = "10.10.2.0/24"
        availability_zone = "ap-south-1b"
        tags = {

           Name = "subnet-2"
          }

}

#Create IGW
resource "aws_internet_gateway" "igw" {

           vpc_id = "${aws_vpc.test.id}"
           tags = {

               Name = "igw"
              }
}
#Create RT
resource "aws_route_table" "Pub-Rt" {

         vpc_id = "${aws_vpc.test.id}"
         tags = {
               Name = "Pub-Rt"
              }

       route {

            cidr_block = "0.0.0.0/0"
            gateway_id = "${aws_internet_gateway.igw.id}"
}
}
#Subnet Association
resource "aws_route_table_association" "one" {

             subnet_id = "${aws_subnet.subnet-1.id}"
                         route_table_id = "${aws_route_table.Pub-Rt.id}"
}

resource "aws_route_table_association" "two" {

             subnet_id = "${aws_subnet.subnet-2.id}"
                         route_table_id = "${aws_route_table.Pub-Rt.id}"
}


#Security Group
resource "aws_security_group" "my-sg" {

            name        = "My-sg-11"
            description = "Allow TLS inbound traffic"
            vpc_id = "${aws_vpc.test.id}"
            tags = { Name = "My-sg" }

        ingress {
        description = "ssh from VPC"
        from_port   = 22
        to_port     = 22
                protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        }

        ingress {
        description = "http from VPC"
        from_port   = 80
        to_port     = 90
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        }

        ingress {
        description = "Custom TCP from VPC"
        from_port   = 8080
        to_port     = 8085
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        }
}
#Launch EC2_instance_1
resource "aws_instance" "test1"{

          #count = 2
           ami = "ami-06e6b44dd2af20ed0"
           instance_type = "t2.micro"
           key_name = "linux-11"
           associate_public_ip_address = "true"
           subnet_id = "${aws_subnet.subnet-1.id}"

           tags = {

                Name = "My-QA"
                Env = "QA"

                  }
}

#Launch EC2_instance_2
resource "aws_instance" "test2"{

           ami = "ami-06e6b44dd2af20ed0"
           instance_type = "t2.micro"
           key_name = "linux-11"
           associate_public_ip_address = "true"
           subnet_id = "${aws_subnet.subnet-2.id}"
           tags = {

                Name = "My-Uat"
                Env = "UAT"

                  }

}

# ELB
resource "aws_lb" "my-lb"{

      name = "my-lb"
      internal = false
      load_balancer_type = "application"
      security_groups = [aws_security_group.my-sg.id]
      subnets = [aws_subnet.subnet-1.id, aws_subnet.subnet-2.id]
      }


# target group
resource "aws_lb_target_group" "Tg" {
  name        = "Tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.test.id
  target_type = "instance"
}

#Register EC2 with Target group
resource "aws_lb_target_group_attachment" "test1" {
  target_group_arn = aws_lb_target_group.Tg.arn
  target_id        = "${aws_instance.test1.id}"
  port             = 80
}

resource "aws_lb_target_group_attachment" "test2" {
  target_group_arn = aws_lb_target_group.Tg.arn
  target_id        = "${aws_instance.test2.id}"
  port             = 80
}


# Create a listener
resource "aws_lb_listener" "listener" {
  load_balancer_arn = "${aws_lb.my-lb.arn}"
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.Tg.arn
  }
}
