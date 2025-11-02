# 🤖 Guia Completo de Automação, Terraform e DevOps na AWS

> Documentação detalhada sobre automação de tarefas, Lambda + S3, Terraform, conceitos e ferramentas DevOps na AWS

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=for-the-badge&logo=amazon-aws)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?style=for-the-badge&logo=terraform)](https://www.terraform.io/)
[![DevOps](https://img.shields.io/badge/DevOps-Culture-0DB7ED?style=for-the-badge&logo=azure-devops)](https://aws.amazon.com/devops/)

## 📑 Índice

- [Automação de Tarefas na AWS](#-automação-de-tarefas-na-aws)
- [Lambda + S3 - Tarefas Automatizadas](#-lambda--s3---tarefas-automatizadas)
- [Terraform - Infrastructure as Code](#-terraform---infrastructure-as-code)
- [Introdução ao DevOps](#-introdução-ao-devops)
- [Conceitos de DevOps na AWS](#-conceitos-de-devops-na-aws)
- [Ferramentas DevOps AWS](#-ferramentas-devops-aws)

---

## 🤖 Automação de Tarefas na AWS

### Por que Automatizar?

```
✅ Reduz erros humanos
✅ Aumenta velocidade de execução
✅ Melhora consistência
✅ Economiza tempo e dinheiro
✅ Facilita escalabilidade
✅ Permite repetibilidade
```

### Métodos de Automação AWS

#### 1. AWS Lambda (Event-Driven)

**Execução serverless** baseada em eventos.

**Casos de uso:**
- 📸 Processar imagens quando uploadadas no S3
- 📊 Gerar relatórios periodicamente
- 🔔 Enviar notificações customizadas
- 🧹 Limpeza automática de recursos
- 🔐 Resposta automática a eventos de segurança

#### 2. EventBridge (CloudWatch Events)

**Barramento de eventos** para automação baseada em schedule ou eventos.

**Exemplos:**
```
✅ Executar Lambda toda manhã às 6h
✅ Reagir a mudanças de estado EC2
✅ Integrar eventos de SaaS (Datadog, Zendesk)
✅ Criar workflows complexos
```

#### 3. Systems Manager (SSM)

**Gerenciamento e automação** de instâncias EC2 e on-premises.

**Recursos:**
- 🚀 Run Command: Executar comandos remotamente
- 📋 State Manager: Manter configuração desejada
- 📦 Patch Manager: Gerenciar patches de SO
- 🔧 Automation: Workflows de automação
- 📝 Parameter Store: Armazenar configurações

#### 4. AWS Config

**Avaliação e auditoria** contínua de configurações.

**Funcionalidades:**
- ✅ Compliance automático com rules
- 📊 Histórico de mudanças
- 🔧 Auto-remediation de não conformidades

#### 5. AWS CLI + Scripts

**Automação via linha de comando** com bash, python, etc.

### Exemplo: Automatizar Backup de EC2

#### Usando Lambda + EventBridge

**Lambda Function (Python):**
```python
import boto3
from datetime import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Encontrar instâncias com tag Backup=true
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Backup', 'Values': ['true']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    timestamp = datetime.now().strftime('%Y-%m-%d-%H-%M')
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            instance_name = next((tag['Value'] for tag in instance.get('Tags', []) if tag['Key'] == 'Name'), 'Unknown')
            
            # Criar AMI
            ami_response = ec2.create_image(
                InstanceId=instance_id,
                Name=f"Backup-{instance_name}-{timestamp}",
                Description=f"Automated backup of {instance_name}",
                NoReboot=True
            )
            
            ami_id = ami_response['ImageId']
            
            # Taguear AMI
            ec2.create_tags(
                Resources=[ami_id],
                Tags=[
                    {'Key': 'Name', 'Value': f'Backup-{instance_name}'},
                    {'Key': 'CreatedBy', 'Value': 'AutoBackup'},
                    {'Key': 'Timestamp', 'Value': timestamp}
                ]
            )
            
            print(f"Created AMI {ami_id} for instance {instance_id}")
    
    return {
        'statusCode': 200,
        'body': 'Backup completed successfully'
    }
```

**EventBridge Rule (Cron):**
```json
{
  "ScheduleExpression": "cron(0 2 * * ? *)",
  "State": "ENABLED",
  "Targets": [
    {
      "Arn": "arn:aws:lambda:us-east-1:123456789012:function:AutoBackup",
      "Id": "1"
    }
  ]
}
```

### Exemplo: Auto-Scaling baseado em Métricas Customizadas

**Lambda que publica métricas customizadas:**
```python
import boto3

cloudwatch = boto3.client('cloudwatch')

def lambda_handler(event, context):
    # Simular coleta de métrica de aplicação
    queue_length = get_queue_length()  # Sua lógica aqui
    
    # Publicar no CloudWatch
    cloudwatch.put_metric_data(
        Namespace='CustomApp',
        MetricData=[
            {
                'MetricName': 'QueueLength',
                'Value': queue_length,
                'Unit': 'Count'
            }
        ]
    )
    
    return {'statusCode': 200}
```

**Auto Scaling Policy baseada na métrica:**
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name scale-on-queue-length \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "CustomizedMetricSpecification": {
      "MetricName": "QueueLength",
      "Namespace": "CustomApp",
      "Statistic": "Average"
    },
    "TargetValue": 100
  }'
```

### Exemplo: Limpeza Automática de Snapshots Antigos

**Lambda Function:**
```python
import boto3
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    # Definir idade máxima (30 dias)
    retention_days = 30
    cutoff_date = datetime.now() - timedelta(days=retention_days)
    
    # Listar snapshots
    snapshots = ec2.describe_snapshots(OwnerIds=['self'])['Snapshots']
    
    deleted_count = 0
    
    for snapshot in snapshots:
        snapshot_id = snapshot['SnapshotId']
        start_time = snapshot['StartTime'].replace(tzinfo=None)
        
        # Verificar se snapshot tem tag DeleteProtection
        tags = {tag['Key']: tag['Value'] for tag in snapshot.get('Tags', [])}
        
        if tags.get('DeleteProtection') == 'true':
            print(f"Skipping protected snapshot: {snapshot_id}")
            continue
        
        # Deletar se mais antigo que cutoff
        if start_time < cutoff_date:
            try:
                ec2.delete_snapshot(SnapshotId=snapshot_id)
                print(f"Deleted snapshot: {snapshot_id} (Age: {(datetime.now() - start_time).days} days)")
                deleted_count += 1
            except Exception as e:
                print(f"Failed to delete {snapshot_id}: {str(e)}")
    
    return {
        'statusCode': 200,
        'body': f'Deleted {deleted_count} snapshots'
    }
```

### Exemplo: Notificação de Custos Elevados

**Lambda que verifica custos diários:**
```python
import boto3
import json
from datetime import datetime, timedelta

ce = boto3.client('ce')
sns = boto3.client('sns')

def lambda_handler(event, context):
    # Obter custos dos últimos 7 dias
    end_date = datetime.now().date()
    start_date = end_date - timedelta(days=7)
    
    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': str(start_date),
            'End': str(end_date)
        },
        Granularity='DAILY',
        Metrics=['UnblendedCost']
    )
    
    # Calcular média e custo de ontem
    daily_costs = [float(day['Total']['UnblendedCost']['Amount']) 
                   for day in response['ResultsByTime']]
    
    avg_cost = sum(daily_costs) / len(daily_costs)
    yesterday_cost = daily_costs[-1]
    
    # Alerta se custo de ontem for 50% maior que média
    threshold = avg_cost * 1.5
    
    if yesterday_cost > threshold:
        message = f"""
        ⚠️ ALERTA DE CUSTOS ELEVADOS!
        
        Custo de ontem: ${yesterday_cost:.2f}
        Média dos últimos 7 dias: ${avg_cost:.2f}
        Threshold: ${threshold:.2f}
        
        Variação: {((yesterday_cost/avg_cost - 1) * 100):.1f}%
        """
        
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:billing-alerts',
            Subject='⚠️ Alerta de Custos AWS',
            Message=message
        )
        
        print("Alert sent!")
    else:
        print("Costs within normal range")
    
    return {'statusCode': 200}
```

---

## 📦 Lambda + S3 - Tarefas Automatizadas

### Arquitetura Event-Driven com S3

```
S3 Bucket
    │
    ├─ Upload File → Event → Lambda
    ├─ Delete File → Event → Lambda
    └─ Object Created → Event → Lambda
```

### Casos de Uso Comuns

#### 1. Processamento de Imagens

**Cenário:** Usuário faz upload de imagem → Lambda cria thumbnails

**Lambda Function (Python):**
```python
import boto3
from PIL import Image
import io

s3 = boto3.client('s3')

def lambda_handler(event, context):
    # Extrair informações do evento S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download da imagem original
    response = s3.get_object(Bucket=bucket, Key=key)
    image_data = response['Body'].read()
    
    # Abrir imagem
    image = Image.open(io.BytesIO(image_data))
    
    # Criar thumbnail (150x150)
    image.thumbnail((150, 150))
    
    # Salvar em buffer
    buffer = io.BytesIO()
    image.save(buffer, format=image.format)
    buffer.seek(0)
    
    # Upload do thumbnail
    thumbnail_key = f"thumbnails/{key}"
    s3.put_object(
        Bucket=bucket,
        Key=thumbnail_key,
        Body=buffer,
        ContentType=f'image/{image.format.lower()}'
    )
    
    print(f"Thumbnail created: {thumbnail_key}")
    
    return {
        'statusCode': 200,
        'body': f'Thumbnail created for {key}'
    }
```

**S3 Event Configuration:**
```json
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:CreateThumbnail",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "uploads/"
            },
            {
              "Name": "suffix",
              "Value": ".jpg"
            }
          ]
        }
      }
    }
  ]
}
```

#### 2. Conversão de Arquivo (CSV → JSON)

**Lambda Function:**
```python
import boto3
import csv
import json
from io import StringIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download CSV
    response = s3.get_object(Bucket=bucket, Key=key)
    csv_content = response['Body'].read().decode('utf-8')
    
    # Parse CSV
    csv_reader = csv.DictReader(StringIO(csv_content))
    data = list(csv_reader)
    
    # Converter para JSON
    json_data = json.dumps(data, indent=2)
    
    # Upload JSON
    json_key = key.replace('.csv', '.json')
    s3.put_object(
        Bucket=bucket,
        Key=json_key,
        Body=json_data,
        ContentType='application/json'
    )
    
    print(f"Converted {key} to {json_key}")
    
    return {'statusCode': 200}
