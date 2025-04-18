## Objetivo

- Entender os conceitos básicos sobre uma plataforma de gerenciamento de hardware.
- Introduzir conceitos básicos sobre redes de computadores.

## Montagem do Roteiro

### Tarefa 1: Instalar o MaaS
O MaaS um software de código aberto e suportado pela Canonical. O MAAS trata servidores físicos como máquinas virtuais ou instâncias na nuvem. 
Esse software possui diversas versões, mas para esse projeto iremos utilizar a versão stable 3.5.3.

Primeiro atualizamos os pacotes do sistema:

```sudo apt update && sudo apt upgrade -y```
   
Em seguida fazemos a instalação:

```
sudo snap install maas --channel=3.5/stable
sudo snap install maas-test-db
```

### Tarefa 2: Acessando e configurando o MaaS
Ainda conectado a máquina local pelo cabo Ethernet podemos acessar a máquina main através da nossa máquina pessoal utilizando o SSH:

```
ssh cloud@172.16.0.3
```
![](imagens/imagem_2025-03-12_185422999.png)

- Inicializando o MaaS

```
sudo maas init region+rack --maas-url http://172.16.0.3:5240/MAAS --database-uri maas-test-db:///
sudo maas createadmin
```

Nessa parte devemos criar um login e uma senha. Pelo padrão da disciplina iremos criar o admin como "cloud" e a senha "cloud"+"letra do Kit", neste caso "cloudv"


- Gerando o par de chaves para autenticação
  
Em seguida, precisamos gerar as chaves para autenticar através do comando:

```
ssh-keygen -t rsa
```

![](imagens/imagem_2025-03-12_194200944.png)

Gerando a chave, devemos copiar e guardar em uma pasta: 


```
cat ./.ssh/id_rsa.pub
```

- Acessando o dashboard do MaaS

