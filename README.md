
# 🐾 Pipeline CI/CD com Jenkins e Docker para Testes Automatizados de API

> Pipeline CI/CD com Jenkins, Docker e Nginx que executa testes automatizados de API com Postman/Newman a cada push.

![Postman](https://img.shields.io/badge/Postman-FF6C37?logo=postman\&logoColor=white)
![Newman](https://img.shields.io/badge/Newman-ef5b25?logo=postman\&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?logo=nodedotjs\&logoColor=white)
![npm](https://img.shields.io/badge/npm-CB3837?logo=npm\&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?logo=github\&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-000000?logo=json\&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker\&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins\&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?logo=nginx\&logoColor=white)


Projeto acadêmico desenvolvido na disciplina de Qualidade e Gerência de Software do INATEL. Ele testa a API pública Swagger Petstore com 21 casos de teste automatizados e executa tudo dentro de um pipeline Jenkins totalmente containerizado.


## 📑 Sumário
 
1. [O Problema que Este Repositório Resolve](#1--o-problema-que-este-repositório-resolve)
2. [Regras de Negócio](#2--regras-de-negócio)
3. [Como Executar](#3--como-executar)
4. [Arquitetura](#4--arquitetura)
5. [Decisões Importantes](#5--decisões-importantes)
6. [Desafios](#6--desafios)
7. [Testes](#7--testes)
8. [Documentação](#8--documentação)
9. [Decisões Técnicas](#9--decisões-técnicas)
10. [Equipe e Uso de IA](#-equipe-e-uso-de-ia)


## 1. 🎯 O Problema que Este Repositório Resolve
 
Executar testes de API manualmente é lento, fácil de esquecer e difícil de reproduzir: o resultado depende de quem roda os testes, em qual máquina e com qual configuração. Regressões são descobertas tarde e não existe um registro compartilhado do que passou ou falhou.
 
Este projeto resolve isso transformando a suíte de testes em um **pipeline automatizado e reproduzível**:
 
- Cada push no repositório dispara a execução dos testes, sem nenhuma etapa manual.
- Toda a infraestrutura (Jenkins, agent, Newman, Nginx) é definida como código e sobe com um único comando.
- Cada execução gera relatórios HTML/JSON e uma notificação por e-mail, dando à equipe um retorno rápido e visível.


## 2. 📏 Regras de Negócio
 
As regras abaixo são aplicadas pelo pipeline (`Jenkinsfile`) e pela suíte de testes:
 
| # | Regra |
|---|-------|
| 1 | O pipeline executa automaticamente a cada push na `main` (gatilho de push do GitHub). |
| 2 | Apenas **um build roda por vez** (builds concorrentes estão desabilitados), evitando interferência entre execuções. |
| 3 | O build é abortado se passar de **30 minutos**. |
| 4 | Somente os **últimos 5 builds** e seus artefatos são mantidos, para limitar o uso de disco. |
| 5 | A cobertura de testes deve ser de **no mínimo 90%**. Abaixo disso, o build fica **UNSTABLE** (instável), e não falho. |
| 6 | Todo build envia um **e-mail** com o status final e a duração. |
| 7 | Relatórios de teste e o pacote da suíte são **arquivados** como artefatos do build. |
| 8 | Os testes são divididos em **Caminho Feliz (Happy Path)** e **Caminho Negativo (Negative Path)**. |
| 9 | Os testes usam **variáveis de ambiente**, mantendo as execuções independentes entre si. |
| 10 | Credenciais (e-mail, segredo do agent) **nunca são versionadas**; são injetadas por variáveis de ambiente. |

## 3. 🚀 Como Executar
 
### Pré-requisitos
 
- [Docker](https://docs.docker.com/get-docker/) e Docker Compose
- Git
- Uma conta Gmail com [senha de app](https://support.google.com/accounts/answer/185833) (apenas se quiser as notificações por e-mail)
### Passo a passo
 
**1. Clone o repositório**
 
```bash
git clone https://github.com/schulzdimitri/jenkins-docker-cicd-pipeline.git
cd jenkins-docker-cicd-pipeline
```
 
**2. Crie um arquivo `.env`** na raiz do projeto com as variáveis usadas pelo `docker-compose.yml`:
 
```env
AGENT_SECRET=<segredo do nó agent do Jenkins>
EMAIL_DESTINO=<endereço que recebe o relatório>
EMAIL_REMETENTE=<endereço que envia o relatório>
APP_PASSWORD=<senha de app da conta remetente>
```
 
**3. Suba todos os serviços**
 
```bash
docker compose up -d --build
```
 
**4. Acesse o Jenkins**
 
| URL | Acesso por |
|-----|------------|
| http://localhost | Proxy reverso Nginx (porta 80) |
| http://localhost:8080 | Jenkins diretamente |
 
**5. Conecte o agent e execute o pipeline**
 
- O agent roda como `s07-jenkins-agent` e o pipeline espera o label de nó `agent`.
- Copie o segredo do nó no Jenkins (*Manage Jenkins → Nodes*) para o `AGENT_SECRET` e reinicie o agent: `docker compose up -d agent`.
- Crie um job do tipo Pipeline apontando para o `Jenkinsfile` deste repositório e clique em **Build Now**.
> Para o disparo automático a cada push, o Jenkins precisa estar acessível pelo webhook do GitHub (por exemplo, por uma URL pública ou um túnel).
 
### Comandos úteis
 
```bash
docker ps                  # containers em execução
docker compose logs -f     # acompanhar logs
docker compose down        # encerrar tudo
```


## 4. 🏗️ Arquitetura
 
### Serviços
 
| Serviço | Container | Responsabilidade |
|---------|-----------|------------------|
| Jenkins | `s07-jenkins` | Controlador CI/CD; agenda e registra as execuções do pipeline (portas 8080 e 50000) |
| Jenkins Agent | `s07-jenkins-agent` | Executa as etapas do pipeline |
| Newman | `s07-newman` | Executa a collection do Postman contra a API alvo |
| Nginx | `s07-nginx` | Proxy reverso que expõe o Jenkins na porta 80 |
 
### Imagem no Docker Hub

A imagem do Jenkins (com Node.js, Docker CLI e os plugins do `jenkins/plugins.txt` já instalados via `Dockerfile.jenkins`) está publicada publicamente no Docker Hub:

🔗 **[kaikfreitas/s07-jenkins](https://hub.docker.com/r/kaikfreitas/s07-jenkins)**

Ela é referenciada diretamente no `docker-compose.yml` (`image: kaikfreitas/s07-jenkins:latest`) e baixada automaticamente ao rodar `docker compose up`.

### Arquitetura da Solução

<div align="center">
  <table>
    <tr>
      <td>
        <img src="docs/images/arquitetura.svg" alt="Arquitetura da Solução" width="1000">
      </td>
    </tr>
  </table>
</div>


## 5. 🧭 Decisões Importantes
 
- **Testar a API com Postman + Newman.** O Postman torna o desenho dos testes acessível a toda a equipe, e o Newman executa a mesma collection pela linha de comando, o que viabiliza a automação.
- **Containerizar tudo.** Docker e Docker Compose garantem um ambiente reproduzível: qualquer pessoa sobe o pipeline completo com um comando.
- **Usar o Jenkins como servidor de CI/CD.** É amplamente usado no mercado e atende ao objetivo da disciplina de aprender como pipelines reais são construídos.
- **Tratar a cobertura como portão de qualidade.** O mínimo de 90% deixa o build UNSTABLE em vez de falho, tornando a baixa cobertura visível sem esconder quais testes realmente passaram.
- **Notificar por e-mail.** A equipe recebe o resultado sem precisar abrir o Jenkins.
- **Usar IA como ferramenta de co-criação, com revisão humana.** A IA apoiou scripts, documentação e infraestrutura, e todo artefato gerado foi revisado e validado pela equipe (veja [Equipe e Uso de IA](#-equipe-e-uso-de-ia)).


## 6. 🧗 Desafios
 
Desafios tratados no desenho do pipeline:
 
- **Isolamento dos testes:** os testes dependem de variáveis de ambiente e os relatórios antigos do Newman são removidos antes de cada execução, para que resultados de um build anterior nunca contaminem o seguinte.
- **Executar Docker de dentro do Jenkins:** o agent precisa rodar `docker exec` e `docker cp` no container do Newman, o que exigiu montar o socket do Docker nos containers do Jenkins e do agent.
- **Conectividade do agent:** o agent se autentica no controlador com um segredo, então a primeira inicialização exige uma pequena etapa manual antes de os builds rodarem.
- **Gestão de segredos:** as credenciais de e-mail e o segredo do agent precisavam chegar ao pipeline sem serem versionados, por isso são passados como variáveis de ambiente.
- **Condições de corrida entre builds:** builds concorrentes foram desabilitados para evitar que duas execuções escrevam no mesmo container do Newman e na mesma pasta de relatórios.


## 7. 🧪 Testes
 
A suíte tem **21 casos de teste (TC)** com identificação única:
 
| Faixa | Tipo | Objetivo |
|-------|------|----------|
| TC-001 a TC-010 | Caminho Feliz | Validar que os fluxos principais funcionam como esperado |
| TC-011 a TC-021 | Caminho Negativo | Validar o tratamento de erros e entradas inválidas |
 
**Como os testes são executados no pipeline**
 
1. `Checkout`: clona o repositório.
2. `Build`: instala as dependências com `npm ci` e empacota a collection e os environments em `pacote-testes.tar.gz`.
3. `Tests`: limpa relatórios antigos, executa a collection do Postman com o Newman (`npm test`) e copia os relatórios para fora do container do Newman.
4. `Test Coverage`: calcula a cobertura a partir do relatório do Newman (`npm run coverage`); abaixo de 90% o build fica UNSTABLE.
5. `Notification`: envia o resultado por e-mail.
**Relatórios:** relatórios HTML (via `newman-reporter-htmlextra`) e JSON são arquivados em cada build do Jenkins.
  

## 8. 📚 Documentação
 
- **Diagrama de arquitetura:** [`docs/images/arquitetura.svg`](docs/images/arquitetura.svg)
- **Diagrama do fluxo do pipeline:** [`docs/images/pipeline_fluxo.svg`](docs/images/pipeline_fluxo.svg)
  ![Fluxo do pipeline](docs/images/pipeline_fluxo.svg)
- **Definição do pipeline:** [`Jenkinsfile`](Jenkinsfile), com cada etapa descrita nas mensagens de log
- **Suíte de testes:** [`postman/`](postman/) (collection e environments)
- **Imagem do Jenkins no Docker Hub:** [`kaikfreitas/s07-jenkins`](https://hub.docker.com/r/kaikfreitas/s07-jenkins), referenciada no `docker-compose.yml` e baixada automaticamente no `docker compose up`
 


## 9. 🔧 Decisões Técnicas
 
| Decisão | Motivo |
|---------|--------|
| **Pipeline declarativo** (`Jenkinsfile`) | Pipeline como código: versionado, revisável e fácil de ler. |
| **Jenkins agent dedicado** (label `agent`) | Separa a execução dos testes do controlador Jenkins. |
| **Socket do Docker montado** no Jenkins e no agent | Permite ao pipeline controlar o container do Newman (`docker exec`, `docker cp`). |
| **Imagem do Jenkins pré-construída no Docker Hub** (Node.js, Docker CLI, plugins do `plugins.txt`) | Inicialização mais rápida e conjunto de plugins consistente para todos. |
| **Setup wizard do Jenkins desabilitado** | Remove passos manuais da primeira execução e acelera a subida do ambiente. |
| **Healthcheck no Jenkins** (`/login`) | Permite ao Docker detectar quando o controlador está realmente pronto. |
| **Volume nomeado `jenkins_data`** | Configuração e histórico de builds do Jenkins sobrevivem a reinicializações dos containers. |
| **Nginx como proxy reverso** | Ponto de entrada único na porta 80 na frente do Jenkins. |
| **`npm ci`** em vez de `npm install` | Instalações reproduzíveis baseadas no `package-lock.json`. |
| **Opções do pipeline:** timeout, retenção de builds, timestamps, logs coloridos, sem builds concorrentes | Execuções previsíveis, legíveis e econômicas em recursos. |
| **`archiveArtifacts`** no bloco `post` | Os relatórios são mantidos mesmo quando o build falha ou fica instável. |
| **Segredos via variáveis de ambiente** | As credenciais ficam fora do repositório. |

## 👥 Equipe e Uso de IA
 
Este projeto foi desenvolvido em equipe e é um fork do repositório original: [kafreitas07/Project_01_group_02_S07](https://github.com/kafreitas07/Project_01_group_02_S07).
 
- [André Augusto](https://github.com/andreaugust0)
- [Arthur Rabelo](https://github.com/arthurrabeloo)
- [Marcus Vinicius](https://github.com/MarcusSouza02)
- [Kaik Freitas](https://github.com/kafreitas07)
- [Dimitri Schulz](https://github.com/schulzdimitri)
- [Anna Rennó](https://github.com/acrenno)

**Uso de IA:** modelos de linguagem (como o Google Gemini) apoiaram a geração de scripts, a documentação, o planejamento de testes, a containerização e o pipeline CI/CD. Todos os artefatos gerados por IA foram revisados, testados e validados pela equipe.