```

#### 3. Análise de Log e Alertas

**Lambda que analisa logs e alerta sobre erros:**
```python
import boto3
import json
import re

s3 = boto3.client('s3')
sns = boto3.client('sns')

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download log file
    response = s3.get_object(Bucket=bucket, Key=key)
    log_content = response['Body'].read().decode('utf-8')
    
    # Analisar erros
    error_pattern = r'ERROR|CRITICAL|FATAL'
    errors = re.findall(f'.*({error_pattern}).*', log_content, re.IGNORECASE)
    
    if errors:
        error_count = len(errors)
        sample_errors = '\n'.join(errors[:5])  # Primeiros 5 erros
        
        message = f"""
        🚨 ERROS DETECTADOS EM LOGS
        
        Arquivo: {key}
        Total de erros: {error_count}
        
        Exemplos:
        {sample_errors}
        """
        
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:log-alerts',
            Subject=f'🚨 {error_count} erros detectados',
            Message=message
        )
        
        print(f"Alert sent for {error_count} errors")
    
    return {'statusCode': 200}
```

#### 4. Backup Automático para Glacier

**Lambda que move objetos antigos para Glacier:**
```python
import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')

def lambda_handler(event, context):
    bucket = 'my-bucket'
    days_threshold = 90
    cutoff_date = datetime.now() - timedelta(days=days_threshold)
    
    # Listar objetos
    paginator = s3.get_paginator('list_objects_v2')
    pages = paginator.paginate(Bucket=bucket)
    
    moved_count = 0
    
    for page in pages:
        for obj in page.get('Contents', []):
            key = obj['Key']
            last_modified = obj['LastModified'].replace(tzinfo=None)
            
            # Se mais antigo que threshold, mover para Glacier
            if last_modified < cutoff_date:
                s3.copy_object(
                    Bucket=bucket,
                    CopySource={'Bucket': bucket, 'Key': key},
                    Key=key,
                    StorageClass='GLACIER'
                )
                print(f"Moved to Glacier: {key}")
                moved_count += 1
    
    return {
        'statusCode': 200,
        'body': f'Moved {moved_count} objects to Glacier'
    }
