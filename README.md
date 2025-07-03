Projeto-Docker -  WordPress na AWS

Índice
•	Descrição
•	Arquitetura
•	Parte 1: Configuração da VPC
•	Parte 2: Configuração de Security Groups
•	Parte 3: Configuração do SSM
•	Parte 4: Criação do Banco de Dados no RDS
•	Parte 5: Configuração do EFS
•	Parte 6: Configuração do Template de Instância EC2
•	Parte 7: Configuração do Elastic Load Balancer
•	Parte 8: Configuração do Auto Scaling Group
•	Parte 9: Testes
•	Referências

Descrição
Este projeto tem como objetivo criar uma infraestrutura escalável e altamente disponível para uma aplicação WordPress na AWS. A arquitetura inclui instâncias EC2 configuradas com Docker, um banco de dados MySQL no RDS, o uso do EFS para armazenar arquivos estáticos e a implementação de um Load Balancer e Auto Scaling Group para garantir a alta disponibilidade e escalabilidade. Toda a configuração será realizada por meio da AWS, com a documentação detalhando o processo de configuração e implementação.
Arquitetura
 
Parte 1: Configuração da VPC
1.1 Criar a VPC
1.	Acesse o Console de Gerenciamento da AWS.
2.	No painel de navegação à esquerda, clique em VPC.
3.	Clique em Your VPCs e depois em Create VPC.
4.	Selecione a opção VPC and more.
5.	Preencha as configurações:
o	Name tag: wp-docker
o	IPv4 CIDR block: 10.0.0.0/16
o	Número de zonas disponíveis: 2
o	Número de sub-redes públicas: 2
o	Número de sub-redes privadas: 2
o	NAT gateways: None (Vamos criar manualmente o NAT Gateway posteriormente).
6.	Clique em Create VPC.
 
1.2 Criar o NAT Gateway
1.	No painel de navegação à esquerda, clique em NAT Gateways.
2.	Clique em Create NAT Gateway.
3.	Preencha as configurações:
o	Nome: wp-natgateway
o	Sub-rede pública: Escolha uma das sub-redes públicas criadas pela sua VPC (por exemplo, sub-rede-publica-1).
o	Elastic IP: Clique em Allocate Elastic IP para alocar um IP elástico e associá-lo ao NAT Gateway.
4.	Clique em Create NAT Gateway.
5.	Aguarde até que o estado do seu NAT Gateway fique como Available antes de prosseguir.
1.3 Associar o NAT Gateway nas Tabelas de Roteamento
1.	No painel de navegação à esquerda, clique em Route Tables.
2.	Identifique as duas tabelas de roteamento associadas às sub-redes privadas criadas pela sua VPC.
o	Tabela de roteamento para a sub-rede privada 1.
o	Tabela de roteamento para a sub-rede privada 2.
3.	Para cada tabela de roteamento, clique no ID da tabela e depois em Edit routes.
4.	Adicione a seguinte rota:
o	Destination: 0.0.0.0/0
o	Target: Selecione NAT Gateway e escolha o ID do NAT Gateway que você criou (por exemplo, wp-natgateway).
5.	Clique em Save routes.
 
1.4 Concluir a Configuração
Agora que a VPC foi criada com suas sub-redes públicas e privadas, o NAT Gateway está configurado e as rotas foram associadas corretamente.
Parte 2: Configuração dos Security Groups
2.1 Security Group - ApplicationLoadBalancer-SG
1.	Navegue até EC2 e clique em Security Groups.
2.	Clique em Create Security Group.
3.	Preencha as configurações:
o	Nome: ApplicationLoadBalancer-SG
o	VPC: Selecione a mesma VPC utilizada no projeto (ex: wp-docker).
Inbound rules:
•	HTTP: Port 80, Source: 0.0.0.0/0
•	HTTPS: Port 443, Source: 0.0.0.0/0
Outbound rules:
•	HTTP: Port 80, Destination: ApplicationServer-SG
•	HTTPS: Port 443, Destination: ApplicationServer-SG
4.	Clique em Create.
2.2 Security Group - ApplicationServer-SG
1.	Navegue até EC2 e clique em Security Groups.
2.	Clique em Create Security Group.
3.	Preencha as configurações:
o	Nome: ApplicationServer-SG
o	VPC: Selecione a mesma VPC utilizada no projeto (wp-docker).
Inbound rules:
•	HTTP: Port 80, Source: ApplicationLoadBalancer-SG
•	HTTPS: Port 443, Source: ApplicationLoadBalancer-SG
•	MySQL/Aurora: Port 3306, Source: Database-SG
•	NFS: Port 2049, Source: EFS-SG
Outbound rules:
•	MySQL/Aurora: Port 3306, Destination: Database-SG
•	NFS: Port 2049, Destination: EFS-SG
•	HTTPS: Port 443, Source: 0.0.0.0/0
4.	Clique em Create.
2.3 Security Group - Database-SG
1.	Navegue até EC2 e clique em Security Groups.
2.	Clique em Create Security Group.
3.	Preencha as configurações:
o	Nome: Database-SG
o	VPC: Selecione a mesma VPC utilizada no projeto (wp-docker).
Inbound rules:
•	MySQL/Aurora: Port 3306, Source: ApplicationServer-SG
Outbound rules:
•	Nenhuma necessária (o banco de dados não precisa se comunicar externamente).
4.	Clique em Create.
2.4 Security Group - EFS-SG
1.	Navegue até EC2 e clique em Security Groups.
2.	Clique em Create Security Group.
3.	Preencha as configurações:
o	Nome: EFS-SG
o	VPC: Selecione a mesma VPC utilizada no projeto (wp-docker).
Inbound rules:
•	NFS: Port 2049, Source: ApplicationServer-SG
•	NFS: Port 2049, Source: EFS-SG (O EFS precisa se conectar a si mesmo para sincronização e bloqueio de arquivos.)
Outbound rules:
•	All Traffic, Destination: 0.0.0.0/0
4.	Clique em Create.
 