Após esses passos é possível acessar o dashboard do MaaS para dar sequência as outras configurações necessárias através do link [http://172.16.0.3:5240/MAAS](http://172.16.0.3:5240/MAAS)

Agora precisamos configurar o DNS forwarder com o DNS do Insper. Para isso precisamos acessar a aba Settings/Network/DNS do dashboard:

![](imagens/imagem_2025-03-12_201158884.png)

Agora vamos importar as imagens do Ubuntu 22 e 24 no dashboard:

![](imagens/imagem_2025-03-13_194540009.png)

Em seguida devemos definir o Global Kernel Parameters:

![](imagens/imagem_2025-03-13_211926729.png)

### Tarefa 3: Chaveando o DHCP

Nessa etapa precisamos habilitar o DHCP no MaaS Controller, definindo o range de IPs iniciar em 172.16.11.1 e acabar em 172.16.14.255. Além disso, tem que deixar o DNS da subnet apontando para o DNS do Insper.

![](imagens/imagem_2025-03-13_025508252.png)
![](imagens/imagem_2025-03-13_025918994.png)

### Tarefa 4: Checando a saúde do Maas

Dentro do dashboard, na aba de Controladores, é possível checar a "saúde" do sistema Maas. Nessa página deve haver uma marca de seleção verde ao lado dos itens 'regiond' até 'dhcpd':

![](imagens/imagem_2025-03-13_025825254.png)

### Tarefa 5: Comissionando os servidores

Agora é o momento de cadastrar (fazer o host) de todas as máquinas do server1 até o server5 através da aba Machines do dashboard do MaaS. Para isso algumas instruções devem ser seguidas:

- Ao cadastrar as máquinas devemos preencher as opções:
   - Preencher a opção Power Type com Intel AMT
   - Inserir MacAddress da máquina respectiva, esse valor está escrito em cada máquina na parte de baixo
   - Inserir a senha como _CloudComp6s!_
   - IP do AMT = 172.16.15.X (sendo X o id do server, por exemplo server1 = 172.16.15.1)

Em seguida, devemos checar todos os nós e verificar se estão todos com o status Ready, além de verificar também o hardware como memória, SSD etc.

![](imagens/imagem_2025-03-13_220911807.png)

Ainda resta um último dispositivo para cadastrar, neste caso o roteador. Adicionamos o roteador pela aba Devices do dashboard do MaaS:

![](imagens/imagem_2025-03-13_222310675.png)

### Tarefa 6: Criando OVS bridge

Uma Open vSwitch (OVS) bridge reduz a necessidade de duas interfaces de rede físicas. As pontes OVS são criadas na aba NetWork ao configurar um nó (machine). Aqui, vamos criar uma ponte a partir da interface regular 'eth0'. O nome da ponte vai ser referenciado em outras partes e como exigência da disciplina iremos chamar de 'br-ex':

![](imagens/imagem_2025-03-13_222218139.png)

Devemos fazer esse procedimento para cada uma das machines da nossa nuvem.

### Tarefa 7: Fazendo acesso remoto ao kit

Agora vamos realizar um NAT para permitir o acesso da "Rede Wi-fi Insper" do computador pessoal ao servidor MAIN, a ideia é utilizar a porta 22. Além disso, temos que configurar uma porta para acessar o MaaS remotamente, usaremos a porta 5240 e configurar no roteador na aba Transmisson -> NAT:

![](imagens/imagem_2025-03-13_222737099.png)

Também é necessário liberar o acesso ao gerenciamento remoto do roteador criando uma regra de gestão para a rede 0.0.0.0/0, na aba System Tools -> Admin Setup -> Remote Management:

![](imagens/imagem_2025-03-13_222843232.png)

Assim já é possível acessar remotamente sem precisar conectar o cabo Ethernet diretamente no Switch. Neste caso, precisamos utilizar um IP diferente disponível no dashboard do roteador na página principal em WAN IPv4 -> WAN1 -> IP Address


## Bare Metal: Django em nuvem

Com a infra pronta agora vamos fazer deploy de uma aplicação Django. Mas antes, precisamos configurar um ajuste no DNS, dentro da aba Subnets clicar na subnet 172.16.0.0/20 e editar a Subnet summary colocando o DNS do Insper - 172.20.129.131

![](imagens/imagem_2025-03-14_164227931.png)

### Primeira Parte: Banco de Dados

Primeiro vamos acessar o dashboard do MaaS na aba Machines e fazer deploy do ubuntu 24.04 no server1. Acessando o dashboard do server1 e em "Take Action" selecionar a versão do Ubuntu 24 e habilitar o script para inserir a chave ssh que foi obtida anteriormente e pode ser recuperada com o comando: 

```
cat ./.ssh/id_rsa.pub
```

![](imagens/imagem_2025-03-14_203900663.png)

Em seguida acessamos o server1 pelo terminal via SSH. Primeiro acessamos a main:

```
ssh cloud@10.103.1.31
```
> Esse IP é para acesso remoto pela rede do Insper

Depois acessamos o server1:

```
ssh ubuntu@172.16.8.196
```
> Esse IP é o que aparece no dashboard do MaaS na aba Machines e aparece logo abaixo do nome de cada server

Agora dentro do server1 vamos dar continuidade com o bando de dados, para isso executamos os comandos:

```
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

Ainda dentro do server1 precisamos criar um usuário:

```
sudo su - postgres
createuser -s cloud -W
```
> Utilizar a senha cloud

Agora vamos criar o database:

```
createdb -O cloud tasks
```

Agora expor o serviço para acesso e remover o comentário e substituir a string da linha para aceitar conexões remotas:

```
nano /etc/postgresql/16/main/postgresql.conf
```
```
listen_addresses = '*'
```

Depois vamos liberar para qualquer máquina dentro da subnet do kit:

```
nano /etc/postgresql/<versão>/main/pg_hba.conf
```
Alterando a linha:

```
  host    all             all             172.16.0.0/20          trust
```

Em seguida, vamos sair do usuário postgres, liberar o firewall e reiniciar o serviço:

```
sudo ufw allow 5432/tcp
sudo systemctl restart postgresql
```

É importante verificar o status do database e checar se há erros. Para isso executamos os comandos:

```
sudo systemctl status postgresql

telnet localhost 5432

telnet 172.16.0.196 5432

sudo ss -tulnp | grep postgres
```


![](imagens/foto.png)


### Parte 2: Deploy manual do django

Agora vamos retornar ao dashboard do MAAS para fazer o deploy do server2. Acessando o dashboard do server2 e em "Take Action" selecionar a versão do Ubuntu 24 e habilitar o script para inserir a chave ssh que foi obtida anteriormente e pode ser recuperada com o comando: 

```
cat ./.ssh/id_rsa.pub
```

Em seguida, acessar o SSH do server2 e clonar o seguinte repositório:

```
git clone https://github.com/raulikeda/tasks.git
```

Esse repositório tasks tem algumas atividades pré-configuradas que iremos utilizar para preparar todo o nosso ambiente do server2. Esse repositório:

- Atualiza o sistema (apt update e apt upgrade).

- Instala pacotes essenciais como Git, curl, snapd, etc.

- Instala ferramentas como Django, Python, Postgres.

Agora entrando no diretório e entrando na pasta tasks:

```
cd tasks
```

Executamos o comando:

```
./install.sh
```

Com isso todo o ambiente do server2 está pronto e com a aplicação django instalada.

Podemos acessar a aplicação django utilizando um túnel SSH e com esse serviço temporário podemos usar a aplicação fora do kit enquanto o terminal que o tunnel estiver utilizando esteja ativo. Para isso, saímos da main e entramos novamente com o seguinte comando utilizando tunnel:

```
ssh cloud@10.103.0.1 -L 8001:[172.16.8.120]:8080
```

O comando acima irá criar um tunel do serviço do server2 na porta 8080 para o seu localhost na porta 8001 usando a conexão SSH. Agora basta acessar o navegador pelo endereço http://localhost:8001/admin/ usando login e senha "cloud" e ver a aplicação funcionando:

![](imagens/foto2.jpeg)

Agora é importante temos uma visão macro de como estão as máquinas até agora, principalmente seus testes de hardware e comissioning e verificar se está tudo OK antes de prosseguir com a próxima tarefa:

- Todas as máquinas

![](imagens/foto3.jpeg)

- Imagens sincronizadas

![](imagens/foto15.jpeg)

- Server1

![](imagens/foto4.jpeg)
![](imagens/foto5.jpeg)

- Server2

![](imagens/foto6.jpeg)
![](imagens/foto7.jpeg)

- Server3

![](imagens/foto8.png)
![](imagens/foto10.png)

- Server4

![](imagens/foto11.png)
![](imagens/foto12.png)

- Server5

![](imagens/foto13.png)
![](imagens/foto14.png)

### Parte 3: Automatização de deploy

Fizemos uma aplicação django no server 2, mas agora teremos 2 aplicações junto com o server3 compartilhando o mesmo banco de dados do server1. Isso é essencial porque se um node cair o outro está no ar, para que nosso cliente acesse. Além disso, é possível balancear os acessos entre as duas aplicações. Para essa nova instalação será feito de forma automática através do gerenciador de deplou Ansible.

Seu principal diferencial é ser idempotente, ou seja, conseguir repetir todos os procedimentos sem afetar os estados intermediários da instação. E para dar seguimento faremos deploy do server3 no dashboard do MAAS e acessar o SSH da main e executar os comandos:

```
sudo apt install ansible
wget https://raw.githubusercontent.com/raulikeda/tasks/master/tasks-install-playbook.yaml
ansible-playbook tasks-install-playbook.yaml --extra-vars server=[IP server3]
```
> Onde tem IP server3 retire os colchetes e coloque o IP

Mais uma vez é importante verificar os status das aplicações e do dashboard do MAAS:

- Máquinas

![](imagens/foto16.jpeg)

- Aplicação django rodando no server2

![](imagens/foto17.jpeg)

- Aplicação django rodando no server3

![](imagens/foto18.jpeg)

Instalar manualmente o Django envolve baixar e configurar cada componente, como Python, pip, virtualenv e as dependências específicas do projeto, além de configurar o ambiente de desenvolvimento, o que pode ser trabalhoso e propenso a erros. Por outro lado, usar o Ansible para instalar o Django permite automatizar todo o processo, criando playbooks que definem as configurações necessárias, como instalar pacotes, configurar o ambiente virtual, clonar o repositório do projeto e ajustar as configurações do Django, tornando o processo mais rápido, além de facilitar a replicação em diferentes ambientes.

### Parte 4: Balanceamento de carga

Para essa última tarefa utilizaremos uma aplicação de proxy reverso como load balancer. O balanceamento de carga é uma técnica eficaz para distribuir o tráfego de entrada entre vários servidores, garantindo que nenhum deles fique sobrecarregado. Ao dividir o processamento entre várias máquinas, você cria uma rede mais resiliente e estável, capaz de lidar com falhas e manter a aplicação funcionando sem interrupções. O algoritmo Round Robin é uma abordagem simples e eficaz para alcançar isso, direcionando os visitantes para diferentes endereços IP de forma rotativa.

Para prosseguir com a configuração do loadbalancing do nginx temos que instalar o Nginx com o comando:

```
sudo apt-get install nginx
```

Para configurar um loadbalancer round robin, precisaremos usar o módulo upstream do nginx. Incorporaremos a configuração nas definições do nginx. Para isso acessamos através do comando:

```
sudo nano /etc/nginx/sites-available/default
```

Dentro do arquivo vamos alterar a seção com o nome "upstream backend" com a seguinte estrutura:

```
upstream backend {
   server backend1;
   server backend2;
   server backend3;
}
```

No nosso caso, colocamos os IPs de cada máquina:

```
upstream backend {
   server 172.16.0.196;
   server 172.16.8.120;
   server 172.16.8.124;
}
```

Ainda dentro do documento iremos alterar um módulo com o nome de "server" com a seguinte configuração:

```
server {
   location / { proxy_pass http://backend;}
}
```

Salve o arquivo e feche com Ctrl+O e Ctrl+X e em seguida reiniciar o nginx:

```
sudo service nginx restart
```

Para verificar se tudo está corretamente configurado vamos acessar cada máquina e entrar na pasta tasks e no arquivo views.py e alterar a mensagem "Hello World" para identificar cada server:

```
from django.shortcuts import render

from django.http import HttpResponse

def index(request):

  return HttpResponse("Agora estou no server2 da Cloud-V")
```

Agora vamos realizar um request GET para testar se tudo funciona. Para isso vamos fazer um túnel com a seguinte estrutura:

```
ssh cloud@(IPMAIN) -L 8081:(IPSERVER):80
```

No nosso caso para cada server:

```
ssh cloud@10.103.1.31 -L 8081:172.16.0.196:80
```

```
ssh cloud@10.103.1.31 -L 8081:172.16.8.120:80
```

Com isso, basta acessar o navegador pelo link http://localhost:8081/tasks/ e se a mensagem identificadora de cada server aparecer está tudo ok:

- Server2

![](imagens/foto19.jpeg)

- Server3

![](imagens/foto20.jpeg)