```

#### 5. Validação e Quarentena de Arquivos

**Lambda que valida arquivos e move para quarentena se inválidos:**
```python
import boto3
import magic  # python-magic library

s3 = boto3.client('s3')

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download arquivo
    response = s3.get_object(Bucket=bucket, Key=key)
    file_content = response['Body'].read()
    
    # Detectar tipo MIME real
    actual_mime = magic.from_buffer(file_content, mime=True)
    
    # Expected MIME (baseado na extensão)
    if key.endswith('.pdf'):
        expected_mime = 'application/pdf'
    elif key.endswith('.jpg') or key.endswith('.jpeg'):
        expected_mime = 'image/jpeg'
    else:
        expected_mime = None
    
    # Validar
    if expected_mime and actual_mime != expected_mime:
        # Mover para quarentena
        quarantine_key = f"quarantine/{key}"
        s3.copy_object(
            Bucket=bucket,
            CopySource={'Bucket': bucket, 'Key': key},
            Key=quarantine_key
        )
        s3.delete_object(Bucket=bucket, Key=key)
        
        print(f"⚠️ File moved to quarantine: {key} (Expected: {expected_mime}, Got: {actual_mime})")
        
        return {
            'statusCode': 400,
            'body': 'File validation failed - moved to quarantine'
        }
    
    print(f"✅ File validated: {key}")
    return {'statusCode': 200, 'body': 'File validated successfully'}