Agora, os Security Groups estarão configurados e prontos para uso com as instâncias EC2, Load Balancer, EFS e RDS.
Parte 3: Configuração do SSM
Criar Função IAM para o SSM
1.	Navegue até IAM no Console de Gerenciamento da AWS.
2.	No menu à esquerda, clique em Roles e depois em Create role.
3.	Selecione AWS service como tipo de entidade confiável.
4.	Escolha EC2 como o serviço que usará a role.
5.	Na lista de permissões, selecione Amazon EC2 Role for SSM para permitir que a instância EC2 utilize o Systems Manager.
 
6.	Clique em Next: Tags. (Você pode adicionar tags se necessário, mas isso é opcional).
7.	Clique em Next: Review.
8.	Dê um nome à função, como EC2-SSM-Role.
9.	Clique em Create role para criar a função.
Parte 4: Criação do Banco de Dados no RDS
4.1 Criar Grupo de Sub-redes para o RDS
1.	Navegue até o RDS no Console de Gerenciamento da AWS.
2.	No menu lateral, clique em Subnet groups e depois em Create DB Subnet Group.
3.	Preencha as configurações:
o	Name: subnetdb-group
o	Description: Grupo de sub-redes para o banco de dados.
o	VPC: Selecione a VPC criada anteriormente (wp-docker).
4.	Em Subnets, selecione:
o	Availability Zones: Escolha pelo menos duas zonas de disponibilidade diferentes.
o	Private Subnets: Selecione as sub-redes privadas da VPC.
5.	Clique em Create para finalizar a criação do grupo de sub-redes.
 
4.2 Criar Banco de Dados RDS
1.	Navegue até o RDS no Console de Gerenciamento da AWS.
2.	Na barra lateral esquerda, clique em Databases e depois em Create database.
3.	Escolha a opção Standard Create.
4.	Selecione MySQL como o banco de dados.
5.	Escolha a versão MySQL 8.4.3 (compatível com o WordPress).
6.	Selecione o template Free tier.
7.	Preencha as configurações:
o	DB instance identifier: Escolha um nome para o banco de dados (por exemplo, wordpressdb).
o	Master username: Escolha um nome de usuário para o banco de dados (ex: admin).
o	Master password: Clique em Generate password ou insira uma senha manualmente.
8.	Instance configuration:
o	DB instance class: Selecione db.t3.micro (Nível gratuito).
o	Storage type: Selecione SSD gp3 (mais novo e mais barato).
o	Allocated storage: Defina para 20 GB.
o	Enable autoscaling: Defina um limite de 50 GB.
9.	Connectivity:
o	VPC: Selecione a VPC criada anteriormente (wp-docker).
o	Subnet group: Selecione o subnetdb-group criado anteriormente.
o	Public access: Defina como No (não permitir acesso público).
o	VPC security group: Selecione Database-SG.
10.	Additional configuration:
o	DB name: Insira o nome para o banco de dados, como wordpressdb.
o	Backup: Desmarque a opção Enable backups (para desabilitar o backup automático).
11.	Clique em Create database e aguarde a criação do banco de dados.
 
Agora o banco de dados RDS está configurado, com o MySQL 8.4.3, e pronto para ser utilizado pela aplicação WordPress.
Parte 5: Configuração do EFS
5.1 Criar File System EFS
1.	Navegue até EFS no Console de Gerenciamento da AWS.
2.	Clique em Create file system.
3.	Selecione a opção Customize para personalizar as configurações.
4.	Preencha as configurações:
o	Name: Insira o nome do sistema de arquivos, como wordpress-efs.
o	Storage class: Deixe como Regional.
o	Backup: Desmarque a opção Enable backups (para desabilitar o backup automático).
5.	Network:
o	VPC: Selecione a VPC que você criou anteriormente (wp-docker).
6.	Mount targets:
o	Em Availability Zones, selecione a subnet privada de cada zona de disponibilidade para garantir que o EFS esteja disponível apenas na sub-rede privada.
o	Em Security groups, remova o grupo de segurança padrão e insira o grupo de segurança SG-EFS que você criou anteriormente.
7.	Clique em Create para criar o EFS.
 
