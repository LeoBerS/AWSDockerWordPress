# AWSDockerWordPress
Projeto para implementação de WordPress em um container Docker instalado em um instância EC2 que não deverá utilizar ip público para saída do serviços WP o tráfego com a internet
deve sair pelo LB (Load Balancer). Também será usado RDS com MYSQL e EFS. Também haverá uso de arquivo start instance (user_data.sh), launch template, auto scaling group e load balancer.

## 1.	Crie a VPC e Sub-redes: Configure sua VPC e sub-redes (públicas e privadas) usando a GUI do AWS.
a.	Digitar VPC no campo de pesquisa (lupa): clicar no resultado VPC para ser direcionado para o console VPC;
b.	clicar no botão “Criar VPC”:
i.	Em “Configurações da VPC” clicar em “VPC e muito mais”;
ii.	Em “Geração automática de etiqueta de nome marcar “Gerar Automaticamente” e dar um nome. Ex.: PB – JUL 2024;
iii.	Bloco CIDR IPv4: 10.0.0.0/16;
iv.	Número de zonas de disponibilidade: 2;
v.	Número de sub-redes públicas: 2;
vi.	Número de sub-redes privadas: 2;
vii.	Gateways NAT: 1 por AZ;
viii.	Endpoints da VPC: Gateway do S3
ix.	Opções de DNS: Habilitar nomes de host DNS; Habilitar resolução de DNS;
x.	Tags adicionais: 
1.	Chave: CostCenter; Valor: C092000024;
2.	Chave: Project; Valor: PB – JUL 2024;
3.	Chave: Atividade; Valor: Docker – WP;
xi.	Clicar no botão “Criar VPC”.

## 2.	Criar a instância RDS: 
a.	Inicie a criação de uma instância RDS:
i.	No menu lateral, clique em RDS e depois em Databases (Bancos de Dados).
ii.	Clique em Create database (Criar banco de dados).
b.	Escolha o tipo de banco de dados:
i.	Selecione o motor de banco de dados que você deseja usar (por exemplo, MySQL, PostgreSQL, etc.).
c.	Configurações: 
i.	Identificador da instância de banco de dados: WP-database-1;
ii.	Configuração de credenciais:
1.	Nome do usuário principal: admin
2.	Gerenciamento de credenciais: autogerenciada;
3.	Senha principal: SenhaRDS_WP
d.	Configuração da instância:
i.	Classes com capacidade de intermitência (inclui classes t): selecionar db.t3.micro
e.	Configuração do banco de dados:
i.	Instance class: Escolha o tipo de instância (por exemplo, selecionar db.t3.micro para o free tier).
f.	Configuração de armazenamento:
i.	Tipo de armazenamento: SSd de uso geral (gp3)
ii.	Allocated storage: Escolha o tamanho do armazenamento (por exemplo, 20 GB para o free tier).
g.	Configuração de rede / Conectividade:
i.	Recurso de computação: Não se conectar a um recurso de computação EC2;
ii.	VPC: Selecione a VPC onde sua instância EC2 está localizada.
iii.	Subnet group: Escolha o grupo de sub-redes que corresponde à sub-rede privada da sua instância EC2 ou, caso não exista, deixar selecionado: Criar novo grupo de sub-redes do banco de dados: wp-subrede-rds;
iv.	Acesso público: Não;
v.	Grupo de segurança de VPC (firewall): Criar novo;
vi.	Novo nome do grupo de segurança da VPC: dockerWP
vii.	Zona de disponibilidade: Sem preferência;
viii.	Configuração adicional: Porta do banco de dados: 3306 (MySQL).
h.	Configuração de segurança:
i.	Security group: Crie um novo grupo de segurança ou selecione um existente.
ii.	Add rules: Adicione uma regra para permitir o tráfego de entrada na porta 3306 (para MySQL) da sua instância EC2.
i.	Configurações adicional:
i.	Nome do banco de dados inicial: DockerWP;
ii.	Backup: Configure as opções de backup conforme necessário.
iii.	Maintenance: Configure as opções de manutenção conforme necessário.
j.	Finalização:
i.	Clique em Create database para criar a instância RDS.
k.	Após a criação, você obterá o endpoint do RDS, que você pode usar para configurar a conexão no seu aplicativo WordPress na instância EC2

## 3.	Criar o script start instance (user_data.sh):
a.	Para criar o arquivo user_data.sh foi usado o WSL Ubuntu no Windows PowerShell;
b.	Usar editor VIM: vim user_data.sh;
c.	Esse arquivo será usado no Launch Template (Modelos de Execução) do painel EC2;
  #!/bin/bash
# Atualizar e instalar o Docker
sudo yum update -y
sudo yum install -y yum-utils
sudo yum install -y docker

#Iniciar Docker
#sudo systemctl start docker
sudo service docker start

#Habilite o Docker para Iniciar na Inicialização
sudo systemctl enable docker

#O amazon-efs-utils precisa estar instalado para o CloudShell reconhecer o efs como um tipo de sistema de arquivos
sudo yum install -y amazon-efs-utils

