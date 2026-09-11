# Documentação de Arquitetura de Infraestrutura Atual — RIEH

**Visão Geral:** Infraestrutura local hospedada no ambiente do **NEES/UFAL**, baseada em máquinas virtuais, containers Docker e gerenciamento através do Portainer. A arquitetura atual não utiliza Kubernetes em produção.

---

### 1. Perfis de Usuários (Acesso)

A plataforma RIEH atende diferentes atores do sistema:

* **Aluno**
* **Professor**
* **Gestor**
* **Curador**
* **Visitante**

Os usuários acessam os serviços do RIEH por meio da infraestrutura hospedada no ambiente do NEES/UFAL.

---

### 2. Infraestrutura e Gerenciamento

A camada principal de execução da aplicação é composta por uma **VM de Produção RIEH**, hospedada na infraestrutura do NEES/UFAL.

* **VM de Produção RIEH:** Ambiente responsável pela execução dos serviços de produção.

* **Docker / Docker Compose:** Responsável pela execução dos serviços da aplicação em containers.

* **Portainer:** Interface de gerenciamento utilizada para administrar os containers, stacks e recursos Docker da produção.

* **Docker Secrets:** Utilizado para armazenar e disponibilizar credenciais sensíveis aos containers, como credenciais de acesso ao banco de dados.

---

### 3. Serviços da Plataforma RIEH

Os sistemas da plataforma são executados como **containers Docker** dentro da infraestrutura de produção:

* **Portal:** Django / Next.js

* **AVA:** Moodle

* **Repositório:** Django / Next.js

* **Observatório:** Django / Next.js

* **Administrativo:** Django (Fullstack)

Diferentemente da arquitetura proposta em Kubernetes, os serviços da produção atual não são executados como Pods, mas diretamente como containers Docker gerenciados pelo Portainer.

#### **Componentes do AVA**

O AVA possui componentes adicionais para seu funcionamento:

* **AVA:** Container principal do Moodle/Apache.

* **AVA-Cron:** Container responsável pela execução das tarefas agendadas do Moodle.

* **Redis:** Utilizado pelo AVA para cache e gerenciamento de sessões.

---

### 4. Banco de Dados e Armazenamento

A infraestrutura utiliza serviços externos aos containers para persistência dos dados.

* **PostgreSQL:** Banco de dados utilizado pelos serviços da plataforma. O banco é executado separadamente dos containers de aplicação.

* **NAS / NFS:** Armazenamento compartilhado utilizado para dados persistentes. No AVA, o armazenamento do `moodledata` é disponibilizado através de NFS.

O **AVA** e o **AVA-Cron** utilizam o armazenamento compartilhado para persistência dos arquivos do Moodle.

---

### 5. Imagens e Implantação dos Containers

* **GitLab UFAL / Container Registry:** Responsável pelo armazenamento das imagens Docker utilizadas pelos serviços RIEH.

* **Docker:** Obtém as imagens disponibilizadas no Container Registry para execução dos serviços na VM de produção.

O fluxo de implantação pode ser representado como:

**GitLab Container Registry → Docker → Containers RIEH**

O **Portainer** é utilizado para gerenciamento da execução dos containers e das stacks em produção.

---

### 6. Logs e Monitoramento

A infraestrutura possui um mecanismo centralizado para recebimento dos logs dos containers:

* **Servidor de Logs:** Recebe os registros gerados pelos serviços da aplicação.

* **GELF:** Os containers enviam seus logs para o servidor utilizando o protocolo/formato GELF.

Fluxo:

**Containers RIEH → GELF → Servidor de Logs**

---

### 7. Características da Arquitetura Atual

* **Infraestrutura Local:** Hospedada no ambiente do NEES/UFAL, sem dependência direta de serviços AWS.

* **Execução em Containers:** Os serviços de produção são executados utilizando Docker.

* **Gerenciamento Centralizado:** O Portainer fornece o gerenciamento dos containers, stacks e Docker Secrets.

* **Persistência Externa:** Banco de dados PostgreSQL e armazenamento NAS/NFS são mantidos fora dos containers de aplicação.

* **Cache e Sessões:** O Redis é utilizado pelo AVA para armazenamento temporário e gerenciamento de sessões.

* **Registro de Imagens:** As imagens Docker são armazenadas no GitLab Container Registry.

* **Centralização de Logs:** Os containers encaminham logs para um servidor externo através de GELF.

* **Kubernetes apenas como Piloto:** Kubernetes não faz parte do ambiente de produção atual.

---

![Diagrama da arquitetura](diagrama.png)


## Pipelines
**AVA (Moodle)**

* **Processo:** Realiza o build da imagem principal do Moodle na raiz do repositório com autenticação via credenciais de build e varredura automatizada de vulnerabilidades.


* **Comportamento e Deploys Interestaduais:** É o projeto com a esteira mais complexa devido à distribuição para diferentes localidades. Possui regras direcionadas para branches específicas de homologação e produção estaduais, disparando webhooks do Portainer para atualizar a aplicação e o serviço de cron (`AVA-Cron`) em ambientes como o geral e instâncias estaduais específicas (a exemplo de Acre e Tocantins através da branch `prod-estados`).



**Portal**

* **Processo:** Constrói a aplicação frontend em Node.js utilizando o `Dockerfile.deploy` e valida a qualidade do código com o ecossistema do SonarQube.


* **Comportamento e Deploys:** O fluxo é acionado em branches de desenvolvimento e feature, realizando o deploy automatizado via webhook para homologação ou produção (`main`) diretamente na VM gerenciada pelo Portainer.



**Observatório (Frontend & Backend)**

* **Processo:** Os ambientes de interface e servidor são mantidos em repositórios e pipelines separados. O front gerencia pacotes Node.js enquanto o back empacota a estrutura em Python.


* **Comportamento e Deploys:** As esteiras monitoram branches dedicadas de liberação (`develop_release` ou `homologacao`) e a branch padrão, acionando atualizações pontuais via Portainer para refletir as melhorias visuais e de API de forma isolada.



**Administrativo (Frontend & Backend)**

* **Processo:** Dividido em pipelines e builds desacoplados para o backend Django e para o proxy Nginx. Antes de gerar as imagens, executa validações rigorosas de código (`flake8`, `ruff`), checagem de migrações e suítes de testes com banco de dados e Redis.


* **Comportamento e Deploys:** O pipeline gerencia ambientes de desenvolvimento, homologação, treinamento e produção de forma segmentada. Conforme o commit na branch correspondente (`dev`, `hmg`, `staging` ou `main`), a esteira dispara requisições independentes para os webhooks do Portainer de cada componente (`$GITLAB_PORTAINER_WEBHOOK_DJANGO`, `$GITLAB_PORTAINER_WEBHOOK_NGINX`), permitindo atualizar o painel administrativo de forma cirúrgica na infraestrutura.