5.2 Obter o DNS do EFS
1.	Após a criação, clique no ID do sistema de arquivos para acessar as informações detalhadas.
2.	Na página de detalhes do EFS, copie o DNS name do EFS para usar no script user_Data.
 
Agora o sistema de arquivos EFS está configurado e pronto para ser montado nas instâncias EC2.
Parte 6: Configuração do Template de Instância EC2
6.1 Criar Template de Lançamento para Instância EC2
1.	Navegue até EC2 no Console de Gerenciamento da AWS.
2.	No menu à esquerda, clique em Launch Templates e depois em Create launch template.
3.	Preencha as configurações conforme abaixo:
o	Template name: Insira o nome do template, como wordpress-temp.
o	Template version: Defina como v1.
o	Auto Scaling guidance: Deixe como OK.
o	AMI: Selecione Amazon Linux 2 (ou a versão do Amazon Linux que você deseja usar).
o	Instance type: Escolha o tipo de instância adequado, como t2.micro para o nível gratuito.
4.	Network settings:
o	Selecione a VPC criada anteriormente.
o	Marque a opção Do not include in launch template Isso será configurado posteriormente no Auto Scaling Group.
5.	Security Group:
o	Selecione o Security Group criado para o servidor de aplicação, applicationServer-SG.
6.	Advanced details:
o	IAM Instance Profile: Selecione o perfil IAM da instância que foi configurado anteriormente (por exemplo, EC2-SSM-Role).
o	User data: No campo User data, insira o script necessário para instalar o Docker, Docker Compose, e configurar os contêineres para WordPress e MySQL utilizando o Docker Compose.
o	 #!/bin/bash
o	
o	 # Atualizar pacotes e instalar dependências
o	 sudo yum update -y
o	 sudo yum install -y docker nfs-utils
o	
o	 # Instalar o SSM Agent
o	 sudo yum install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm
o	 sudo systemctl enable amazon-ssm-agent
o	 sudo systemctl start amazon-ssm-agent
o	
o	 # Habilitar e iniciar o Docker
o	 sudo systemctl enable docker
o	 sudo systemctl start docker
o	
o	 # Adicionar ec2-user ao grupo docker
o	 sudo usermod -aG docker ec2-user
o	
o	 # Instalar Docker Compose
o	 sudo curl -SL https://github.com/docker/compose/releases/download/v2.32.4/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
o	 sudo chmod +x /usr/local/bin/docker-compose
o	
o	 # Montar EFS
o	 EFS_DNS="seu-caminho-efs"  # DNS do EFS diretamente
o	 EFS_PATH="/mnt/efs/wordpress"  # Caminho do diretório EFS
o	
o	 # Criar diretório EFS e montar
o	 sudo mkdir -p $EFS_PATH
o	 sudo chmod -R 777 $EFS_PATH
o	 echo "$EFS_DNS:/ $EFS_PATH nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
o	 sudo mount -a
o	
o	 # Definir as variáveis de ambiente para o WordPress
o	 WORDPRESS_DB_HOST="seu-endpoint-rds"
o	 WORDPRESS_DB_USER="seu-usuario"
o	 WORDPRESS_DB_PASSWORD="sua-senha"
o	 WORDPRESS_DB_NAME="wordpressdb"
o	
o	 # Criar o arquivo docker-compose.yaml com as variáveis de ambiente
o	 cat <<EOL > /home/ec2-user/docker-compose.yaml
o	
o	 services:
o	 wordpress:
o	    image: wordpress:latest
o	    container_name: wordpress
o	    restart: always
o	    volumes:
o	       - $EFS_PATH:/var/www/html  # Monta os arquivos do WordPress no EFS
o	    ports:
o	       - "80:80"
o	    environment:
o	       WORDPRESS_DB_HOST: "$WORDPRESS_DB_HOST"
o	       WORDPRESS_DB_USER: "$WORDPRESS_DB_USER"
o	       WORDPRESS_DB_PASSWORD: "$WORDPRESS_DB_PASSWORD"
o	       WORDPRESS_DB_NAME: "$WORDPRESS_DB_NAME"
o	 EOL
o	
o	 # Inicializar o container do WordPress com Docker Compose
o	 sudo -u ec2-user bash -c "cd /home/ec2-user && docker-compose -f docker-compose.yaml up -d"
o	
o	 echo "Instalação concluída! WordPress está rodando e conectado ao RDS."
Observação: Substitua o endereço EFS pelo o seu, e insira suas informações no Banco de Dados.
7.	Clique em Create launch template para criar o template de lançamento.
Agora temos o template de lançamento EC2 configurado com o Amazon Linux 2, as configurações de rede e segurança adequadas, e com o IAM e agentes do Systems Manager configurados no User data.
 
