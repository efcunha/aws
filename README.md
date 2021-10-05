# Roteiro Ambiente AWS

O que iremos fazer?

![single-deployment](https://user-images.githubusercontent.com/52961166/135733356-af69d804-6d47-48f2-9154-be62f842fa78.png)

## Parte 1
1. Criação de usuário do IAM e permissões
2. Criação da instância do RancherServer pela aws-cli
3. Configuração do Rancher.
4. Configuração do Cluster Kubernetes.
5. Deployment do cluster pela aws-cli.

## Parte 2
6. Configuração do Traefik
7. Configuração do Longhorn
8. Criação do certificado não válido
9. Configuração do ELB
10. Configuração do Route 53

# Estamos prontos para rodar nossa aplicação.

# Parte 1

## 1 - Criação de usuário do IAM e permissões e configuração da AWS-CLI

https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

## 2 - Criação da instância do RancherServer pela aws-cli.

```sh 

# RANCHER SERVER

# --image-id              ami-01e7ca2ef94a0ae86
# --instance-type         t3.medium 
# --key-name              multicloud 
# --security-group-ids    sg-0b0e8363b215900f0 
# --subnet-id             subnet-4f5e7705

$ aws ec2 run-instances --image-id ami-01e7ca2ef94a0ae86 --count 1 --instance-type t3.medium --key-name multicloud --security-group-ids sg-0b0e8363b215900f0 --subnet-id subnet-67c83f0e --user-data file://rancher.sh --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=rancherserver}]' 'ResourceType=volume,Tags=[{Key=Name,Value=rancherserver}]' 
```

## 3 - Configuração do Rancher
Acessar o Rancher e configurar

https://rancher.dominio.com.br

## 4 - Configuração do Cluster Kubernetes.

Criar o cluster pelo Rancher e configurar.

## 5 - Deployment do cluster pela aws-cli

```sh
# --image-id ami-01e7ca2ef94a0ae86
# --count 3 
# --instance-type t3.large 
# --key-name multicloud 
# --security-group-ids sg-0b0e8363b215900f0 
# --subnet-id subnet-09c5a4961e6056757 
# --user-data file://k8s.sh

$ aws ec2 run-instances --image-id ami-01e7ca2ef94a0ae86 --count 3 --instance-type t3.large --key-name multicloud --security-group-ids sg-0b0e8363b215900f0 --subnet-id subnet-67c83f0e --user-data file://k8s.sh   --block-device-mapping "[ { \"DeviceName\": \"/dev/sda1\", \"Ebs\": { \"VolumeSize\": 70 } } ]" --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8s}]' 'ResourceType=volume,Tags=[{Key=Name,Value=k8s}]'     
```

Instalar o kubectl 

https://kubernetes.io/docs/tasks/tools/

# Parte 2

## 6 - Configuração do Traefik

Acesse o endereço abaixo e siga as instruções.

https://github.com/efcunha/Traefik-v2-TLS

## 7 - Criação do certificado

Criar certificado para nossos dominios:

 *.dominio.com.br

```sh
> openssl req -new -x509 -keyout cert.pem -out cert.pem -days 365 -nodes
Country Name (2 letter code) [AU]:DE
State or Province Name (full name) [Some-State]:Germany
Locality Name (eg, city) []:nameOfYourCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:nameOfYourCompany
Organizational Unit Name (eg, section) []:nameOfYourDivision
Common Name (eg, YOUR name) []:*.example.com
Email Address []:webmaster@example.com
```

arn:aws:acm:us-east-2:984102645395:certificate/ffdf5439-9d21-421e-b730-0dadb52bbd01

## 8 - Configuração do ELB

```sh
# LOAD BALANCER

# !! ESPECIFICAR O SECURITY GROUPS DO LOAD BALANCER

# --subnets subnet-4f5e7705 subnet-67c83f0e

$ aws elbv2 create-load-balancer --name multicloud --type application --subnets subnet-4f5e7705 subnet-67c83f0e
#	 "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-east-2:984102645395:loadbalancer/app/multicloud/1a4af5c3698503fb"

# --vpc-id vpc-238e664a

$ aws elbv2 create-target-group --name multicloud --protocol HTTP --port 80 --vpc-id vpc-238e664a --health-check-port 8080 --health-check-path /api/providers
#	 "TargetGroupArn": "arn:aws:elasticloadbalancing:us-east-2:984102645395:targetgroup/multicloud/0e70910ded08498f"
	
# REGISTRAR OS TARGETS  
$ aws elbv2 register-targets --target-group-arn arn:aws:elasticloadbalancing:us-east-2:984102645395:targetgroup/multicloud/0e70910ded08498f --targets Id=i-04c0b078f1ef0968c Id=i-014c4de5f78e1d911 Id=i-0aea6b0657ad26b34

i-04c0b078f1ef0968c
i-014c4de5f78e1d911
i-0aea6b0657ad26b34

# ARN DO Certificado - arn:aws:acm:us-east-1:984102645395:certificate/fa016001-254f-4127-b51a-61588b15c555
# HTTPS - CRIADO PRIMEIRO
$ aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-2:984102645395:loadbalancer/app/multicloud/1a4af5c3698503fb \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:us-east-2:984102645395:certificate/ffdf5439-9d21-421e-b730-0dadb52bbd01   \
    --ssl-policy ELBSecurityPolicy-2016-08 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-2:984102645395:targetgroup/multicloud/0e70910ded08498f
#  "ListenerArn": "arn:aws:elasticloadbalancing:us-east-2:984102645395:listener/app/multicloud/1a4af5c3698503fb/0ba2e3ab81d739b7"

$ aws elbv2 describe-target-health --target-group-arn targetgroup-arn

# DESCRIBE NO LISTENER
$ aws elbv2 describe-listeners --listener-arns arn:aws:elasticloadbalancing:us-east-1:984102645395:listener/app/multicloud/0c7e036793bff35e/a7386cf3e0dc3c0e
```