```

### Boas Práticas Lambda + S3

- 🎯 **Filtre eventos** com prefix/suffix para evitar invocações desnecessárias
- ⏱️ **Configure timeout** adequado (imagens grandes precisam mais tempo)
- 💾 **Use /tmp** para arquivos temporários (512 MB disponíveis)
- 🔄 **Implemente retry logic** para falhas transitórias
- 📊 **Log tudo** para debugging
- 🔐 **Valide inputs** antes de processar
- ⚡ **Otimize cold starts** minimizando dependências
- 🏷️ **Tag objetos processados** para rastreabilidade

---

## 🏗️ Terraform - Infrastructure as Code

### O que é Terraform?

Terraform é uma ferramenta de **Infrastructure as Code (IaC)** open-source da HashiCorp que permite definir recursos de infraestrutura em arquivos de configuração declarativos.

### Por que Terraform?

```
✅ Multi-cloud (AWS, Azure, GCP, etc)
✅ Sintaxe declarativa e legível (HCL)
✅ Plano de execução (preview de mudanças)
✅ State management
✅ Modularização e reutilização
✅ Versionamento de infraestrutura
✅ Community modules extensos
```

### Terraform vs CloudFormation

| Aspecto | Terraform | CloudFormation |
|---------|-----------|----------------|
| **Provider** | Multi-cloud | AWS apenas |
| **Linguagem** | HCL (HashiCorp Configuration Language) | JSON/YAML |
| **State** | Local ou remoto (S3, Terraform Cloud) | Gerenciado pela AWS |
| **Preview** | `terraform plan` | Change Sets |
| **Comunidade** | Enorme (multi-cloud) | AWS específico |
| **Custo** | Gratuito (open-source) | Gratuito |

### Instalação

**Linux/macOS:**
```bash
# Download
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip

# Extrair
unzip terraform_1.6.0_linux_amd64.zip

# Mover para PATH
sudo mv terraform /usr/local/bin/

# Verificar
terraform version
```

**Windows (Chocolatey):**
```powershell
choco install terraform
```

**macOS (Homebrew):**
```bash
brew install terraform
```

### Estrutura Básica

```hcl
# Provider configuration
provider "aws" {
  region = "us-east-1"
}

# Resource definition
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  
  tags = {
    Name = "MyWebServer"
  }
}
```

### Workflow Terraform

```
1. terraform init    → Inicializa e baixa providers
2. terraform plan    → Preview das mudanças
3. terraform apply   → Aplica mudanças
4. terraform destroy → Destroi recursos
```

### Exemplo Completo: Web Server com ALB

**Directory Structure:**
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**main.tf:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Security Group para ALB
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

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

  tags = {
    Name = "${var.project_name}-alb-sg"
  }
}

# Security Group para EC2
resource "aws_security_group" "ec2" {
  name        = "${var.project_name}-ec2-sg"
  description = "Security group for EC2 instances"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-ec2-sg"
  }
}

# Launch Template
resource "aws_launch_template" "web" {
  name_prefix   = "${var.project_name}-lt-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.ec2.id]

  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from $(hostname -f)</h1>" > /var/www/html/index.html
              EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.project_name}-web-server"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web" {
  name                = "${var.project_name}-asg"
  vpc_zone_identifier = aws_subnet.public[*].id
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  desired_capacity    = var.asg_desired_capacity

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-asg-instance"
    propagate_at_launch = true
  }
}

# Application Load Balancer
resource "aws_lb" "web" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  tags = {
    Name = "${var.project_name}-alb"
  }
}

# Target Group
resource "aws_lb_target_group" "web" {
  name     = "${var.project_name}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }

  tags = {
    Name = "${var.project_name}-tg"
  }
}

# Listener
resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# Data source para AZs
data "aws_availability_zones" "available" {
  state = "available"
}
```