Parte 7: Configuração do Elastic Load Balancer
7.1 Criar o Target Group
1.	Vá para o EC2 no Console de Gerenciamento da AWS.
2.	No menu à esquerda, em Target Groups, clique em Create target group.
3.	Preencha os campos da seguinte forma:
o	Target group name: wordpress-tg
o	Target type: Instances
o	Protocol: HTTP
o	Port: 80
o	VPC: Selecione a VPC que você criou anteriormente.
4.	Clique em Create para criar o grupo de destino.
 
7.2 Criar o Load Balancer
1.	Vá para Load Balancers no Console de Gerenciamento da AWS.
2.	Clique em Create Load Balancer e selecione Application Load Balancer.
3.	Preencha as configurações conforme abaixo:
o	Name: wordpress-lb
o	Scheme: Internet-facing
o	IP address type: IPv4
o	VPC: Selecione a VPC que você criou anteriormente.
o	Availability Zones: Selecione as duas zonas de disponibilidade e suas sub-redes públicas.
o	Security Groups: Selecione o Security Group ApplicationLoadBalancer-SG.
4.	Em Listeners and routing, selecione o grupo de destino wordpress-tg que você criou anteriormente.
5.	Clique em Create load balancer para finalizar a criação do Load Balancer.
 
Parte 8: Configuração do Auto Scaling Group
8 Criar e Configurar o Auto Scaling Group
1.	No menu à esquerda, vá em Auto Scaling Groups e clique em Create Auto Scaling group.
2.	Preencha as configurações conforme abaixo:
o	Auto Scaling group name: wordpress-asg
o	Launch template: Escolha o Launch template que você criou anteriormente (wordpress-temp).
o	VPC: Selecione a VPC que você criou.
o	Availability Zones: Selecione as zonas privadas da sua VPC.
 
•	Attach to an existing load balancer: Marque essa opção e selecione o Load Balancer wordpress-lb e o grupo de destino wordpress-tg.
 
•	Health checks: Ative a opção Elastic Load Balancing health checks.
•	Desired capacity: 2
•	Minimum capacity: 1
•	Maximum capacity: 2
 
3.	Clique em Next para configurar as notificações, se necessário.
4.	Configure as notificações por email, caso desejado.
5.	Clique em Create Auto Scaling group para finalizar a criação do Auto Scaling Group.
 
Parte 9: Testes
9.1 Verificar o estado do Load Balancer
1.	Aguarde as instâncias EC2 serem inicializadas e o Auto Scaling Group realizar a distribuição das instâncias.
2.	Vá até o Load Balancer no Console de EC2 e verifique se o estado do Load Balancer está como Active.
3.	Certifique-se de que o status de saúde das instâncias no Target Group também esteja saudável (Healthy).
9.2 Acessar a aplicação pelo Load Balancer
1.	Copie o DNS name do Load Balancer gerado.
2.	Abra o seu navegador e cole o DNS name do Load Balancer na barra de endereços.
3.	Se tudo estiver configurado corretamente, você deverá ver a página inicial do WordPress.
9.3 Acessar as instâncias EC2 pelo SSM
1.	Vá até o SSM (AWS Systems Manager) no Console de Gerenciamento da AWS.
2.	No menu à esquerda, clique em Instances & Nodes.
3.	Selecione Managed Instances e procure pelas instâncias EC2 que estão sendo gerenciadas pelo SSM.
4.	Clique na instância que deseja acessar e, em seguida, clique em Connect.
5.	Escolha a opção Session Manager para iniciar uma sessão na instância EC2 diretamente pelo console, sem a necessidade de uma chave SSH.
 
9.4 Configurar o WordPress
1.	Ao acessar o WordPress pela primeira vez, será solicitado que você configure uma conta de login.
 
2.	Siga o assistente de instalação do WordPress para configurar o banco de dados, escolher o idioma e definir as credenciais de administrador.
3.	Após a configuração, você poderá acessar o painel administrativo do WordPress para gerenciar sua aplicação.
 
Pronto! A aplicação WordPress está configurada, escalável e funcionando corretamente.
Referências
Documentação AWS
•	EC2
•	RDS
•	EFS
•	ALB
•	Auto Scaling
Outros Recursos
•	Configurar Auto Scaling e Load Balancer
•	DevOps na Infra
•	Como preparar Security Groups para o projeto WordPress
•	Tutorial de instalação e configuração
