# Objetivos

1. Entender os conceitos básicos Infraestrutura como código.
2. Entender os conceitos básicos sobre SLA e DR.

# Infra

A infraestrutura necessária para cumprir com os objetivos desse roteiro consiste em segmentar por aluno - de forma a criar uma separação lógica entre os recursos utilizados por estes - e estruturar uma hierarquia de projeto para cada através do dashboard do OpenStack.
Para isso, seguimos o seguinte passo-a-passo:

## 1. Criação de um Domain único

No dashboard do OpenStack, nevagamos até Identity > Domains, clicamos em "Create Domain" e adicionamos o domínio AlunosDomain. Em seguida, definimos AlunosDomain como o novo contexto de uso.

## 2. Criação de um projeto para cada aluno

Em Identity > Projects, criamos os projetos `KitV_Vinicius` e `KitV_AndersonFranco`. Em seguida, em Identity > Users, criamos os usuários Vinicius, com papel administrativo no projeto KitV_Vinicius, e AndersonFranco, com papel administrativo no projeto KitV_AndersonFranco.