---
title:
description:
draft: false
publish: true
permalink:
created: 2026-07-26
published: 2026-07-26
tags:
aliases:
image:
quartzProperties: false
quartzPropertiesCollapse: true
enableToc: true
---

Se refere aos 12 pilares de software que devem ser levados em consideração em um software de qualidade.
https://12factor.net/

## Introdução

Atualmente, grande parte dos softwares são entregues como serviços ou web-apps. A ideia é construir apps que são:

- Declarativos
- Portáteis ao máximo
- Deployáveis em plataformas de cloud modernas, sem necessidade de um servidor ou sysadmin.
- Pouca divergência entre desenvolvimento e produção
- Podem ser escaláveis sem grandes mudanças no ferramental, arquitetura ou práticas de desenvolvimento

Essa metodologia pode ser aplicada a qualquer linguagem de programação e qualquer combinação de serviços (databases, caches, filas, etc)

## 1. Codebase

- Manter código em apenas um local
- Versionado com VCS - como o Git
- Um código múltiplos deploys/ambientes/versões

## 2. Dependências

- Dependências sempre declaradas
- Isolamento e declaração de dependências (pip e virtualenv por exemplo)
- Declarações facilitam novos desenvolvedores a se envolverem no projeto
- Não depender da existência implícita de ferramentas (se necessário, garantir a instalação)

## 3. Config

- Uma config é tudo o que pode diferir e variar entre.
- Nunca salvar config como constantes no código.
        - Um bom teste é avaliar se a aplicação pode virar de código aberto sem comprometer credenciais.
- Essa config não é a configuração da aplicação, que será a mesma entre deploys.
        - Quando é a mesma, o ideal é que seja no código mesmo.
- Configurações sempre em variáveis de ambiente.
        - Agnósticas de linguagem e SO.
- Configs nunca são agrupadas como environmentes, mas são gerenciadas independentemente por deploy, o que garante escalabilidade.

## 4. Backing Services

- Serviços terceiros (externos) à aplicação.
- Sempre tratados como parte da aplicação, como recursos attachados.
- A aplicação deve conseguir trocar um MySQL por um RDS sem mudanças no código.
        - Trocar apenas o 'handle' e a aplicação ainda consegue se virar.
- Cada serviço é um recurso, uma database é um recurso. Duas databases (sharding na camada de aplicação) são dois recursos distintos.
- Cada recurso deve poder ser attachado e detachado de deploys sem mudança de código.

## 5. Buildar, Lançar, Executar (Build, Release, Run)

