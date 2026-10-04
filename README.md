# github-actions-cicd-kubernetes

Pipeline de CI/CD com GitHub Actions que testa uma API em Go, publica a imagem no Docker Hub e entrega a aplicação em um cluster Kubernetes (Amazon EKS) provisionado com Terraform.

Projeto do curso **Integração Contínua: automatizando a entrega no Kubernetes**, da Alura. Último passo da série iniciada em [github-actions-ci](https://github.com/ssdvd/github-actions-ci).

## O pipeline

```
push / pull request
        │
      test ──► build ──► docker ──► deploy_eks
```

| Job | Workflow | O que faz |
| --- | --- | --- |
| `test` | [`go.yml`](.github/workflows/go.yml) | Matriz com Go `1.19`, `1.20` e `>=1.20`; sobe o PostgreSQL com `docker-compose` |
| `build` | [`go.yml`](.github/workflows/go.yml) | Publica o binário `main` como artefato (`app_go`) |
| `docker` | [`docker.yml`](.github/workflows/docker.yml) | Builda e envia a imagem `ssdvd/go_ci:<número da execução>` |
| `deploy_eks` | [`eks.yml`](.github/workflows/eks.yml) | Provisiona a infraestrutura, configura o `kubectl`, cria os secrets e atualiza o deployment |

### Deploy no EKS

1. Clona o repositório de infraestrutura [Infra_CI_Kubernetes](https://github.com/ssdvd/Infra_CI_Kubernetes) e roda o Terraform no ambiente `env/Homolog`.
2. Gera o kubeconfig com `aws eks update-kubeconfig` para o cluster `homolog2`.
3. Recria os secrets do Kubernetes (`dbhost`, `dbport`, `dbuser`, `dbpassword`, `dbname`, `port`) a partir dos secrets do repositório e do endereço do banco devolvido pelo Terraform.
4. Aplica o manifesto `go.yaml` e troca a imagem com `kubectl set image deployment/go-api`.

Os workflows [`ec2.yml`](.github/workflows/ec2.yml), [`ecs.yml`](.github/workflows/ecs.yml) e [`loadtest.yml`](.github/workflows/loadtest.yml) dos projetos anteriores continuam no repositório, mas não são chamados pelo `go.yml`.

> Estado atual: o `eks.yml` ainda tem passos de depuração do curso (um `terraform destroy` seguido de `return 1`), que interrompem o job antes do `apply`. Os passos `go test` e `go build` também estão comentados no `go.yml`.

### Secrets necessários

| Secret | Uso |
| --- | --- |
| `USER_DOCKERHUB`, `PW_DOCKERHUB` | Login no Docker Hub |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Credenciais da AWS (região `us-east-2`) |
| `DBPORT`, `DBUSER`, `DBPASSWORD`, `DBNAME` | Conexão com o PostgreSQL |

## A aplicação

API REST de cadastro de alunos escrita em Go com [Gin](https://gin-gonic.com/) e [GORM](https://gorm.io/), usando PostgreSQL. É a aplicação de exemplo dos cursos da Alura (`guilhermeonrails/api-go-gin`); o foco deste repositório é o pipeline, não a API.

| Método | Rota | Descrição |
| --- | --- | --- |
| `GET` | `/:nome` | Saudação em JSON |
| `GET` | `/alunos` | Lista todos os alunos |
| `GET` | `/alunos/:id` | Busca um aluno pelo ID |
| `GET` | `/alunos/cpf/:cpf` | Busca um aluno pelo CPF |
| `POST` | `/alunos` | Cria um aluno (`nome`, `cpf` com 11 dígitos, `rg` com 9 dígitos) |
| `PATCH` | `/alunos/:id` | Edita um aluno |
| `DELETE` | `/alunos/:id` | Remove um aluno |
| `GET` | `/index` | Página HTML com a lista de alunos |

## Rodando localmente

Pré-requisitos: Go 1.19 ou superior, Docker e Docker Compose.

```bash
# sobe o PostgreSQL e o pgAdmin
docker-compose up -d

# variáveis lidas pela aplicação para conectar no banco
export HOST=localhost USER=root PASSWORD=root DBNAME=root DBPORT=5432

go run main.go              # API em http://localhost:8080
go test -v main_test.go     # testes de integração (precisam do banco no ar)
```

O pgAdmin fica em <http://localhost:54321>.

## Série de CI/CD com GitHub Actions

Este repositório faz parte de uma sequência em que o mesmo pipeline vai ganhando etapas:

| # | Repositório | O que acrescenta |
| --- | --- | --- |
| 1 | [github-actions-ci](https://github.com/ssdvd/github-actions-ci) | Testes automatizados e matriz de versões do Go |
| 2 | [github-actions-ci-docker](https://github.com/ssdvd/github-actions-ci-docker) | Build da imagem e push para o Docker Hub |
| 3 | [github-actions-cicd-ec2](https://github.com/ssdvd/github-actions-cicd-ec2) | Deploy contínuo em uma instância EC2 via SSH |
| 4 | [github-actions-cicd-ecs](https://github.com/ssdvd/github-actions-cicd-ecs) | Deploy contínuo no Amazon ECS |
| 5 | [github-actions-cicd-rollback-tests](https://github.com/ssdvd/github-actions-cicd-rollback-tests) | Rollback automático e teste de carga |
| 6 | [github-actions-cicd-kubernetes](https://github.com/ssdvd/github-actions-cicd-kubernetes) | Deploy contínuo no Kubernetes (EKS) |

As anotações de cada aula estão na pasta [`notes/`](notes).
