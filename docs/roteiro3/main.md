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

### Keystone

O aplicativo **keystone** será **containerizado** na máquina **0** com o charm **keystone**. Para implantar:

```bash
juju deploy --to lxd:0 --channel yoga/stable keystone
```

Conecte o **keystone** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router keystone-mysql-router
juju integrate keystone-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate keystone-mysql-router:shared-db keystone:shared-db
```

Duas relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate keystone:identity-service neutron-api:identity-service
juju integrate keystone:certificates vault:certificates
```

### RabbitMQ

O aplicativo **rabbitmq-server** será **containerizado** na máquina **2** com o charm **rabbitmq-server**. Para implantar:

```bash
juju deploy --to lxd:2 --channel 3.9/stable rabbitmq-server
```

Duas relações podem ser adicionadas neste momento:

```bash
juju integrate rabbitmq-server:amqp neutron-api:amqp
juju integrate rabbitmq-server:amqp nova-compute:amqp
```

### Nova Cloud Controller

O aplicativo **nova-cloud-controller**, que inclui os serviços **nova-scheduler**, **nova-api** e **nova-conductor**, será **containerizado** na máquina **2** com o charm **nova-cloud-controller**. O arquivo **ncc.yaml** contém a configuração:

**ncc.yaml**

```yaml
nova-cloud-controller:
  network-manager: Neutron
```

Para implantar:

```bash
juju deploy --to lxd:2 --channel yoga/stable --config ncc.yaml nova-cloud-controller
```

Conecte o **nova-cloud-controller** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router ncc-mysql-router
juju integrate ncc-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate ncc-mysql-router:shared-db nova-cloud-controller:shared-db
```

Para manter a saída do `juju status` mais compacta, o nome esperado da aplicação **nova-cloud-controller-mysql-router** foi abreviado para **ncc-mysql-router**.

Cinco relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate nova-cloud-controller:identity-service keystone:identity-service
juju integrate nova-cloud-controller:amqp rabbitmq-server:amqp
juju integrate nova-cloud-controller:neutron-api neutron-api:neutron-api
juju integrate nova-cloud-controller:cloud-compute nova-compute:cloud-compute
juju integrate nova-cloud-controller:certificates vault:certificates
```

### Placement

O aplicativo **placement** será **containerizado** na máquina **2** com o charm **placement**. Para implantar:

```bash
juju deploy --to lxd:2 --channel yoga/stable placement
```

Conecte o **placement** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router placement-mysql-router
juju integrate placement-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate placement-mysql-router:shared-db placement:shared-db
```

Três relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate placement:identity-service keystone:identity-service
juju integrate placement:placement nova-cloud-controller:placement
juju integrate placement:certificates vault:certificates
```

### Horizon - OpenStack Dashboard

O aplicativo **openstack-dashboard** (Horizon) será **containerizado** na máquina **2** com o charm **openstack-dashboard**. Para implantar:

```bash
juju deploy --to lxd:2 --channel yoga/stable openstack-dashboard
```

Conecte o **openstack-dashboard** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router dashboard-mysql-router
juju integrate dashboard-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate dashboard-mysql-router:shared-db openstack-dashboard:shared-db
```

Para manter a saída do `juju status` mais compacta, o nome esperado da aplicação **openstack-dashboard-mysql-router** foi abreviado para **dashboard-mysql-router**.

Duas relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate openstack-dashboard:identity-service keystone:identity-service
juju integrate openstack-dashboard:certificates vault:certificates
```

### Glance 

O aplicativo **glance** será **containerizado** na máquina **2** com o charm **glance**. Para implantar:

```bash
juju deploy --to lxd:2 --channel yoga/stable glance
```

Conecte o **glance** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router glance-mysql-router
juju integrate glance-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate glance-mysql-router:shared-db glance:shared-db
```

Quatro relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate glance:image-service nova-cloud-controller:image-service
juju integrate glance:image-service nova-compute:image-service
juju integrate glance:identity-service keystone:identity-service
juju integrate glance:certificates vault:certificates
```

### Ceph Monitor

O aplicativo **ceph-mon** será **containerizado** nas máquinas **0, 1 e 2** com o charm **ceph-mon**. O arquivo **ceph-mon.yaml** contém a configuração:

**ceph-mon.yaml**

```yaml
ceph-mon:
  expected-osd-count: 3
  monitor-count: 3
```

A configuração acima informa ao cluster MON que ele é composto por **três nós** e que deve esperar **pelo menos três OSDs (discos)**.

Para implantar:

```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --channel quincy/stable --config ceph-mon.yaml ceph-mon
```

Três relações podem ser adicionadas neste momento:

```bash
juju integrate ceph-mon:osd ceph-osd:mon
juju integrate ceph-mon:client nova-compute:ceph
juju integrate ceph-mon:client glance:ceph
```

Sobre as relações acima:

* A relação **nova-compute\:ceph** faz com que o **Ceph** seja o **backend de armazenamento** para as imagens de disco **não bootáveis** do Nova. Para que isso tenha efeito, a opção do charm **nova-compute**, chamada `libvirt-image-backend`, deve estar definida como `'rbd'`.

* A relação **glance\:ceph** faz com que o **Ceph** seja o **backend de armazenamento** para o **Glance**.

### Cinder

O aplicativo **cinder** será **containerizado** na máquina **1** com o charm **cinder**. O arquivo **cinder.yaml** contém a seguinte configuração:

**cinder.yaml**

```yaml
cinder:
  block-device: None
  glance-api-version: 2
```

A opção `block-device` está definida como `'None'` para indicar que o charm **não deve gerenciar dispositivos de bloco**. A opção `glance-api-version` está definida como `'2'` para indicar que deve ser usada a versão 2 da API do Glance.

Para implantar:

```bash
juju deploy --to lxd:1 --channel yoga/stable --config cinder.yaml cinder
```

Conecte o **cinder** ao banco de dados da nuvem:

```bash
juju deploy --channel 8.0/stable mysql-router cinder-mysql-router
juju integrate cinder-mysql-router:db-router mysql-innodb-cluster:db-router
juju integrate cinder-mysql-router:shared-db cinder:shared-db
```

Cinco relações adicionais podem ser adicionadas neste momento:

```bash
juju integrate cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
juju integrate cinder:identity-service keystone:identity-service
juju integrate cinder:amqp rabbitmq-server:amqp
juju integrate cinder:image-service glance:image-service
juju integrate cinder:certificates vault:certificates
```

A relação acima com **glance\:image-service** permitirá que o **Cinder consuma a API do Glance** (por exemplo, possibilitando ao Cinder realizar **snapshots de volumes a partir de imagens do Glance**).

Assim como o Glance, o **Cinder utilizará o Ceph como backend de armazenamento** (daí o uso de `block-device: None` no arquivo de configuração). Isso será implementado por meio do charm subordinado **cinder-ceph**:

```bash
juju deploy --channel yoga/stable cinder-ceph
```

Três relações podem ser adicionadas neste momento:

```bash
juju integrate cinder-ceph:storage-backend cinder:storage-backend
juju integrate cinder-ceph:ceph ceph-mon:client
juju integrate cinder-ceph:ceph-access nova-compute:ceph-access
```

### Ceph RADOS Gateway