- Separar estágios de build e execução.
- O estágio de build transforma o repositório em um 'executável'. Instala dependências, compila binários e assets, e etc.
- O Release pega o build e aplica a [[#3. Config|config]] do deploy em questão e prepara para execução no ambiente desejado.
- O Run (runtime) executa o ambiente, lançando a aplicação.
        - Não pode ser feito alterações de código, pois é impossível replicar ao build.

- Toda release deve ter um ID específico, como um timestamp ou número de versão incremental.
- A release deve ser como um ledger: mudanças sempre 'appendam' e nunca mudam o passado.
- Toda mudança deve gerar uma nova versão.
- A Run deve ter o mínimo de partes móveis, para evitar problemas que impeça o run de acontecer.
- O estágio de build pode ser mais complexo e propenso a erros (já que esses não serão replicados à produção)

## 6. [Processos](https://12factor.net/processes)

- Executar a aplicação como um ou mais processos stateless
- Aplicações de 12 fatores são stateless e não compartilham nada. Quaisquer dados que precisem persistir são armazenados em um [[#4. Backing Services|backing service]] stateful, como uma database.
- A aplicação não assume que nada cacheado em memória ou disco estará disponível em request futura.
- Deploys costumam limpar todo o estado, por isso é importante se manter stateless.
- Não usar sticky-sessions. Se necessário, salvar sessões em datastored com expiração (memcached, redis)

## 7. [Port Binding](https://12factor.net/port-binding)

- Exportar serviços via port-binding.
- Aplicações 12-factor são 'self-contained' e não usam runtime injection. A aplicação exporta HTTP como serviço bindando a uma porta e ouvindo requests.
- Bindar um serviço a uma porta e ouvir conexões.
- Usar port-binding permite que uma aplicação se torne um [[#4. Backing Services|backing service]] para outra, provendo URL para ser usada como um handle de [[#3. Config|config]].

## 8. [Concorrência](https://12factor.net/concurrency) - escala via modelo de processos

- Programas são representados por uma ou várias aplicações simultâneamente
        - aplicações php spawnam vários processos do apache, on-demand
        - aplicações java possuem um JVM maior que reserva todos os recursos que podem ser necessários
- Processos devem ser 'first class citizens' em aplicações 12-factor.
- Arquitetar a aplicação para que cada tipo de trabalho seja processado por um tipo de processo:
        - requests http > processo web
        - jobs em background > worker
- A aplicação deve ser capaz de espalhar processos em multiplas máquinas/instâncias
- Aplicações não devem 'daemonizar' ou escrever arquivos de PID.
        - Depender sempre de systemd ou afins para determinar fluxos de saída (logs), processar restarts e shutdowns ou responder a 'crashes'

## 9. [Disposability](https://12factor.net/disposability)

- Processos devem ser descartáveis, sendo iniciado ou parados a qualquer momento.
- Isso implica em maior velocidade de atualização da [[#1. Codebase|codebase]] ou [[#3. Config|configs]] e maior robustez no processo de deploy.
- Minimizar startup time
        - Scales mais rápidos
        - Lançar [[#5. Buildar, Lançar, Executar (Build, Release, Run)|releases]] mais rápido
- Graceful shutdown
        - Parar de ouvir na porta instantaneamente, finalizar a request e encerrar o processo
        - Para pollings mais longos, a aplicação deve conseguir encerrar e tentar uma reconexão automaticamente em outro local ([[#8. [Concorrência](https //12factor.net/concurrency) - escala via modelo de processos|concorrência]])
- A aplicação também deve conseguir lidar com falhas repentinas.
- Retornar o job para uma fila automaticamente quando ele for encerrado (caso de workers)
        - A execução deve ser idempotente
        - Quaisquer locks devem ser liberados ao encerrar o processo
        - O código deve permitir [re-entrada](https://en.wikipedia.org/wiki/Reentrancy_(computing))

## 10. [Paridade Dev/Prod](https://12factor.net/dev-prod-parity)

- Continuous Deployment deve garantir que o gap entre produção e desenvolvimento seja mínimo:
        - O código escrito deve ser deployado logo após ser escrito
        - Desenvolvedores devem estar envolvidos no processo de deploy e acompanhamento do código que foi escrito, observando seu comportamento
        - Os ambientes devem ser o mais semelhante quando possível
- Os [[#4. Backing Services|backing services]] deve estar sempre atualizados e iguais entre ambos os ambientes (não usar conectores de bancos de dados diferentes, por exemplo), mesmo que seja deployado localmente

## 11. [Logs](https://12factor.net/logs)

- Tratar logs como streams de eventos
- A aplicação não deve se preocupar com roteamento ou armazenamento da stream de eventos.
- Cada processo escreve sua event stream à stdout.
- A coleta da stream ou destino de arquivamento não deve ser visível ou configurável a partir da aplicação. Quem cuida disso é o ambiente de execução.
- O ambiente pode rotear o stream para um indexador de logs (Splunk, Opensearch, etc) ou data-warehouse que pode trabalhar com os dados, ou mesmo gerar alertas.

## 12. [Processos Administrativos](https://12factor.net/admin-processes)

- Rodar migrations em databases ou REPL shell como one-off process e devem ser executados contra uma [[#5. Buildar, Lançar, Executar (Build, Release, Run)|release]] específica.
- Essas execuções devem ocorrer de forma semelhante à execuções de longa duração da aplicação, usando o mesmo [[#1. Codebase|código]] e [[#3. Config|config]] que qualquer outro processo usa contra a mesma release.