**variables.tf:**
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "myapp"
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "asg_min_size" {
  description = "Minimum size of ASG"
  type        = number
  default     = 2
}

variable "asg_max_size" {
  description = "Maximum size of ASG"
  type        = number
  default     = 4
}

variable "asg_desired_capacity" {
  description = "Desired capacity of ASG"
  type        = number
  default     = 2
}
```

**outputs.tf:**
```hcl
output "alb_dns_name" {
  description = "DNS name of the load balancer"
  value       = aws_lb.web.dns_name
}

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "asg_name" {
  description = "Name of the Auto Scaling Group"
  value       = aws_autoscaling_group.web.name
}
```

**terraform.tfvars:**
```hcl
aws_region           = "us-east-1"
project_name         = "mywebapp"
ami_id               = "ami-0c55b159cbfafe1f0"
instance_type        = "t3.micro"
asg_min_size         = 2
asg_max_size         = 4
asg_desired_capacity = 2
```

### Terraform Commands

```bash
# Inicializar
terraform init

# Formatar código
terraform fmt

# Validar configuração
terraform validate

# Plan (preview)
terraform plan

# Apply
terraform apply

# Apply com auto-approve
terraform apply -auto-approve

# Destruir recursos
terraform destroy

# Mostrar state
terraform show

# Listar recursos no state
terraform state list

# Ver recurso específico
terraform state show aws_instance.web_server

# Importar recurso existente
terraform import aws_instance.web_server i-1234567890abcdef0

# Output values
terraform output

# Refresh state
terraform refresh
```

### Terraform State

**State** é o mapeamento entre configuração Terraform e recursos reais.

**State Local:**
```
terraform.tfstate      → State atual
terraform.tfstate.backup → Backup do state anterior
```

**Remote State (S3):**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

**Benefícios do Remote State:**
- 👥 Colaboração em equipe
- 🔐 Criptografia
- 🔒 Locking (evita conflitos)
- 💾 Backup automático

### Terraform Modules

**Modules** permitem reutilização e organização de código.

**Structure:**
```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── ec2/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── rds/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

**Using Module:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
  project_name = "myapp"
}

module "ec2" {
  source = "./modules/ec2"
  
  vpc_id = module.vpc.vpc_id
  subnet_ids = module.vpc.public_subnet_ids
}
```

### Workspaces

**Workspaces** permitem gerenciar múltiplos ambientes.

```bash
# Listar workspaces
terraform workspace list

# Criar workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Selecionar workspace
terraform workspace select prod

# Workspace atual
terraform workspace show
```

**Usar workspace no código:**
```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  
  tags = {
    Name = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

### Data Sources

**Data sources** permitem ler informações de recursos existentes.

```hcl
# Buscar AMI mais recente
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Usar no resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}

# Buscar VPC padrão
data "aws_vpc" "default" {
  default = true
}

# Buscar zonas de disponibilidade
data "aws_availability_zones" "available" {
  state = "available"
}
```

### Provisioners

**Provisioners** executam ações após criação do recurso.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  key_name      = var.key_name

  # Provisioner local-exec
  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }

  # Provisioner remote-exec
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y httpd",
      "sudo systemctl start httpd",
      "sudo systemctl enable httpd"
    ]

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/mykey.pem")
      host        = self.public_ip
    }
  }

  # Provisioner file
  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/mykey.pem")
      host        = self.public_ip
    }
  }
}
```

⚠️ **Nota:** Provisioners são "último recurso". Prefira user_data ou AMIs configuradas.

### Terraform Best Practices

- 📁 **Use módulos** para código reutilizável
- 🔐 **Remote state** em S3 com locking
- 🏷️ **Tag tudo** com Owner, Environment, Project
- 📝 **Documente** variáveis e outputs
- 🎯 **Use data sources** para recursos existentes
- 🔄 **Versionamento** de módulos e providers
- 🧪 **Teste** com terraform plan sempre
- 🚫 **Não commite** .tfstate ou credenciais
- 📊 **Use workspaces** para ambientes
- ⚡ **Módulos pequenos** e focados

---

## 🚀 Introdução ao DevOps

### O que é DevOps?

**DevOps** é uma cultura, movimento e prática que **unifica desenvolvimento (Dev) e operações (Ops)** para entregar software de forma mais rápida, confiável e frequente.

### Pilares do DevOps

#### 1. Cultura
```
👥 Colaboração entre equipes
🤝 Quebrar silos organizacionais
📢 Comunicação aberta
🎯 Responsabilidade compartilhada
💡 Aprendizado contínuo
```

#### 2. Automação
```
🤖 CI/CD pipelines
🏗️ Infrastructure as Code
🧪 Testes automatizados
📦 Deployment automatizado
📊 Monitoramento automatizado
```

#### 3. Medição
```
📈 Métricas de performance
📊 KPIs de negócio
🔍 Observabilidade
⏱️ Tempo de deploy
🐛 Taxa de bugs
```

#### 4. Compartilhamento
```
📚 Documentação viva
🎓 Knowledge sharing
🔄 Feedback loops
📖 Post-mortems blameless
💬 Retrospectivas
```

### Princípios DevOps

#### 1. Continuous Integration (CI)

**Integrar código frequentemente** (múltiplas vezes ao dia).

```
Developer → Commit → Automated Build → Automated Tests → Feedback
```

**Benefícios:**
- ✅ Detectar bugs cedo
- ✅ Reduzir conflitos de merge
- ✅ Melhorar qualidade do código
- ✅ Acelerar desenvolvimento

#### 2. Continuous Delivery (CD)

**Código sempre em estado deployável**, deploy manual.

```
CI → Staging Deploy → Manual Approval → Production Deploy
```

#### 3. Continuous Deployment

**Deploy automático** para produção após passar nos testes.

```
CI → Automated Tests → Automatic Production Deploy → Monitor
```

### Ciclo DevOps

```
        Plan
         ↓
       Code
         ↓
      Build
         ↓
       Test
         ↓
      Release
         ↓
      Deploy
         ↓
     Operate
         ↓
     Monitor
         ↓
    (Feedback to Plan)
