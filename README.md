# Atividade Docker PB -JAN 2025 | DevSecOps

![logos](https://github.com/user-attachments/assets/5f9c51b8-4329-464c-8f30-dbeb6f61b87c)

## Projeto Prático Docker-AWS

### Etapas

1. Configuração de Ambiente
2. Recursos de Implementação
3. Load Balancer
4. Auto Scaling Group
5. Monitoramento


### Tecnologias Utilizadas

- AWS
- Linux
- Docker
- Wordpress

## Etapa 1 - Configuração de Ambiente

Nesta etapa será configurado uma estutura de rede com VPC (Virtual Private Cloud) junto com seus Security Groups e Subnets, e um Banco de dados RDS.

### 1.1 Configurando a Rede VPC

Uma VPC é um ambiente de rede isolado e seguro na AWS, onde onde é possível executar recursos como instâncias EC2 (Elastic Compute Cloud), bancos de dados e outros serviços. Esta etapa cria uma VPC com sub-redes públicas e privadas para organizar e proteger os recursos do projeto.

No console principal da AWS, utilize a barra de busca localizada no topo para buscar pelo serviço de VPC.
Clique em ***Ciar VPC / Create VPC*** para iniciar o processo de criação.

- **VPC e muito mais / VPC and more**: Esta opção cria a VPC juntamente com duas Subnets públicas e duas privadas.
- Defina um nome para a VPC.
- **IPv4 CIDR block**: Defina o intervalo de endereços de IP como `10.0.0.0/24`. Este bloco permite 256 endereços de IP, o que é mais do que suficiente para os fins deste projeto.
- Clique em ***Ciar VPC*** para finalizar a criação.

#### Grupos de Segurança / Security Groups

As **regras de segurança** definem como o tráfego de rede pode acessar os recursos dentro da VPC. Nesta etapa, configuramos quatro Grupos de Segurança e suas regras de entrada (inbound) para permitir acesso HTTP na porta 80, SSH apenas a partir do seu IP e comunicação entre os dois Grupos de Segurança.

### Vamos criar 4 Security Groups:

1. Instâncias: webserver-sg
2. Banco de dados RDS: rds-sg
3. Sistema de arquivos EFS: efs-sg
4. Classic Load Balancer: clb-sg

- CLique em ***Criar grupo de segurança***.
- Dê um nome e descrição que são obrigatórios.
- Selecione a VPC criada anteriormente.
- Regras de entrada e saída serão configuradas a seguir, após criar os outros grupos
- Finalize clicando em ***Criar grupo de segurança***.

Configure as regras da seguinte maneira:

📌**webserver-sg**
|Entrada | | |
|--------|--------|--------|
| Tipo: HTTP | Porta 80 | clb-sg |

|Saída | | |
|--------|--------|--------|
| Tipo: MySQL/Aurora | Porta 3306 | Destino: rds-sg |
| Tipo: NFS  | Porta 2049 | Destino: efs-sg |
| Tipo: Todo o tráfego | Tudo | 0.0.0.0/0 |
| Tipo: HTTP | Porta 80 | clb-sg |


📌**rds-sg**
|Entrada | | |
|--------|--------|--------|
| Tipo: MySQL/Aurora | Porta 3306 | Destino: webserver-sg |

|Saída | | |
|--------|--------|--------|
| Tipo: MySQL/Aurora | Porta 3306 | Destino: webserver-sg |


📌**efs-sg**
|Entrada | | |
|--------|--------|--------|
| Tipo: NFS  | Porta 2049 | Destino: webserver-sg |

|Saída | | |
|--------|--------|--------|
| Tipo: Todo o tráfego | Tudo | 0.0.0.0/0 |


📌**clb-sg**
|Entrada | | |
|--------|--------|--------|
| Tipo: HTTP | Porta 80 | 0.0.0.0/0 |

|Saída | | |
|--------|--------|--------|
| Tipo: HTTP | Porta 80 | webserver-sg |


Após salva-las, os grupos estão devidamente configurados para comunicação necessária para o projeto.


## Etapa 2 - Recursos de Implementação

Utilziaremos recursos de Banco de Dados RDS, Elastic File System (EFS) e Modelos de Execução de Instância (Templates).

### 2.1 Criar Banco de Dados Amazon RDS (Relational Database Service)

Segundo a [documentação oficial](https://aws.amazon.com/pt/rds/):
> O Amazon Relational Database Service (Amazon RDS) é um serviço de banco de dados relacional de fácil gerenciamento e otimizado para o custo total de propriedade. (...) O Amazon RDS automatiza as tarefas genéricas de gerenciamento de banco de dados, como provisionamento, configuração, backups e aplicação de patches.

No console principal da AWS, pesquise por RDS e clique em **Aurora and RDS**.

Clique em **Banco de dados** no painel à esquerda, em seguida clique em **Criar banco de dados**.

Para esse projeto será feita a seguinte configuração durante a criação do Banco de Dados:

![Captura de tela 2025-04-02 142627](https://github.com/user-attachments/assets/5f14a303-b939-4660-afb9-87835d32964e)

- **Método**: Criação Padrão
- **Mecanismo**: MySQL
- **Versão do Mecanismo**: sempre a última
- **Modelo**: Gratuito/Free tier
- **Disponibilidade**: Single-AZ (1 instância)

![Captura de tela 2025-04-02 142646](https://github.com/user-attachments/assets/9d3df6e6-4d28-4aea-b338-2b9936510311)

### Configurações

- Defina um **Identificador** para o banco de dados.
- ⚠️ Defina um **Nome de usuário** e **Senha** que serão utilizados nas próximas etapas como Docker Compose.

![Captura de tela 2025-04-02 142729](https://github.com/user-attachments/assets/fdc82701-0184-45c9-b0c6-31bbecfe2731)

### Configurações da instância e Armazenamento

- Altere a classe para `db.t3.micro`
- Em **Configuração adicional de armazenamento** reduza o limite para o mnínimo permitido (22GB).

![Captura de tela 2025-04-02 143230](https://github.com/user-attachments/assets/2b5c3efd-f6bb-47f6-9682-e14c4037e25a)


### Conectividade

Nesta etapa conferimos apenas **VPC** selecionada é a que criamos anteriormente e o **Grupo de Segurança** `rds-sg`.

![Captura de tela 2025-04-02 143710](https://github.com/user-attachments/assets/5f5d3f13-5972-494d-88b8-6524d739c86d)

Finalize clicando em **Criar Banco de Dados**. ✅
O processo demora de 5 a 10 minutos para disponibilidade do banco.

### 2.2 Criar Sistema de Arquivos EFS (Elasitc File System)

[Documentação oficial Amazon EFS](https://docs.aws.amazon.com/pt_br/efs/)
> O Amazon Elastic File System (Amazon EFS) fornece um sistema de arquivos elástico, escalável, simples e totalmente gerenciado para uso com os Serviços de nuvem AWS e recursos locais.

No painel principal da AWS busque por **EFS** e escolha, clique e em **Criar sistema de arquivos**, dê um nome e mantenha a VPC que criamos anteriormente. 

![Captura de tela 2025-04-02 144314](https://github.com/user-attachments/assets/6564ff16-150f-4f57-b2dc-d666c72a6265)

Clique em **Personalizar**:
- **DESABILITE** os backups automáticos.
- Altere a **Transcrição para Archive** para **NENHUM**.

Clique em **Próximo**.

![Captura de tela 2025-04-03 194937](https://github.com/user-attachments/assets/8d2a0ab5-71a1-4766-8a98-dd7c7aec5b00)


- Selecione a **VPC** que criamos.
- Altere as duas sub-redes para as redes **Privadas**.
- Selecione ambos os Grupos de Segurança como `efs-sg`.

![Captura de tela 2025-04-03 195214](https://github.com/user-attachments/assets/b27c3cc4-2319-4d3a-a69c-1f191460f5cc)


Clique em **Próximo** em todas as etapas a seguir até disponibilizar **Criar**, e clique.

Após criado, no console principal do EFS, clique no que criamos e na tela seguinte clique em **Anexar**.
![Captura de tela 2025-04-03 195503](https://github.com/user-attachments/assets/44b25286-a13e-4e77-ad82-9ab7fa2ef886)

Copie o **assistente de montagem do EFS**:

![Captura de tela 2025-04-03 200253](https://github.com/user-attachments/assets/b4da7257-c004-420b-a990-cddea859c569)

Guarde esse comando para ser usado no User Data posteriormente.

**EFS criado!** ✅

### 2.3 Instâncias EC2

As instãncias serão configuradas para que o Auto Scaling Group as inicie e controle sua execução. Para isso precisamos criar um modelo de Instância que será usado nesse processo utilziando script de iniciação via User Data.

> Um **UserData** é um conjunto de comandos que pode ser fornecido ao iniciar uma instância, executado automaticamente **apenas uma vez**, geralmente utilizado para instalar softwares, configurar serviços e executar tarefas de inicialização personalizadas.


1. No console principal da AWS, busque por **"EC2"**.
2. No painel esquerdo clique em **Modelos de Execução** > **Criar novo modelo de execução**.
3. Defina um nome para esse template `wordpress-template`, por exemplo, adicione uma descrição obrigatória.
4. **Imagens de aplicação**

### Configurações

Este projeto será feito baseado nas configurações a seguir.

- **Imagem de máquina da Amazon (AMI - Amazon Machine Image)**: Selecione **Ubuntu Server 24.04 LTS (HMV), SSD Volume Type**.
- **Tipo de Instância**: Escolha **t2.micro**.

- **Par de Chaves (Key Pair)**: Utilize as configurações padrões de criar uma nova Key Pair. Defina um nome (exemplo.pem) para a chave e faça o download após a criação.

- **VPC**: Selecione a VPC criada anteriormente.
- **Sub-rede**: não selecione nenhuma!
- **Firewall (grupos de segurança)**: Selecione o **sg_instance**, que configuramos especificamente para as instâncias.

- **Configurar armazenamento**: Altere para **gp3**.

- **Configurações avançadas (Advanced details)**: o seguinte User-Data será utilizado na implementação do projeto.

⚠️ Atenção para colar o comando EFS copiado anteriormente `sudo mount -t efs -o tls fs-XXXXXXXX:/ /wordpress` em [COMANDO_DE_MONTAGEM_EFS]


```
#!/bin/bash

# Atualiza os pacotes do Linux e instala Docker, wget e amazon-efs-utils (necessário para montar EFS)
sudo yum update -y
sudo yum install -y docker wget amazon-efs-utils

# Inicia e habilita para iniciar automaticamente com boot o serviço Docker
sudo service docker start
sudo systemctl enable docker.service

# Adiciona o usuário padrão "ec2-user" ao grupo do Docker para permitir execução sem sudo
sudo usermod -aG docker ec2-user

# Baixa a versão mais recente do Docker Compose diretamente do repositório oficial
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Concede permissão de execução ao Docker Compose
sudo chmod +x /usr/local/bin/docker-compose

# Cria o diretório onde o EFS será montado
sudo mkdir -p /wordpress

# Comando para montar o EFS (sudo mount -t efs -o tls fs-XXXXXXXX:/ /wordpress)
sudo [COMANDO_DE_MONTAGEM_EFS]

# Verifica se o EFS foi montado corretamente e exibe uma mensagem correspondente
if mountpoint -q /wordpress; then
    echo "EFS montado com sucesso em /wordpress"
else
    echo "Falha ao montar EFS"
fi

# Baixa o arquivo docker-compose.yml de um repositório remoto (coloque o link de seu repositorio)
wget -O /home/ec2-user/docker-compose.yml [HTTP_RAW_REPOSITORIO_DOCKER_COMPOSE_GITHUB]

# Ajusta as permissões para que o arquivo pertença ao usuário ec2-user
sudo chown ec2-user:ec2-user /home/ec2-user/docker-compose.yml

# Acessa o diretório do usuário ec2-user
cd /home/ec2-user

# Inicia os containers em segundo plano com Docker Compose
sudo docker-compose up -d

```


![Captura de tela 2025-04-02 094356](https://github.com/user-attachments/assets/a23cc14c-0324-48c4-8418-3d44da64f95b)

Clicar em **"Criar modelo de execuçao"**. ✅


## Etapa 3 - Balanceamento de Carga (Load Balancer)

Conforme a [documentação oficial](https://docs.aws.amazon.com/pt_br/elasticloadbalancing/latest/classic/introduction.html):
> Seu load balancer serve como ponto único de contato para os clientes. Isso aumenta a disponibilidade do seu aplicativo. Você pode adicionar e remover instâncias do load balancer do conforme mudarem suas necessidades, sem perturbar o fluxo geral de solicitações para sua aplicação.

No console das Instâncias EC2, procure no painel clique em **Balanceamento de Carga / Load Balancer** e depois em **Criar Load Balancer**.

Usaremos o **Classic Load Balancer**, distribuindo o tráfego de entrada do Wordpress nos dois destinos de instância do EC2 e suas zonas de disponibilidade. Isso aumenta a tolerância a falhas da aplicação.

![Captura de tela 2025-04-03 092916](https://github.com/user-attachments/assets/3726eb62-1bb5-4010-9fcd-ee0b4591b453)

Nas **Configurações básicas** escolha um nome (wordpress-lb, por exemplo), mantenha selecionado a opção **"Voltado para a Internet"**.

**Mapeamento de Rede**: 
- Selecione a VPC criada anteriormente,
- Selecione as duas sub-redes **públicas** onde estão as Instâncias,
- Selecione o Grupo de segurança criado anteriormente para o Load Balancer `clb-sg`.

**Verificação de Integridade**
- Altere o caminho de ping para: `wp-admin/install.php` ⚠️ Importante!

Clique em **Criar Load Balancer**. ✅

## Etapa 4 - Auto Scaling Group

Conforme a [documentação oficial](https://docs.aws.amazon.com/pt_br/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
> O Amazon EC2 Auto Scaling ajuda você a garantir que você tenha o número correto de EC2 instâncias da Amazon disponíveis para lidar com a carga do seu aplicativo. Você cria coleções de EC2 instâncias, chamadas de grupos de Auto Scaling. Você pode especificar o número mínimo de instâncias em cada grupo de Auto Scaling, e o Amazon Auto EC2 Scaling garante que seu grupo nunca fique abaixo desse tamanho. Você pode especificar o número máximo de instâncias em cada grupo de Auto Scaling, e o Amazon Auto EC2 Scaling garante que seu grupo nunca ultrapasse esse tamanho

No console EC2, no painel direito procure e clique em **Auto Scaling / Grupos de Auto Scaling**, depois em **Criar grupo de Auto Scaling**.

Dê um nome para seu ASG e selecione o **Modelo de execuçao / Template** que criamos anteriormente no passo 2.3, altere a versão para `latest` para que sempre utilize a ultima independente da alteração que você fizer nesse template, e clique em **próximo**.

![Captura de tela 2025-04-03 202859](https://github.com/user-attachments/assets/f902f964-3afb-478a-8da3-79f53110b843)

Selecione a VPC que criamos e as duas Sub-redes privadas, clique em **Próximo**.

![Captura de tela 2025-04-03 202931](https://github.com/user-attachments/assets/bd197c45-cabd-4c5b-add3-73cb9d58d588)

Aqui altere conforme a imagem a seguir, coloque o **Load Balancer** que criamos e abaixo marque **Ative as verificações de integridade do Elastic Load Balancing**, e clique em **Próximo**.

![Captura de tela 2025-04-03 203129](https://github.com/user-attachments/assets/460ca654-e3b3-4f35-bdbc-223867f35f74)
![Captura de tela 2025-04-03 203246](https://github.com/user-attachments/assets/eacfd407-4f0e-48fd-b635-162aa87a46fa)

Nesta tela só iremos alterar **Capacidade desejada** para `2` e a **Escalabilidade** para mínimo `2` e máximo `4`. Isso sginifica que o mínimo de instâncias em execução será 2 e o máximo será 4, dependendo da carga de acesso.

![Captura de tela 2025-04-03 203325](https://github.com/user-attachments/assets/f71b2540-acce-439c-912f-77a7fe1411d0)

Habilite as métricas de grupo no CloudWatch e clique em **Próximo** nas telas seguintes até **Criar o ASG**.

![Captura de tela 2025-04-03 203502](https://github.com/user-attachments/assets/204a73e1-5f12-43d7-a2dd-3c9a5b7d5bb9)

**Auto Scaling Group Criado!** ✅

Navegando até o console de EC2 e Instâncias em execução, aguarde as duas instâncias ficarem disponíveis para testar o acesso ao servidor.

Para acessar, navegue até o Load Balancer, copie o **Nome do DNS** e cole no navegador. Se as instâncias já estiverem em execução será mostrado a tela inicial de configuração do Wordpress.

![Captura de tela 2025-04-03 204138](https://github.com/user-attachments/assets/b8d9f243-a1fb-4cdc-8504-53ba0cf12a64)


## Etapa 5 - Monitoramento

Nessa etapa usaremos monitoramento via Cloud Watch.

A principio criaremos as políticas.

No console EC2, vá até o Auto Scalig Group e clique no que criamos. 

Selecione a aba **Políticas de Escalabilidade Dinâmica** e clique em **Criar política de escalabilidade dinâmica**.

![Captura de tela 2025-04-03 204955](https://github.com/user-attachments/assets/5d91b053-bff8-44fe-a2a3-91d96965a073)

Preencha conforme a imagem a seguir e clique em **Criar**.

![Captura de tela 2025-04-03 211317](https://github.com/user-attachments/assets/d734c782-b635-4525-8249-e0b7ce8d603e)

Pesquise por ClourWatch no consle da AWS e vá para a página. 

- Clique em **Alarmes** > **Em alarme** > **Criar alarme**.
- Clique em **Selecionar Métrica** > **EC2** > **By Auto Scaling Group** e selecione `CPUUtilization`, clique em **Selecionar métrica**.

![Captura de tela 2025-04-03 211840](https://github.com/user-attachments/assets/de7aeaac-5575-45a1-b7d5-2b45e169d2a1)

Selecione **Maior/Igual** e defina `80`, clique em **Próximo**.

![Captura de tela 2025-04-03 212003](https://github.com/user-attachments/assets/8835916c-df38-4768-8d7c-cc08cede66d3)

Remova as notificações e adicione **Ação do Auto Scaling** selecionando o ASG que criamos. Clique em **Próximo**.

![Captura de tela 2025-04-03 212346](https://github.com/user-attachments/assets/9d77d366-cd18-4273-876e-ca88b574d4c1)

Nomeie o alarme e clique em **Próximo** e em seguida **Criar Alarme**.

![Captura de tela 2025-04-03 212442](https://github.com/user-attachments/assets/afc1b018-6131-4e97-b71b-b83026dcf3ae)

![Captura de tela 2025-04-03 212644](https://github.com/user-attachments/assets/9dad458a-fba0-4819-8b6b-31260b861cb8)

## Conclusão

Este projeto demonstrou a implementação de um ambiente altamente disponível e escalável para hospedar o WordPress na AWS, utilizando Docker e diversas ferramentas da AWS para garantir desempenho, segurança e resiliência. Ao longo das etapas, configuramos a infraestrutura de rede com VPC, sub-redes e Security Groups, implementamos um Banco de Dados RDS e um sistema de arquivos compartilhado via EFS, e configuramos um Load Balancer junto com um Auto Scaling Group para gerenciamento dinâmico das instâncias EC2.

Os principais aprendizados desta atividade incluem:
- Gerenciamento de infraestrutura na AWS: Como configurar recursos essenciais para uma aplicação web em produção.
- Utilização de Docker: Implementação de um ambiente conteinerizado para garantir portabilidade e facilidade na implantação.
- Automação com User Data: Configuração automatizada das instâncias EC2 para garantir a inicialização correta do ambiente de aplicação.
- Escalabilidade e alta disponibilidade: Utilização do Auto Scaling Group e Load Balancer para distribuir o tráfego e ajustar a capacidade conforme a demanda.