#EFS seja montado automaticamente em cada instância nova: Adicione Montagem Persistente ao /etc/fstab
sudo yum update -y
sudo yum install -y nfs-utils
sudo mkdir -p /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1 fs-0f3f994e1db0bf26f.efs.us-east-1.amazonaws.com:/ /mnt/efs
echo "fs-0f3f994e1db0bf26f.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs4 defaults,_netdev 0 0" | sudo tee -a /etc/fstab

#Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Criar link simbólico para o Docker Compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

#Criar Arquivo Docker Compose para WordPress e MySQL
sudo tee /mnt/efs/docker-compose.yml > /dev/null <<EOF
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: wp-database-1.c742yg8w8ch8.us-east-1.rds.amazonaws.com
      WORDPRESS_DB_USER: admin
      WORDPRESS_DB_PASSWORD: SenhaRDS_WP
      WORDPRESS_DB_NAME: dockerWPdb
    volumes:
      - /mnt/efs:/var/www/html

  db:
    image: mysql:latest
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: admin
      MYSQL_PASSWORD: SenhaRDS_WP
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
EOF

#Navegar para o diretório e Iniciar Docker Compose
cd /mnt/efs
sudo docker-compose up -d

## 4.	Criar o Launch Template (Modelos de Execução) no painel EC2
a.	Clicar em Modelos de Execução;
b.	Nome do modelo de execução: dockerWPUsEast1a;
c.	Descrição da versão do modelo: Implementação de docker e WP em instâncias EC2, em subredes privadas.;
d.	Orientação sobre o Auto Scaling: clicar em Fornecer orientação para me ajudar a configurar um modelo que eu possa usar com o Auto Scaling do EC2;
e.	Conteúdo do modelo de execução: Imagens de aplicação e de sistema operacional (imagem de máquina da Amazon) – obrigatório: Amazon Linux 2 AMI 
f.	Tipo de instância: t3.micro;
g.	Par de chaves (login): Chave_PB;
h.	Configurações de rede:
i.	Sub-rede: Não incluir no grupo de execução; (ficará a cargo do Auto Scaling Group selecionar)
ii.	Firewall (grupos de segurança): Criar grupo de segurança: 
1.	Nome do grupo de segurança: launchTemplate_sg
2.	Regras do grupo de segurança de entrada:

## 5.	Criar o Load Balancer no painel EC2:
a.	Clicar no botão Criar load balancer;
b.	Em Tipos de Load Balancer, na seção Application Load Balancer, clicar em Criar;
c.	Em Configuração Básica preencher / alterar: Nome;
d.	Em Mapeamento de rede selecionar a VPC que acabamos de criar e selecionar suas duas zonas de disponibilidade e suas subredes públicas;
e.	Em Grupos de segurança, crie um grupo de segurança;
  i.	Regras de Entrada para o Load Balancer:
    1.	Permitir Tráfego HTTP:
      a.	Type: HTTP
      b.	Protocol: TCP
      c.	Port Range: 80
      d.	Source: 0.0.0.0/0 (todos os IPs)
    2.	Permitir Tráfego HTTPS (se estiver usando SSL):
      a.	Type: HTTPS
      b.	Protocol: TCP
      c.	Port Range: 443
      d.	Source: 0.0.0.0/0 (todos os IPs)
  ii.	Regras de Saída para o Load Balancer:
    1.	Permitir Tráfego para as Instâncias EC2:
      a.	Type: Custom TCP Rule
      b.	Protocol: TCP
      c.	Port Range: 80 (ou a porta configurada para o WordPress)
      d.	Destination: Subredes privadas (ex.: 10.0.0.0/16)
    2.	Permitir Conexões Internas:
      a.	Type: All traffic
      b.	Protocol: All
      c.	Port Range: All
      d.	Destination: Security Group ID das instâncias EC2 privadas
f.	Em Listeners e roteamento, protocolo HTTP 80 e clicar em Criar grupos de destino;
  i.	Adicionar Listeners:
    1.	Porta 80 (HTTP):
      a.	Protocol: HTTP
      b.	Port: 80
      c.	Default Action: crie ou escolha o Target Group com suas instâncias privadas;
    2.	Porta 443 (HTTPS) (se estiver usando SSL):
      a.	Protocol: HTTPS
      b.	Port: 443
      c.	Default Action: mesmo Target Group;
    3.	Configuração do Roteamento:
      a.	Target Group: Escolha o Target Group que contém suas instâncias EC2 privadas.
      b.	Health Checks: Configure verificações de integridade para garantir que apenas instâncias saudáveis recebam tráfego.
      c.	Protocol: HTTP
      d.	Path: / ou /health-check dependendo da sua configuração.

## 6.	Criar o Auto Scaling Group no painel EC2:
a.	Clicar em Auto Scaling Group;
b.	Clicar no botão: Criar grupo de auto scaling;
c.	Nome do grupo do Auto Scaling: dockerWP
d.	Modelo de execução: dockerWPUsEast1a; Versão: Default (1);

## 7.	Para verificar aplicação wordpress funcionando (tela de login):
a. No painel EC2, em Load Balancers, copiar o Nome do DNS;
b. Colar o Nome do DNS no navegador;
c. Deverá aparecer a tela do WordPress.

