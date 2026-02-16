# Processo Seletivo - Desenvolvedor Ruby on Rails Pleno

## Visão Geral

Este teste técnico tem como objetivo avaliar suas habilidades em Ruby on Rails,
arquitetura de microsserviços, comunicação entre APIs e web scraping. Você
desenvolverá um sistema de web scraping de anúncios de veículos que integra
três microsserviços de suporte.

**Tempo estimado:** 7 dias  
**Importante:** Leia todas as instruções antes de começar.

## 📋 Descrição do Projeto

Você desenvolverá um sistema principal de web scraping que gerencia tarefas
de coleta de dados de anúncios de veículos, e se comunica com três
microsserviços de suporte:

1. Sistema Principal - Gerenciamento de tarefas de web scraping
2. Microsserviço de Autenticação - Gerencia login e registro de usuários
3. Microsserviço de Notificações - Recebe e armazena notificações do sistema

## 🎯 Requisitos Funcionais

### 1. Sistema Principal - Web Scraping Manager

**Funcionalidades obrigatórias:**

- **Autenticação de usuários**
  - Tela de login que consome o microsserviço de autenticação
  - Tela de registro de novos usuários
  - Proteção de rotas (apenas usuários autenticados podem acessar)

- **Gerenciamento de Tarefas de Scraping**
  - Criar tarefa: informar URL do anúncio a ser coletado
  - Listar tarefas: visualizar todas as tarefas e seus status
  - Visualizar tarefa: ver detalhes e resultado da coleta
  - Excluir tarefa: remover tarefa do sistema

- **Estrutura da Tarefa**
  - Título/descrição
  - Status: pendente, processando, concluída, falha
  - URL do anúncio (Webmotors)
  - Resultado da coleta (marca, modelo, preço)
  - Mensagem de erro (quando aplicável)
  - Timestamps (criação/atualização/conclusão)
  - Usuário que criou a tarefa

### 2. Microsserviço de Autenticação

**Requisitos técnicos:**

- Utilizar JWT (JSON Web Token) para autenticação
- Validar e-mail e senha com critérios mínimos de segurança
- Retornar token com tempo de expiração

### 3. Microsserviço de Notificações

**Requisitos técnicos:**

- Cada notificação deve conter:
  - Tipo de evento (`task_created`, `task_completed`, `task_failed`)
  - ID da tarefa relacionada
  - Dados do usuário
  - Dados coletados (quando aplicável)

### 4. Microsserviço de Processamento (Web Scraping)

**Requisitos técnicos:**

- Utilizar Nokogiri ou biblioteca similar para scraping
- Coletar dados de anúncios da Webmotors:
  - Marca do veículo
  - Modelo do veículo
  - Preço do veículo
- URL de exemplo:
  - https://www.webmotors.com.br/comprar/bmw/x2/20-turbo-gasolina-xdrive-m35i-steptronic/4-portas/2025-2026/65066397
  - (Se o link expirar, utilize qualquer outro anúncio de veículo do site)
- Processar scraping de forma assíncrona (Sidekiq recomendado)
- Armazenar resultado em banco de dados próprio

## 🛠 Stack Tecnológico

**Obrigatório:**

- Ruby on Rails
- MySQL ou PostgreSQL
- JWT para autenticação
- Nokogiri para web scraping
- Docker e Docker Compose

**Recomendado:**

- Sidekiq para jobs assíncronos
- RSpec para testes
- Rubocop para linting
- Faraday ou HTTParty para comunicação entre serviços

## 📦 Entregáveis

### 1. Código-fonte

1. `webscraping-manager` - Sistema principal
2. `auth-service` - Microsserviço de autenticação
3. `notification-service` - Microsserviço de notificações

### 2. Documentação (`README.md` em cada repositório)

Cada README deve conter:

- Descrição do serviço e sua responsabilidade
- Requisitos (versões Ruby, Rails, dependências)
- Instruções de instalação e configuração
- Como executar com Docker Compose
- Variáveis de ambiente necessárias
- Documentação dos endpoints da API (request/response)
- Como executar os testes
- Diagrama de arquitetura (opcional, mas valorizado)

### 3. Docker Compose

Arquivo `docker-compose.yml` no repositório do sistema principal que sobe
todos os serviços:

- Sistema principal (`webscraping-manager`)
- Microsserviço de autenticação
- Microsserviço de notificações
- Microsserviço de processamento
- Banco(s) de dados (pode ser compartilhado ou separado)
- Redis (se usar Sidekiq)

**Importante:** Deve ser possível subir todo o ambiente com um único comando.

### 4. Vídeo Demonstrativo (5-15 minutos)

Grave um vídeo mostrando todos os serviços funcionando e executando.

## 🚀 Como Submeter

1. Envie os links dos repositórios ou repositório
2. Envie o link do vídeo demonstrativo
3. Certifique-se de que os repositórios estão públicos
4. Inclua no email:
   - Seu nome completo
   - Links dos repositórios
   - Link do vídeo
   - Tempo aproximado que levou para completar
   - Comentários ou observações sobre o projeto (opcional)

Boa sorte! Estamos ansiosos para ver sua solução. 🚀
