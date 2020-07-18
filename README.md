provider "aws" {
  region     = "ap-south-1"
  profile   = "myvimal"
}

variable "enter_ur_key_name" {
          type = string
          default = "mykey1111"
}


resource "aws_vpc" "main" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "main"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = "${aws_vpc.main.id}"
  tags = {
    Name = "gw"
  }
  depends_on  = [aws_vpc.main,]
}




/*public subnet*/
resource "aws_subnet" "subnet_1" {
  vpc_id     = "${aws_vpc.main.id}"
  cidr_block = "10.0.1.0/24"
  availability_zone = "ap-south-1a"

  tags = {
    Name = "Main"
  }
  depends_on  = [aws_vpc.main,]
}


resource "aws_route_table" "eu-west-1a-public" {
    vpc_id = "${aws_vpc.main.id}"
     
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.gw.id}"
          
    }   
    depends_on  = [aws_vpc.main,]
}


resource "aws_route_table_association" "eu-west-1a-public" {
    subnet_id = "${aws_subnet.subnet_1.id}"
    route_table_id = "${aws_route_table.eu-west-1a-public.id}"
     depends_on = [ aws_route_table.eu-west-1a-public ]
}




/*private subnet*/

resource "aws_subnet" "subnet_2" {
  vpc_id     = "${aws_vpc.main.id}"
  cidr_block = "10.0.0.0/24"
  availability_zone = "ap-south-1b"
  tags = {
    Name = "subnet_2"
  }
  depends_on  = [aws_vpc.main,]
}


/*this security group for public/webserver*/



resource "aws_security_group" "nat" {
    name = "vpc_nat"
    description = "Allow traffic to pass from the private subnet to the internet"

    ingress {
        from_port = 80
        to_port = 80
        protocol = "tcp"
        cidr_blocks = [ "10.0.1.0/24" ]
    }
    ingress {
        from_port = 443
        to_port = 443
        protocol = "tcp"
        cidr_blocks = [ "10.0.1.0/24" ]
    }
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port = -1
        to_port = -1
        protocol = "icmp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
        from_port = 80
        to_port = 80
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    egress {
        from_port = 443
        to_port = 443
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    egress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = [ "10.0.0.0/16" ]
    }
    egress {
        from_port = -1
        to_port = -1
        protocol = "icmp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    vpc_id = "${aws_vpc.main.id}"

}



resource "aws_instance"  "myin" {
  ami           = "ami-07a8c73a650069cf3"
  instance_type = "t2.micro"
  key_name      =  var.enter_ur_key_name
  vpc_security_group_ids  =  ["${aws_security_group.nat.id}"] 
  subnet_id       =     "${aws_subnet.subnet_1.id}"
  associate_public_ip_address = true
  tags = {
    Name = "webos"
  }
   depends_on  = [aws_vpc.main,]
} 


/*
resource "aws_eip" "Public_webos_ip" {
    instance = "${aws_instance.myin.id}"
    vpc = true
}

*/



/* this security group for myaql_server*/

 resource "aws_security_group" "web" {
    name = "vpc_web"
    description = "Allow incoming HTTP connections."

    ingress {
        from_port = 80
        to_port = 80
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port = 443
        to_port = 443
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port = -1
        to_port = -1
        protocol = "icmp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress { # MySQL
        from_port = 3306
        to_port = 3306
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    vpc_id = "${aws_vpc.main.id}"
}


resource "aws_instance"  "myin12" {
  ami           = "ami-07a8c73a650069cf3"
  instance_type = "t2.micro"
  key_name      =  var.enter_ur_key_name
  vpc_security_group_ids  =  ["${aws_security_group.nat.id}"] 
  subnet_id       =     "${aws_subnet.subnet_2.id}"
  tags = {
    Name = "mysql"
  }
  # depends_on = ["aws_nat_gateway.nat_gw"]
} 


/*
resource "aws_eip" "Public_mysql_ip" {
    instance = "aws_instance.myin12.id"
    vpc = true
   # depends_on = ["aws_nat_gateway.nat_gw"]
}


resource "aws_nat_gateway" "nat_gw" {
  allocation_id = "aws_eip.Public_mysql_ip.id"
  subnet_id     = "${aws_subnet.subnet_1.id}"
  tags = {
    Name = "gw NAT"
  }
}

*/



resource "aws_route_table" "route_for_mysql" {
    vpc_id = "${aws_vpc.main.id}"
     
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = "${aws_internet_gateway.gw.id}"
        #nat_gateway_id = "aws_nat_gateway.nat_gw.id"
    }
}


resource "aws_route_table_association" "a" {
  subnet_id      = "${aws_subnet.subnet_2.id}"
  route_table_id = aws_route_table.route_for_mysql.id
}


