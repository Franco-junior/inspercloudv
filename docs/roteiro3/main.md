# Objetivos

- Entender os conceitos básicos de Private Cloud.

- Aprofundar conceitos sobre redes virtuais SDN.

# Infra

Este projeto visa construir um ambiente de nuvem privada utilizando OpenStack sobre uma infraestrutura gerenciada por MAAS e Juju, e então implantar uma aplicação específica nesse ambiente virtualizado.

Confira se os seus recursos fisicos seguem, no MÍNIMO, a tabela abaixo, volte ao dashboard do MAAS e crie as Tags conforme descrito:

![](img/req.png)

Antes de começar a instalação do Openstack, verifique se o MAAS está configurado corretamente (Brigdes, Subnets, Tags, etc).

Verifique se o bridge br-ex está configurado corretamente no MAAS. O br-ex é crucial para a comunicação do OpenStack com a rede externa. Ele deve estar configurado para TODOS os nós.

## Implementando o OpenStack

É importante não instalar nada no server2 que deve estar reservado, altere os comandos que forem necessários para utilizar o Node server 1 como controller, o node server 2 como Reserva e os nodes server 3,4 e 5 como compute (onde o Openstack será instalado).
Verifique o status (juju status) para ver se a implantação está correndo como o esperado.
Aguarde a instalação terminar, só vá para o próximo passo quando tiver certeza que o comando anterior foi finalizado
Esse roteiro é baseado na documentação oficial do Openstack, porém adaptado para o nosso ambiente. Logo, atente para o número de máquinas que você tem disponível e para a configuração de rede que você fez no MAAS.

Para monitorar o status da instalação do Openstack, você pode usar o comando abaixo:

```
watch -n 2 --color "juju status --color"
```

Em seguida, vamos começar a configurar o openstack pelo terminal da main:

```
juju add-model --config default-series=jammy openstack
```

```
juju switch maas-controller:openstack
```

### Ceph OSD

O aplicativo **ceph-osd** será implantado em três nós com o charm **ceph-osd**.

Os nomes dos dispositivos de bloco que sustentam os OSDs dependem do hardware nos nós do MAAS. Todos os dispositivos possíveis (em todos os nós) que serão usados para armazenamento do Ceph devem ser incluídos no valor da opção **osd-devices** (separados por espaço). Aqui, usaremos os mesmos dispositivos em cada nó: `/dev/sda` e `/dev/sdb`. O arquivo **ceph-osd.yaml** contém a configuração:

**ceph-osd.yaml**

```yaml
ceph-osd:
  osd-devices: /dev/sda /dev/sdb
```

Para implantar o aplicativo, usaremos a tag **compute** que foi atribuída a cada um desses nós na página *Instalar MAAS*. O comando para implantar o aplicativo **ceph-osd** é:

```bash
juju deploy -n 3 --channel quincy/stable --config ceph-osd.yaml --constraints tags=compute ceph-osd
```

**Nota**: A opção `-n 3` especifica que três unidades do aplicativo **ceph-osd** devem ser implantadas.

Se uma mensagem de uma unidade ceph-osd como **“Non-pristine devices detected”** aparecer na saída do comando `juju status`, será necessário usar as ações **zap-disk** e **add-disk** que acompanham o charm **ceph-osd**. A ação **zap-disk** é de natureza destrutiva. Use-a apenas se quiser apagar completamente o disco de todos os dados e assinaturas para uso pelo Ceph.

### Nova Compute

**Nova** é o projeto do OpenStack que fornece uma forma de provisionar instâncias de computação (também conhecidas como servidores virtuais). O charm **nova-compute** é implantado nos nós de computação com o charm **nova-compute**. O arquivo **nova-compute.yaml** contém a configuração:

**nova-compute.yaml**

```yaml
nova-compute:
  config-flags: default_ephemeral_format=ext4
  enable-live-migration: true
  enable-resize: true
  migration-auth-type: ssh
  virt-type: qemu
```

Os nós devem ser direcionados pelo ID da máquina, já que não há mais máquinas Juju (nós MAAS) livres disponíveis. Isso significa que estaremos colocando vários serviços em nossos nós. Escolhemos as máquinas **0, 1 e 2**. Para implantar:

```bash
juju deploy -n 3 --to 0,1,2 --channel yoga/stable --config nova-compute.yaml nova-compute
```

### MySQL InnoDB Cluster

O **MySQL InnoDB Cluster** sempre requer **pelo menos três unidades de banco de dados**. O aplicativo **mysql-innodb-cluster** é implantado em três nós com o charm **mysql-innodb-cluster**. Eles serão **containerizados** nas máquinas **0, 1 e 2**. Para implantar:

```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel 8.0/stable mysql-innodb-cluster
```

### Vault

O **Vault** é necessário para gerenciar os certificados TLS que permitirão a comunicação criptografada entre os aplicativos da nuvem. O aplicativo **vault** será **containerizado** na máquina **2** com o charm **vault**. Para implantar:

```bash
juju deploy --to lxd:2 vault --channel 1.8/stable
```

Este é o **primeiro aplicativo** a ser conectado ao banco de dados da nuvem que foi configurado na seção anterior. O processo é:

1. Criar uma instância específica para o aplicativo do **mysql-router** com o charm subordinado **mysql-router**;
2. Adicionar uma relação entre a instância do **mysql-router** e o banco de dados;
3. Adicionar uma relação entre a instância do **mysql-router** e o aplicativo.

A combinação dos passos **2 e 3** conecta o aplicativo ao banco de dados da nuvem.

Aqui estão os comandos correspondentes para o Vault:

```bash
juju deploy --channel 8.0/stable mysql-router vault-mysql-router
juju integrate vault-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate vault-mysql-router:shared-db vault:shared-db
```

#### Desbloqueio (Unseal)

Agora o Vault deve ser **inicializado e desbloqueado**. O charm **vault** também precisará ser autorizado para realizar certas tarefas. Esses passos estão descritos na documentação do charm Vault. 

Instalando o cli do Vault e configurando-o:

```
sudo snap install vault
export VAULT_ADDR="http://<IP of vault unit>:8200"
```

Gerando:

```
$ vault operator init -key-shares=5 -key-threshold=3
```

Assim iremos receber 5 Unseal Keys e 1 Initial Root Token. Copie e guarde as hashs geradas.
Removendo o selo, repita a operação com 3 keys diferentes:

```
vault operator unseal <Unseal Key> # com 3 keys diferentes
```
Autorizando o charm (esse passo precisa ser feito em 50 minutos):

```
export VAULT_TOKEN=<Initial Root Token>
vault token create -ttl=50m
```

Anote o token gerado pelo comando e em seguida digite o comando:

```
juju run vault/leader authorize-charm token=token
```


####  Certificado da CA (Autossinado)

Para fornecer ao Vault um certificado de autoridade (CA), é necessário **gerar um certificado CA autossinado**, para que ele possa emitir certificados para os serviços da API da nuvem. Isso está descrito na página *Gerenciando certificados TLS*. **Execute isso agora**:

```bash
juju run vault/leader generate-root-ca
```

Os aplicativos da nuvem são habilitados para TLS por meio da relação **vault\:certificates**. Abaixo, começamos com o banco de dados da nuvem. Embora ele já possua um certificado autossinado, é recomendado usar um **certificado assinado pela CA do Vault**:

```bash
juju integrate mysql-innodb-cluster:certificates vault:certificates
```

### Neutron

A rede **Neutron** é implementada com quatro aplicativos:

* **neutron-api**
* **neutron-api-plugin-ovn** (subordinado)
* **ovn-central**
* **ovn-chassis** (subordinado)

O arquivo **neutron.yaml** contém as configurações necessárias (apenas dois dos aplicativos requerem configuração):

**neutron.yaml**

```yaml
ovn-chassis:
  bridge-interface-mappings: br-ex:eth0
  ovn-bridge-mappings: physnet1:br-ex

neutron-api:
  neutron-security-groups: true
  flat-network-providers: physnet1
```

A configuração `bridge-interface-mappings` impacta o **OVN Chassis** e se refere a um mapeamento de ponte OVS para interface de rede. Conforme descrito na seção *Criar ponte OVS* na página *Instalar MAAS*, neste exemplo é `br-ex:enp1s0`.

**Nota**

Para usar **endereços de hardware** (em vez de nomes de interface comuns a todos os três nós), a opção `bridge-interface-mappings` pode ser expressa da seguinte forma (substitua pelos seus próprios valores):

**neutron.yaml**

```yaml
bridge-interface-mappings: >-
  br-ex:52:54:00:03:01:02
  br-ex:52:54:00:03:01:03
  br-ex:52:54:00:03:01:04
```

A configuração `flat-network-providers` ativa o provedor de rede **flat** do Neutron usado neste cenário de exemplo e atribui a ele o nome **physnet1**. O provedor de rede flat e seu nome serão referenciados quando configurarmos a **rede pública** na próxima página.

A configuração `ovn-bridge-mappings` mapeia a interface de dados para o provedor de rede flat.

O aplicativo principal do OVN é o **ovn-central**, e ele requer **pelo menos três unidades**. Elas serão **containerizadas** nas máquinas **0, 1 e 2** com o charm **ovn-central**. Para implantar:

```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel 22.03/stable ovn-central
```

O aplicativo **neutron-api** será **containerizado** na máquina **1** com o charm **neutron-api**:

```bash
juju deploy --to lxd:1 --channel yoga/stable --config neutron.yaml neutron-api
```

Implante os charms subordinados com os charms **neutron-api-plugin-ovn** e **ovn-chassis**:

```bash
juju deploy --channel yoga/stable neutron-api-plugin-ovn
juju deploy --channel 22.03/stable --config neutron.yaml ovn-chassis
```

Adicione as relações necessárias:

```bash
juju integrate neutron-api-plugin-ovn:neutron-plugin neutron-api:neutron-plugin-api-subordinate
juju integrate neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
juju integrate ovn-chassis:ovsdb ovn-central:ovsdb
juju integrate ovn-chassis:nova-compute nova-compute:neutron-plugin
juju integrate neutron-api:certificates vault:certificates
juju integrate neutron-api-plugin-ovn:certificates vault:certificates
juju integrate ovn-central:certificates vault:certificates
juju integrate ovn-chassis:certificates vault:certificates
```

Conecte o **neutron-api** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router neutron-api-mysql-router
juju integrate neutron-api-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate neutron-api-mysql-router:shared-db neutron-api:shared-db
```
