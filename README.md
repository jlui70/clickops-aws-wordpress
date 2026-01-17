# WordPress na AWS: Implementação Manual (ClickOps)

> **Guia passo a passo para implementação manual** de arquitetura WordPress escalável e altamente disponível na AWS usando Console (ClickOps).

[![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?style=flat-square&logo=amazon-aws)](https://aws.amazon.com/)
[![WordPress](https://img.shields.io/badge/CMS-WordPress-21759B?style=flat-square&logo=wordpress)](https://wordpress.org/)
[![Docker](https://img.shields.io/badge/Container-Docker-2496ED?style=flat-square&logo=docker)](https://www.docker.com/)
[![MySQL](https://img.shields.io/badge/Database-MySQL_8.0-4479A1?style=flat-square&logo=mysql)](https://www.mysql.com/)

---

## 🎯 Sobre Este Projeto

Este projeto demonstra a **implementação manual via AWS Console (ClickOps)** de uma infraestrutura escalável e altamente disponível para WordPress na AWS. A arquitetura inclui instâncias EC2 configuradas com Docker, banco de dados MySQL no RDS, EFS para armazenamento compartilhado, Load Balancer e Auto Scaling Group.

### 📌 Comparação com IaC

Este é o projeto de referência para comparação com a versão automatizada em Terraform:
- 🔧 **Projeto Terraform (IaC):** https://github.com/jlui70/clickops-to-iac-wordpress

---

## 📚 Índice

- [Descrição](#descrição)
- [Arquitetura](#arquitetura)
- [Parte 1: Configuração da VPC](#parte-1-configuração-da-vpc)
- [Parte 2: Configuração de Security Groups](#parte-2-configuração-de-security-groups)
- [Parte 3: Configuração do SSM](#parte-3-configuração-do-ssm)
- [Parte 4: Criação do Banco de Dados no RDS](#parte-4-criação-do-banco-de-dados-no-rds)
- [Parte 5: Configuração do EFS](#parte-5-configuração-do-efs)
- [Parte 6: Configuração do Template de Instância EC2](#parte-6-configuração-do-template-de-instância-ec2)
- [Parte 7: Configuração do Elastic Load Balancer](#parte-7-configuração-do-elastic-load-balancer)
- [Parte 8: Configuração do Auto Scaling Group](#parte-8-configuração-do-auto-scaling-group)
- [Parte 9: Testes](#parte-9-testes)

---


## **Descrição**

Este projeto tem como objetivo criar uma infraestrutura escalável e altamente disponível para uma aplicação WordPress na AWS. A arquitetura inclui instâncias EC2 configuradas com Docker, um banco de dados MySQL no RDS, o uso do EFS para armazenar arquivos estáticos e a implementação de um Load Balancer e Auto Scaling Group para garantir a alta disponibilidade e escalabilidade. Toda a configuração será realizada por meio da AWS, com a documentação detalhando o processo de configuração e implementação.

## **Arquitetura**

![](Img/arquitetura-diagrama.png)

## Parte 1: Configuração da VPC

### 1.1 Criar a VPC
1. Acesse o Console de Gerenciamento da AWS.
2. No painel de navegação à esquerda, clique em **VPC**.
3. Clique em **Your VPCs** e depois em **Create VPC**.
4. Selecione a opção **VPC and more**.
5. Preencha as configurações:
   - **Name tag**: `wp-docker`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - **Número de zonas disponíveis**: 2
   - **Número de sub-redes públicas**: 2
   - **Número de sub-redes privadas**: 2
   - **NAT gateways**: **None** (Vamos criar manualmente o NAT Gateway posteriormente).
6. Clique em **Create VPC**.

![](Img/vpc.png)


### 1.2 Criar o NAT Gateway
1. No painel de navegação à esquerda, clique em **NAT Gateways**.
2. Clique em **Create NAT Gateway**.
3. Preencha as configurações:
   - **Nome**: `wp-natgateway`
   - **Sub-rede pública**: Escolha uma das sub-redes públicas criadas pela sua VPC (por exemplo, `sub-rede-publica-1`).
   - **Elastic IP**: Clique em **Allocate Elastic IP** para alocar um IP elástico e associá-lo ao NAT Gateway.
4. Clique em **Create NAT Gateway**.
5. Aguarde até que o estado do seu NAT Gateway fique como **Available** antes de prosseguir.

### 1.3 Associar o NAT Gateway nas Tabelas de Roteamento
1. No painel de navegação à esquerda, clique em **Route Tables**.
2. Identifique as duas tabelas de roteamento associadas às sub-redes privadas criadas pela sua VPC.
   - **Tabela de roteamento para a sub-rede privada 1**.
   - **Tabela de roteamento para a sub-rede privada 2**.
3. Para cada tabela de roteamento, clique no **ID** da tabela e depois em **Edit routes**.
4. Adicione a seguinte rota:
   - **Destination**: `0.0.0.0/0`
   - **Target**: Selecione **NAT Gateway** e escolha o ID do NAT Gateway que você criou (por exemplo, `wp-natgateway`).
5. Clique em **Save routes**.

![](Img/rota-natgateway.png)

### 1.4 Concluir a Configuração
Agora que a VPC foi criada com suas sub-redes públicas e privadas, o NAT Gateway está configurado e as rotas foram associadas corretamente.

## Parte 2: Configuração dos Security Groups

## 2.1 Security Group - ApplicationLoadBalancer-SG
1. Navegue até **EC2** e clique em **Security Groups**.
2. Clique em **Create Security Group**.
3. Preencha as configurações:
   - **Nome:** ApplicationLoadBalancer-SG
   - **VPC:** Selecione a mesma VPC utilizada no projeto (ex: wp-docker).

### Inbound rules:
   - **HTTP**: Port **80**, Source: **0.0.0.0/0**
   - **HTTPS**: Port **443**, Source: **0.0.0.0/0**

### Outbound rules:
   - **HTTP**: Port **80**, Destination: **ApplicationServer-SG**
   - **HTTPS**: Port **443**, Destination: **ApplicationServer-SG**

4. Clique em **Create**.

## 2.2 Security Group - ApplicationServer-SG
1. Navegue até **EC2** e clique em **Security Groups**.
2. Clique em **Create Security Group**.
3. Preencha as configurações:
   - **Nome:** ApplicationServer-SG
   - **VPC:** Selecione a mesma VPC utilizada no projeto (wp-docker).

### Inbound rules:
   - **HTTP**: Port **80**, Source: **ApplicationLoadBalancer-SG**
   - **HTTPS**: Port **443**, Source: **ApplicationLoadBalancer-SG**
   - **MySQL/Aurora**: Port **3306**, Source: **Database-SG**
   - **NFS**: Port **2049**, Source: **EFS-SG**

### Outbound rules:
   - **MySQL/Aurora**: Port **3306**, Destination: **Database-SG**
   - **NFS**: Port **2049**, Destination: **EFS-SG**
   - **HTTPS**: Port **443**, Source: **0.0.0.0/0**

4. Clique em **Create**.

## 2.3 Security Group - Database-SG
1. Navegue até **EC2** e clique em **Security Groups**.
2. Clique em **Create Security Group**.
3. Preencha as configurações:
   - **Nome:** Database-SG
   - **VPC:** Selecione a mesma VPC utilizada no projeto (wp-docker).

### Inbound rules:
   - **MySQL/Aurora**: Port **3306**, Source: **ApplicationServer-SG**

### Outbound rules:
   - **Nenhuma necessária** (o banco de dados não precisa se comunicar externamente).

4. Clique em **Create**.

## 2.4 Security Group - EFS-SG
1. Navegue até **EC2** e clique em **Security Groups**.
2. Clique em **Create Security Group**.
3. Preencha as configurações:
   - **Nome:** EFS-SG
   - **VPC:** Selecione a mesma VPC utilizada no projeto (wp-docker).

### Inbound rules:
   - **NFS**: Port **2049**, Source: **ApplicationServer-SG**
   - **NFS**: Port **2049**, Source: **EFS-SG** (O EFS precisa se conectar a si mesmo para sincronização e bloqueio de arquivos.)

### Outbound rules:
   - **All Traffic**, Destination: **0.0.0.0/0**

4. Clique em **Create**.

![](Img/sg-group.png)

Agora, os **Security Groups** estarão configurados e prontos para uso com as instâncias EC2, Load Balancer, EFS e RDS.

## Parte 3: Configuração do SSM

### Criar Função IAM para o SSM
1. Navegue até **IAM** no Console de Gerenciamento da AWS.
2. No menu à esquerda, clique em **Roles** e depois em **Create role**.
3. Selecione **AWS service** como tipo de entidade confiável.
4. Escolha **EC2** como o serviço que usará a role.
5. Na lista de permissões, selecione **Amazon EC2 Role for SSM** para permitir que a instância EC2 utilize o Systems Manager.

![](Img/iam-ssm.png)

6. Clique em **Next: Tags**. (Você pode adicionar tags se necessário, mas isso é opcional).
7. Clique em **Next: Review**.
8. Dê um nome à função, como `EC2-SSM-Role`.
9. Clique em **Create role** para criar a função.

## Parte 4: Criação do Banco de Dados no RDS

### 4.1 Criar Grupo de Sub-redes para o RDS
1. Navegue até o **RDS** no Console de Gerenciamento da AWS.
2. No menu lateral, clique em **Subnet groups** e depois em **Create DB Subnet Group**.
3. Preencha as configurações:
   - **Name**: `subnetdb-group`
   - **Description**: Grupo de sub-redes para o banco de dados.
   - **VPC**: Selecione a VPC criada anteriormente (`wp-docker`).
4. Em **Subnets**, selecione:
   - **Availability Zones**: Escolha pelo menos duas zonas de disponibilidade diferentes.
   - **Private Subnets**: Selecione as sub-redes privadas da VPC.
5. Clique em **Create** para finalizar a criação do grupo de sub-redes.

![](Img/subnet-db.png)

### 4.2 Criar Banco de Dados RDS
1. Navegue até o **RDS** no Console de Gerenciamento da AWS.
2. Na barra lateral esquerda, clique em **Databases** e depois em **Create database**.
3. Escolha a opção **Standard Create**.
4. Selecione **MySQL** como o banco de dados.
5. Escolha a versão **MySQL 8.4.3** (compatível com o WordPress).
6. Selecione o template **Free tier**.
7. Preencha as configurações:
   - **DB instance identifier**: Escolha um nome para o banco de dados (por exemplo, `wordpressdb`).
   - **Master username**: Escolha um nome de usuário para o banco de dados (ex: `admin`).
   - **Master password**: Clique em **Generate password** ou insira uma senha manualmente.
8. **Instance configuration**:
   - **DB instance class**: Selecione `db.t3.micro` (Nível gratuito).
   - **Storage type**: Selecione **SSD gp3** (mais novo e mais barato).
   - **Allocated storage**: Defina para **20 GB**.
   - **Enable autoscaling**: Defina um limite de **50 GB**.
9. **Connectivity**:
   - **VPC**: Selecione a VPC criada anteriormente (`wp-docker`).
   - **Subnet group**: Selecione o **subnetdb-group** criado anteriormente.
   - **Public access**: Defina como **No** (não permitir acesso público).
   - **VPC security group**: Selecione **Database-SG**.
10. **Additional configuration**:
    - **DB name**: Insira o nome para o banco de dados, como `wordpressdb`.
    - **Backup**: Desmarque a opção **Enable backups** (para desabilitar o backup automático).
11. Clique em **Create database** e aguarde a criação do banco de dados.

![](Img/rds.png)

Agora o banco de dados RDS está configurado, com o MySQL 8.4.3, e pronto para ser utilizado pela aplicação WordPress.

## Parte 5: Configuração do EFS

### 5.1 Criar File System EFS
1. Navegue até **EFS** no Console de Gerenciamento da AWS.
2. Clique em **Create file system**.
3. Selecione a opção **Customize** para personalizar as configurações.
4. Preencha as configurações:
   - **Name**: Insira o nome do sistema de arquivos, como `wordpress-efs`.
   - **Storage class**: Deixe como **Regional**.
   - **Backup**: Desmarque a opção **Enable backups** (para desabilitar o backup automático).
5. **Network**:
   - **VPC**: Selecione a **VPC** que você criou anteriormente (`wp-docker`).
6. **Mount targets**:
   - Em **Availability Zones**, selecione a **subnet privada** de cada zona de disponibilidade para garantir que o EFS esteja disponível apenas na sub-rede privada.
   - Em **Security groups**, remova o grupo de segurança padrão e insira o grupo de segurança **SG-EFS** que você criou anteriormente.
7. Clique em **Create** para criar o EFS.

 ![](Img/efs.png)

### 5.2 Obter o DNS do EFS
1. Após a criação, clique no **ID** do sistema de arquivos para acessar as informações detalhadas.
2. Na página de detalhes do EFS, copie o **DNS name** do EFS para usar no script user_Data.

![](Img/efs-dns.png)   

Agora o sistema de arquivos EFS está configurado e pronto para ser montado nas instâncias EC2.

## Parte 6: Configuração do Template de Instância EC2

### 6.1 Criar Template de Lançamento para Instância EC2
1. Navegue até **EC2** no Console de Gerenciamento da AWS.
2. No menu à esquerda, clique em **Launch Templates** e depois em **Create launch template**.
3. Preencha as configurações conforme abaixo:
   - **Template name**: Insira o nome do template, como `wordpress-temp`.
   - **Template version**: Defina como **v1**.
   - **Auto Scaling guidance**: Deixe como **OK**.
   - **AMI**: Selecione **Amazon Linux 2** (ou a versão do Amazon Linux que você deseja usar).
   - **Instance type**: Escolha o tipo de instância adequado, como `t2.micro` para o nível gratuito.
   
4. **Network settings**:
   - Selecione a VPC criada anteriormente.
   - Marque a opção **Do not include in launch template**  Isso será configurado posteriormente no Auto Scaling Group.

5. **Security Group**:
   - Selecione o **Security Group** criado para o servidor de aplicação, `applicationServer-SG`.

6. **Advanced details**:
   - **IAM Instance Profile**: Selecione o perfil IAM da instância que foi configurado anteriormente (por exemplo, `EC2-SSM-Role`).
   - **User data**: No campo **User data**, insira o script necessário para instalar o **Docker**, **Docker Compose**, e configurar os contêineres para **WordPress** e **MySQL** utilizando o **Docker Compose**.

     ```bash
      #!/bin/bash

      # Atualizar pacotes e instalar dependências
      sudo yum update -y
      sudo yum install -y docker nfs-utils

      # Instalar o SSM Agent
      sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
      sudo systemctl enable amazon-ssm-agent
      sudo systemctl start amazon-ssm-agent

      # Habilitar e iniciar o Docker
      sudo systemctl enable docker
      sudo systemctl start docker

      # Adicionar ec2-user ao grupo docker
      sudo usermod -aG docker ec2-user

      # Instalar Docker Compose
      sudo curl -SL https://github.com/docker/compose/releases/download/v2.32.4/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
      sudo chmod +x /usr/local/bin/docker-compose

      # Montar EFS
      EFS_DNS="seu-caminho-efs"  # DNS do EFS diretamente
      EFS_PATH="/mnt/efs/wordpress"  # Caminho do diretório EFS

      # Criar diretório EFS e montar
      sudo mkdir -p $EFS_PATH
      sudo chmod -R 777 $EFS_PATH
      echo "$EFS_DNS:/ $EFS_PATH nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
      sudo mount -a

      # Definir as variáveis de ambiente para o WordPress
      WORDPRESS_DB_HOST="seu-endpoint-rds"
      WORDPRESS_DB_USER="seu-usuario"
      WORDPRESS_DB_PASSWORD="sua-senha"
      WORDPRESS_DB_NAME="wordpressdb"

      # Criar o arquivo docker-compose.yaml com as variáveis de ambiente
      cat <<EOL > /home/ec2-user/docker-compose.yaml

      services:
      wordpress:
         image: wordpress:latest
         container_name: wordpress
         restart: always
         volumes:
            - $EFS_PATH:/var/www/html  # Monta os arquivos do WordPress no EFS
         ports:
            - "80:80"
         environment:
            WORDPRESS_DB_HOST: "$WORDPRESS_DB_HOST"
            WORDPRESS_DB_USER: "$WORDPRESS_DB_USER"
            WORDPRESS_DB_PASSWORD: "$WORDPRESS_DB_PASSWORD"
            WORDPRESS_DB_NAME: "$WORDPRESS_DB_NAME"
      EOL

      # Inicializar o container do WordPress com Docker Compose
      sudo -u ec2-user bash -c "cd /home/ec2-user && docker-compose -f docker-compose.yaml up -d"

      echo "Instalação concluída! WordPress está rodando e conectado ao RDS."

     ```
**Observação**: Substitua o endereço EFS pelo o seu, e insira suas informações no Banco de Dados.

7. Clique em **Create launch template** para criar o template de lançamento.

Agora temos o template de lançamento EC2 configurado com o Amazon Linux 2, as configurações de rede e segurança adequadas, e com o IAM e agentes do Systems Manager configurados no **User data**.

![](Img/wp-template.png)

## Parte 7: Configuração do Elastic Load Balancer

### 7.1 Criar o Target Group
1. Vá para o **EC2** no Console de Gerenciamento da AWS.
2. No menu à esquerda, em **Target Groups**, clique em **Create target group**.
3. Preencha os campos da seguinte forma:
   - **Target group name**: `wordpress-tg`
   - **Target type**: **Instances**
   - **Protocol**: **HTTP**
   - **Port**: **80**
   - **VPC**: Selecione a VPC que você criou anteriormente.
4. Clique em **Create** para criar o grupo de destino.

![](Img/wordpress-tg.png)

### 7.2 Criar o Load Balancer
1. Vá para **Load Balancers** no Console de Gerenciamento da AWS.
2. Clique em **Create Load Balancer** e selecione **Application Load Balancer**.
3. Preencha as configurações conforme abaixo:
   - **Name**: `wordpress-lb`
   - **Scheme**: **Internet-facing**
   - **IP address type**: **IPv4**
   - **VPC**: Selecione a VPC que você criou anteriormente.
   - **Availability Zones**: Selecione as duas zonas de disponibilidade e suas sub-redes públicas.
   - **Security Groups**: Selecione o **Security Group** `ApplicationLoadBalancer-SG`.
4. Em **Listeners and routing**, selecione o grupo de destino **wordpress-tg** que você criou anteriormente.
5. Clique em **Create load balancer** para finalizar a criação do Load Balancer.

![](Img/elb.png)

## Parte 8: Configuração do Auto Scaling Group

### 8 Criar e Configurar o Auto Scaling Group
1. No menu à esquerda, vá em **Auto Scaling Groups** e clique em **Create Auto Scaling group**.
2. Preencha as configurações conforme abaixo:
   - **Auto Scaling group name**: `wordpress-asg`
   - **Launch template**: Escolha o **Launch template** que você criou anteriormente (`wordpress-temp`).

   - **VPC**: Selecione a VPC que você criou.
   - **Availability Zones**: Selecione as **zonas privadas** da sua VPC.

 ![](Img/asg-vpc.png)

   - **Attach to an existing load balancer**: Marque essa opção e selecione o **Load Balancer** `wordpress-lb` e o grupo de destino `wordpress-tg`.

 ![](Img/asg-alb.png)

   - **Health checks**: Ative a opção **Elastic Load Balancing health checks**.
   - **Desired capacity**: 2
   - **Minimum capacity**: 1
   - **Maximum capacity**: 2

![](Img/asg-group-size.png)

3. Clique em **Next** para configurar as notificações, se necessário.
4. Configure as notificações por email, caso desejado.
5. Clique em **Create Auto Scaling group** para finalizar a criação do Auto Scaling Group.

![](Img/asg-group.png)

## Parte 9: Testes

### 9.1 Verificar o estado do Load Balancer
1. Aguarde as instâncias EC2 serem inicializadas e o Auto Scaling Group realizar a distribuição das instâncias.
2. Vá até o **Load Balancer** no Console de EC2 e verifique se o estado do Load Balancer está como **Active**.
3. Certifique-se de que o status de saúde das instâncias no **Target Group** também esteja saudável (Healthy).

### 9.2 Acessar a aplicação pelo Load Balancer
1. Copie o **DNS name** do **Load Balancer** gerado.
2. Abra o seu navegador e cole o **DNS name** do Load Balancer na barra de endereços.
3. Se tudo estiver configurado corretamente, você deverá ver a página inicial do **WordPress**.

### 9.3 Acessar as instâncias EC2 pelo SSM
1. Vá até o **SSM** (AWS Systems Manager) no Console de Gerenciamento da AWS.
2. No menu à esquerda, clique em **Instances & Nodes**.
3. Selecione **Managed Instances** e procure pelas instâncias EC2 que estão sendo gerenciadas pelo SSM.
4. Clique na instância que deseja acessar e, em seguida, clique em **Connect**.
5. Escolha a opção **Session Manager** para iniciar uma sessão na instância EC2 diretamente pelo console, sem a necessidade de uma chave SSH.

![](Img/ec2-ssm.png)  

### 9.4 Configurar o WordPress
1. Ao acessar o WordPress pela primeira vez, será solicitado que você configure uma conta de login.

![](Img/wordpress.png)

2. Siga o assistente de instalação do WordPress para configurar o banco de dados, escolher o idioma e definir as credenciais de administrador.
3. Após a configuração, você poderá acessar o painel administrativo do WordPress para gerenciar sua aplicação.

![](Img/dashboard-wp.png)

---

## 💰 Custos Estimados

| Recurso | Custo Estimado | Observações |
|---------|----------------|-------------|
| NAT Gateway | ~$1.00/dia | Componente mais caro |
| ALB | ~$0.50/dia | Load balancer |
| EC2 t2.micro (2x) | Free tier ou ~$0.25/dia | Elegível para free tier |
| RDS t3.micro | Free tier ou ~$0.50/dia | Elegível para free tier |
| EFS | ~$0.10-0.20/dia | Baseado em uso |

**Total estimado**: $2-3/dia durante testes

⚠️ **IMPORTANTE**: Configure billing alerts e destrua os recursos quando não estiver usando!

---

## 📞 Links Úteis

- **Repositório:** https://github.com/jlui70/clickops-aws-wordpress
- **Projeto com Terraform (IaC):** https://github.com/jlui70/clickops-to-iac-wordpress
- **AWS Console:** https://console.aws.amazon.com/
- **AWS Well-Architected:** https://aws.amazon.com/architecture/well-architected/
- **WordPress on AWS:** https://aws.amazon.com/getting-started/hands-on/launch-a-wordpress-website/
- **Docker Hub - WordPress:** https://hub.docker.com/_/wordpress

---

## 📄 Licença

Este projeto é open source e está disponível sob a [MIT License](LICENSE).

---

<p align="center">
  <sub>Construído com ❤️ para demonstração de implementação manual na AWS</sub>
</p>
