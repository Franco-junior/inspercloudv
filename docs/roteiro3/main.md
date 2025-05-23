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