```

### DevOps vs Traditional

| Aspecto | Traditional | DevOps |
|---------|------------|--------|
| **Deploys** | Mensal/Trimestral | Diário/Semanal |
| **Equipes** | Silos separados | Colaborativas |
| **Processos** | Manuais | Automatizados |
| **Mudanças** | Grandes releases | Pequenas iterações |
| **Feedback** | Lento | Rápido |
| **Responsabilidade** | "Jogar por cima do muro" | Compartilhada |

### Benefícios do DevOps

```
⚡ Deploy mais rápido
🎯 Maior qualidade
💰 Redução de custos
😊 Melhor colaboração
📈 Maior inovação
🛡️ Mais segurança
🔄 Feedback rápido
📊 Melhor visibilidade
```

---

## 🎯 Conceitos de DevOps na AWS

### AWS DevOps Services

#### 1. CodeCommit - Source Control

**Git-based** repository privado na AWS.

**Características:**
- 🔐 Integração com IAM
- 🔒 Criptografia em repouso e trânsito
- 👥 Pull requests e code reviews
- 🔔 Triggers e notificações

```bash
# Clonar repositório
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MyRepo

# Push
git push origin main

# Configurar credential helper
git config --global credential.helper '!aws codecommit credential-helper $@'
```

#### 2. CodeBuild - Build Service

**Serviço gerenciado** de build e testes.

**buildspec.yml:**
```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG

artifacts:
  files:
    - '**/*'
```

#### 3. CodeDeploy - Deployment

Já coberto na seção anterior.

#### 4. CodePipeline - CI/CD Orchestration

**Orquestrador** de pipelines CI/CD.

**Exemplo de Pipeline:**
```
Source (CodeCommit)
    ↓
Build (CodeBuild)
    ↓
Test (CodeBuild)
    ↓
Manual Approval
    ↓
Deploy to Staging (CodeDeploy)
    ↓
Integration Tests
    ↓
Manual Approval
    ↓
Deploy to Production (CodeDeploy)
```

**Criar via CLI:**
```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

**pipeline.json:**
```json
{
  "pipeline": {
    "name": "MyPipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "SourceAction",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "configuration": {
              "RepositoryName": "MyRepo",
              "BranchName": "main"
            },
            "outputArtifacts": [{"name": "SourceOutput"}]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildAction",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "MyBuildProject"
            },
            "inputArtifacts": [{"name": "SourceOutput"}],
            "outputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "DeployAction",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeploy",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "MyApp",
              "DeploymentGroupName": "Production"
            },
            "inputArtifacts": [{"name": "BuildOutput"}]
          }
        ]
      }
    ]
  }
}
```

### Infrastructure as Code na AWS

#### CloudFormation vs Terraform

Já comparado anteriormente. Ambos são IaC válidos.

#### AWS CDK (Cloud Development Kit)

