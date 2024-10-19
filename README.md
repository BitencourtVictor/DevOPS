Análise Técnica do Código Terraform

1. **Provider:**
   
  ```
  provider "aws" {
    region = "us-east-1"
  }
  ```

Configura a AWS como provedor, definindo a região como `us-east-1`. O Terraform usará essa região para todos os recursos que forem criados.

2. **Variáveis:**
   
  ```
  variable "projeto" {
    description = "Nome do projeto"
    type        = string
    default     = "VExpenses"
  }
  
  variable "candidato" {
    description = "Nome do candidato"
    type        = string
    default     = "SeuNome"
  }
  ```
Duas variáveis são definidas: `projeto` e `candidato`, que têm valores padrão

3. **Chave Privada TLS:**

  ```
  resource "tls_private_key" "ec2_key" {
    algorithm = "RSA"
    rsa_bits  = 2048
  }
  ```
Gera uma chave privada RSA de 2048bits.

4. **Par de Chaves EC2:**

  ```
  resource "aws_key_pair" "ec2_key_pair" {
    key_name   = "${var.projeto}-${var.candidato}-key"
    public_key = tls_private_key.ec2_key.public_key_openssh
  }
  ```
Cria um par de chaves na AWS utilizando a chave pública gerada no recurso anterior. O nome da chave é construído usando as variáveis `projeto` e `candidato`.

5. **VPC:**

  ```
  resource "aws_vpc" "main_vpc" {
    cidr_block           = "10.0.0.0/16"
    enable_dns_support   = true
    enable_dns_hostnames = true
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-vpc"
    }
  }
  ```

Cria uma VPC com o CIDR `10.0.0.0/16`, permitindo suporte a DNS e nomes de host. A VPC é nomeada com base nas variáveis definidas.

6. **Sub-rede:**

  ```
  resource "aws_subnet" "main_subnet" {
    vpc_id            = aws_vpc.main_vpc.id
    cidr_block        = "10.0.1.0/24"
    availability_zone = "us-east-1a"
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-subnet"
    }
  }
  ```

Cria uma sub-rede dentro da VPC, com o CIDR `10.0.1.0/24` na zona de disponibilidade `us-east-1a`. Assim como a VPC, a sub-rede é nomeada usando as variáveis.

7. **Gateway de Internet:**

  ```
  resource "aws_internet_gateway" "main_igw" {
    vpc_id = aws_vpc.main_vpc.id
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-igw"
    }
  }
  ```
Cria um gateway de internet associado à VPC, permitindo que as instâncias na VPC acessem a internet.

8. **Tabela de Rotas:**

  ```
  resource "aws_route_table" "main_route_table" {
    vpc_id = aws_vpc.main_vpc.id
  
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.main_igw.id
    }
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-route_table"
    }
  }
  ```

Cria uma tabela de rotas que define uma rota padrão (0.0.0.0/0) para o gateway de internet, permitindo tráfego externo.

9. **Associação da Tabela de Rotas:**

  ```
  resource "aws_route_table_association" "main_association" {
    subnet_id      = aws_subnet.main_subnet.id
    route_table_id = aws_route_table.main_route_table.id
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-route_table_association"
    }
  }
  ```
Associa a tabela de rotas criada à sub-rede, garantindo que o tráfego da sub-rede utilize a tabela de rotas para direcionar pacotes para a internet.

10. **Grupo de Segurança:**

  ```
  resource "aws_security_group" "main_sg" {
    name        = "${var.projeto}-${var.candidato}-sg"
    description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
    vpc_id      = aws_vpc.main_vpc.id
  
    # Regras de entrada
    ingress {
      description      = "Allow SSH from anywhere"
      from_port        = 22
      to_port          = 22
      protocol         = "tcp"
      cidr_blocks      = ["0.0.0.0/0"]
      ipv6_cidr_blocks = ["::/0"]
    }
  
    # Regras de saída
    egress {
      description      = "Allow all outbound traffic"
      from_port        = 0
      to_port          = 0
      protocol         = "-1"
      cidr_blocks      = ["0.0.0.0/0"]
      ipv6_cidr_blocks = ["::/0"]
    }
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-sg"
    }
  }
  ```
Cria um grupo de segurança que permite acesso SSH (porta 22) de qualquer endereço IP (0.0.0.0/0) e permite todo o tráfego de saída.

11. **AMI para Instância:**

  ```
  data "aws_ami" "debian12" {
    most_recent = true
  
    filter {
      name   = "name"
      values = ["debian-12-amd64-*"]
    }
  
    filter {
      name   = "virtualization-type"
      values = ["hvm"]
    }
  
    owners = ["679593333241"]
  }
  ```
Obtém a AMI mais recente do Debian 12 com tipo de virtualização HVM, filtrando pelo nome e pelo proprietário.

12. **Instância EC2:**

  ```
  resource "aws_instance" "debian_ec2" {
    ami             = data.aws_ami.debian12.id
    instance_type   = "t2.micro"
    subnet_id       = aws_subnet.main_subnet.id
    key_name        = aws_key_pair.ec2_key_pair.key_name
    security_groups = [aws_security_group.main_sg.name]
  
    associate_public_ip_address = true
  
    root_block_device {
      volume_size           = 20
      volume_type           = "gp2"
      delete_on_termination = true
    }
  
    user_data = <<-EOF
                #!/bin/bash
                apt-get update -y
                apt-get upgrade -y
                EOF
  
    tags = {
      Name = "${var.projeto}-${var.candidato}-ec2"
    }
  }
  ```

Cria uma instância EC2 com a AMI do Debian 12,associando à sub-rede criada, usando o par de chaves gerado para SSH, um endereço IP público, um bloco de dispositivo de root de 20 GB e Um script de `user_data` que atualiza e faz upgrade do sistema ao ser iniciado.

13. **Saídas:**

  ```
  output "private_key" {
    description = "Chave privada para acessar a instância EC2"
    value       = tls_private_key.ec2_key.private_key_pem
    sensitive   = true
  }
  
  output "ec2_public_ip" {
    description = "Endereço IP público da instância EC2"
    value       = aws_instance.debian_ec2.public_ip
  }
  ```

Define duas saídas:
13.1 A chave privada gerada para acesso à instância;
13.2 O endereço IP público da instância EC2, permitindo que o usuário se conecte a ela.