**Define infraestrutura** usando linguagens de programação.

**Exemplo Python:**
```python
from aws_cdk import (
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns,
    Stack
)
from constructs import Construct

class MyStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # VPC
        vpc = ec2.Vpc(self, "MyVpc", max_azs=2)

        # ECS Cluster
        cluster = ecs.Cluster(self, "MyCluster", vpc=vpc)

        # Fargate Service com ALB
        ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "MyFargateService",
            cluster=cluster,
            cpu=256,
            memory_limit_mib=512,
            desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_registry("amazon/amazon-ecs-sample")
            )
        )
```

### Containerization

#### Docker na AWS

**ECR (Elastic Container Registry)** - Registry Docker privado.

```bash
# Login no ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build imagem
docker build -t myapp .

# Tag
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

#### ECS vs EKS

Já comparado anteriormente.

### Monitoring e Logging

#### CloudWatch

- 📊 **Metrics**: Performance dos recursos
- 📝 **Logs**: Logs centralizados
- 🔔 **Alarms**: Alertas automáticos
- 📈 **Dashboards**: Visualização

#### X-Ray

**Distributed tracing** para debugging de aplicações.

```python
from aws_xray_sdk.core import xray_recorder

@xray_recorder.capture('my_function')
def my_function():
    # Seu código
    pass
```

#### CloudTrail

**Auditoria** de API calls (já coberto anteriormente).

---

## 🛠️ Ferramentas DevOps AWS

### 1. AWS Systems Manager (SSM)

#### Run Command

**Executar comandos** remotamente em instâncias.

```bash
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=production" \
  --parameters commands=["sudo yum update -y"]
```

#### Session Manager

**SSH sem SSH** - acesso via console/CLI sem keys.

```bash
# Iniciar sessão
aws ssm start-session --target i-1234567890abcdef0
```

#### Parameter Store

**Armazenar configurações** e secrets.

```bash
# Criar parâmetro
aws ssm put-parameter \
  --name "/myapp/database/password" \
  --value "MySecurePassword123" \
  --type "SecureString"

# Recuperar parâmetro
aws ssm get-parameter \
  --name "/myapp/database/password" \
  --with-decryption
```

**Usar no código:**
```python
import boto3

ssm = boto3.client('ssm')

response = ssm.get_parameter(
    Name='/myapp/database/password',
    WithDecryption=True
)

password = response['Parameter']['Value']
```

#### Patch Manager

**Gerenciar patches** de SO automaticamente.

#### State Manager

**Manter configuração** desejada.

### 2. AWS Secrets Manager

**Gerenciar secrets** com rotação automática.

```bash
# Criar secret
aws secretsmanager create-secret \
  --name prod/myapp/database \
  --secret-string '{"username":"admin","password":"MyPass123"}'

# Recuperar secret
aws secretsmanager get-secret-value \
  --secret-id prod/myapp/database
```

**Rotação automática:**
```bash
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/database \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:RotateSecret \
  --rotation-rules AutomaticallyAfterDays=30
```

### 3. AWS Config

**Auditoria e compliance** de recursos.

**Config Rules:**
```bash
# Rule: S3 buckets devem ter encryption
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
    }
  }'
```

### 4. AWS CloudFormation

Já coberto anteriormente.

### 5. AWS OpsWorks

**Configuration management** com Chef/Puppet.

### 6. AWS Service Catalog

**Catálogo de produtos** aprovados para self-service.

### 7. AWS Control Tower

**Landing zone** e governança multi-conta.

### 8. AWS Organizations

**Gerenciamento centralizado** de múltiplas contas.

### 9. AWS Cost Explorer

**Análise de custos** e otimização.

### 10. AWS Trusted Advisor

**Recomendações** de best practices.

**5 Pilares:**
- 💰 Cost Optimization
- 🚀 Performance
- 🔐 Security
- 🛡️ Fault Tolerance
- 📊 Service Limits

---

## 🎓 Exemplo Completo: Pipeline DevOps

### Cenário: Aplicação Node.js com CI/CD Completo

**Architecture:**
```
GitHub → CodePipeline → CodeBuild → ECR → ECS (Fargate) → ALB
                ↓
            CodeDeploy (Blue/Green)
                ↓
            CloudWatch (Monitoring)
```

### 1. Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "app.js"]
```

### 2. buildspec.yml

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
  
  build:
    commands:
      - echo Build started on `date`
      - echo Running tests...
      - npm install
      - npm test
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG
  
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"myapp","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
```

### 3. appspec.yaml (ECS)

```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "myapp"
          ContainerPort: 3000
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "LambdaFunctionToValidateBeforeInstall"
  - AfterInstall: "LambdaFunctionToValidateAfterInstall"
  - AfterAllowTestTraffic: "LambdaFunctionToValidateAfterTestTraffic"
  - BeforeAllowTraffic: "LambdaFunctionToValidateBeforeAllowingProductionTraffic"
  - AfterAllowTraffic: "LambdaFunctionToValidateAfterAllowingProductionTraffic"
```

### 4. Terraform para Infraestrutura

```hcl
# ECR Repository
resource "aws_ecr_repository" "app" {
  name                 = "myapp"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "myapp-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "myapp"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name      = "myapp"
      image     = "${aws_ecr_repository.app.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/myapp"
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "ecs"
        }
      }
      environment = [
        {
          name  = "NODE_ENV"
          value = "production"
        }
      ]
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "myapp-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "myapp"
    container_port   = 3000
  }

  deployment_controller {
    type = "CODE_DEPLOY"
  }

  depends_on = [aws_lb_listener.app]
}

# CodeBuild Project
resource "aws_codebuild_project" "app" {
  name          = "myapp-build"
  service_role  = aws_iam_role.codebuild.arn

  artifacts {
    type = "CODEPIPELINE"
  }

  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "aws/codebuild/standard:5.0"
    type                        = "LINUX_CONTAINER"
    privileged_mode             = true
    image_pull_credentials_type = "CODEBUILD"

    environment_variable {
      name  = "AWS_ACCOUNT_ID"
      value = data.aws_caller_identity.current.account_id
    }

    environment_variable {
      name  = "IMAGE_REPO_NAME"
      value = aws_ecr_repository.app.name
    }
  }

  source {
    type = "CODEPIPELINE"
  }
}

# CodePipeline
resource "aws_codepipeline" "app" {
  name     = "myapp-pipeline"
  role_arn = aws_iam_role.codepipeline.arn

  artifact_store {
    location = aws_s3_bucket.artifacts.bucket
    type     = "S3"
  }

  stage {
    name = "Source"

    action {
      name             = "Source"
      category         = "Source"
      owner            = "ThirdParty"
      provider         = "GitHub"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        Owner      = "myusername"
        Repo       = "myapp"
        Branch     = "main"
        OAuthToken = var.github_token
      }
    }
  }

  stage {
    name = "Build"

    action {
      name             = "Build"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]
      version          = "1"

      configuration = {
        ProjectName = aws_codebuild_project.app.name
      }
    }
  }

  stage {
    name = "Deploy"

    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CodeDeployToECS"
      input_artifacts = ["build_output"]
      version         = "1"

      configuration = {
        ApplicationName                = aws_codedeploy_app.app.name
        DeploymentGroupName            = aws_codedeploy_deployment_group.app.deployment_group_name
        TaskDefinitionTemplateArtifact = "build_output"
        AppSpecTemplateArtifact        = "build_output"
      }
    }
  }
}
```

### 5. Monitoramento

**CloudWatch Dashboard:**
```hcl
resource "aws_cloudwatch_dashboard" "app" {
  dashboard_name = "myapp-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ECS", "CPUUtilization", {stat: "Average"}],
            [".", "MemoryUtilization", {stat: "Average"}]
          ]
          period = 300
          stat   = "Average"
          region = "us-east-1"
          title  = "ECS Metrics"
        }
      },
      {
        type = "metric"
        properties = {
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", {stat: "Average"}],
            [".", "RequestCount", {stat: "Sum"}]
          ]
          period = 300
          region = "us-east-1"
          title  = "ALB Metrics"
        }
      }
    ]
  })
}
```

**CloudWatch Alarms:**
```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "myapp-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS CPU utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.app.name
  }
}
```

---

## 📚 Recursos Adicionais

### Documentação Oficial

- [AWS DevOps](https://aws.amazon.com/devops/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/)
- [AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

### Livros Recomendados

- 📖 "The DevOps Handbook" - Gene Kim
- 📖 "Accelerate" - Nicole Forsgren
- 📖 "The Phoenix Project" - Gene Kim
- 📖 "Terraform: Up & Running" - Yevgeniy Brikman

### Cursos e Certificações

- 🎓 **AWS Certified DevOps Engineer** - Professional
- 🎓 **HashiCorp Certified: Terraform Associate**
- 🎓 **AWS Certified Solutions Architect**
- 🎓 **AWS Certified Developer**
