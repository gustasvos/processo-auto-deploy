# Processo de Deploy Automatizado (CI/CD) — Guia Independente de Stack

## 1. Objetivo

Este documento descreve o **padrão de deploy automatizado** usado nos projetos, de forma que ele possa
ser reaplicado em qualquer stack (Spring + React, TS/Node + React, etc.) sem alterar a lógica do
pipeline — apenas os detalhes de build de cada linguagem mudam.

A ideia central: **o pipeline não sabe (nem precisa saber) qual linguagem/framework está rodando dentro
do container**. Ele só lida com três coisas genéricas:

1. Build de uma imagem Docker por serviço
2. Publicação dessa imagem em um registry (Docker Hub, ECR, GHCR, etc.)
3. Atualização do ambiente remoto (servidor) para rodar a nova imagem

---

## 2. Visão geral da arquitetura de deploy

```
┌─────────────┐      push       ┌──────────────────┐
│  Repositório │ ───────────────▶│  GitHub Actions   │
│   (main)     │                 │   (CI runner)     │
└─────────────┘                 └─────────┬─────────┘
                                           │
                       1) docker build     │
                       2) docker push      ▼
                                 ┌───────────────────┐
                                 │   Registry de      │
                                 │   imagens (Hub)    │
                                 └─────────┬─────────┘
                                           │
                       3) SSH: pull + up   ▼
                                 ┌───────────────────┐
                                 │   Servidor (EC2)   │
                                 │  docker compose    │
                                 └───────────────────┘
```

Esse fluxo é **agnóstico de linguagem**: seja um `Dockerfile` de Java com Maven, ou um `Dockerfile` de
Node com `npm run build`, o restante do pipeline (push, cópia de arquivos, SSH, compose up) é idêntico.

---

## 3. Pré-requisitos (uma vez por infraestrutura)

- [ ] Servidor provisionado (EC2 ou equivalente) com Docker e Docker Compose instalados
- [ ] Conta em um registry de imagens (Docker Hub, ECR, GHCR...)
- [ ] Chave SSH de acesso ao servidor cadastrada como secret no repositório
- [ ] Diretório fixo no servidor onde os arquivos de infraestrutura vivem (ex.: `/home/ubuntu/app`)
- [ ] Rede Docker (`networks`) e volumes nomeados definidos no compose de produção

### Secrets genéricos necessários no GitHub

| Secret | Uso |
|---|---|
| `REGISTRY_USERNAME` / `REGISTRY_TOKEN` | Login no registry de imagens |
| `EC2_HOST` | IP/DNS do servidor |
| `EC2_USER` | Usuário SSH |
| `EC2_SSH_KEY` | Chave privada SSH |
| `*_PASS` (DB, MQ, admin, etc.) | Credenciais de cada peça de infraestrutura, injetadas como variável de ambiente no compose |

> Esses nomes são só um padrão sugerido — o ponto é que **toda credencial fica em secrets**, nunca no
> compose ou no código.

---

## 4. Etapas genéricas do pipeline

Independente da stack, todo `deploy.yml` segue as mesmas fases lógicas:

### Etapa 1 — Build & Push (por serviço)
1. Checkout do código
2. Login no registry
3. Build da imagem a partir do `Dockerfile` de cada serviço
4. Push da imagem com uma tag (`:latest` ou, idealmente, `:${{ github.sha }}` — ver seção 7)

Isso se repete **uma vez por serviço** que existir no repositório (1 serviço = 1 bloco de build/push).
Se amanhã os 4 microsserviços Spring viram 4 serviços Node, ou viram 1 monólito, só muda a
**quantidade de blocos**, não a estrutura de cada bloco.

### Etapa 2 — Entrega dos arquivos de infraestrutura
Copiar para o servidor apenas o que é necessário para **orquestrar** os containers, nunca o código-fonte:
- `docker-compose.prod.yml`
- Pastas de configuração/inicialização (scripts SQL, realm exports, temas, certificados, etc.)

### Etapa 3 — Deploy remoto via SSH
No servidor, sempre as mesmas duas ações:
```bash
docker compose -f docker-compose.prod.yml pull [serviço opcional]
docker compose -f docker-compose.prod.yml up -d --remove-orphans [serviço opcional]
```
- Sem argumento de serviço → atualiza tudo (caso do backend, com vários serviços dependentes entre si)
- Com argumento de serviço → atualiza só aquele container (caso do frontend, deploy independente)

---

## 5. Template genérico de workflow

```yaml
name: Deploy <NOME_DO_SERVICO_OU_PROJETO>
on:
  push:
    branches: [ "main" ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login no registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - name: Setup Buildx
        uses: docker/setup-buildx-action@v3

      # Repetir este bloco para cada serviço do repositório
      - name: Build & push <nome-servico>
        uses: docker/build-push-action@v5
        with:
          context: .
          file: <caminho>/Dockerfile
          push: true
          tags: ${{ secrets.REGISTRY_USERNAME }}/<nome-servico>:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Copiar arquivos de infra para o servidor
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "docker-compose.prod.yml,infra-*"
          target: "/home/ubuntu/app"

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          envs: REGISTRY_USER
          script: |
            cd /home/ubuntu/app
            export REGISTRY_USER=${{ secrets.REGISTRY_USERNAME }}
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d --remove-orphans
```

**O que muda de projeto para projeto:** só o caminho dos `Dockerfile`s, o nome das imagens e a
quantidade de blocos de build. O restante do YAML é copy-paste.

---

## 6. O que fica de fora do pipeline (e por quê)

| Item | Onde vive | Motivo |
|---|---|---|
| Código-fonte compilado/buildado | Dentro da imagem Docker | O pipeline não sabe (nem precisa) como o `Dockerfile` builda — Maven, npm, etc. |
| Variáveis de build específicas do front (ex. `VITE_API_URL`) | `build-args` do próprio serviço | São detalhe de implementação, não do processo |
| Segredos de aplicação | Secrets do GitHub → env vars do compose | Nunca versionados |
| Lógica de orquestração (quem depende de quem, health checks) | `docker-compose.prod.yml` | O compose é o "contrato" de infraestrutura; o pipeline só aciona `pull`/`up` |

Isso é o que te permite reusar o mesmo `deploy.yml` entre o projeto Spring e o projeto TS/Node: a
diferença de stack fica isolada dentro de cada `Dockerfile` e do compose, nunca no workflow.

---

## 7. Pontos de atenção / melhorias recomendadas

Coisas que já dá pra observar nos workflows que você tem hoje e que valem registrar como
recomendação para o novo projeto:

1. **Tag `:latest` é frágil.** Sem uma tag única (ex. `${{ github.sha }}`), não dá pra saber qual commit
   está rodando em produção nem fazer rollback fácil. Recomenda-se taguear com `latest` **e** com o SHA.
2. **Múltiplos repositórios, um único `docker-compose.prod.yml`.** O compose "canônico" deveria viver
   em um lugar só (ex. repositório de infra) e cada pipeline de serviço apenas dar `pull`/`up` do que é
   dele — hoje o compose do backend parece ser copiado de novo a cada deploy do backend, o que é OK,
   mas o do frontend depende que ele já exista no servidor (não copia o compose).
3. **Credenciais em texto plano no compose atual** (ex. `rabbit`/`rabbit` fixos, `mongouser` fixo) —
   ideal é que também venham de variável de ambiente/secret, como já é feito com `DB_APP_PASS`.
4. **Healthchecks e `depends_on: condition: service_healthy`** são o mecanismo que te dá deploy
   confiável independente da stack — mantenha esse padrão em qualquer projeto novo, cada serviço deve
   expor uma forma de checar se está pronto.
5. **Padronizar nomes de secrets** entre projetos (ex. `REGISTRY_USERNAME` em vez de
   `DOCKERHUB_USERNAME`) facilita reusar o workflow-template ipsis litteris entre repositórios.

---

## 8. Checklist para aplicar em um novo projeto (ex.: TS/Node + React)

- [ ] Cada serviço tem seu próprio `Dockerfile`
- [ ] Existe um `docker-compose.prod.yml` único descrevendo toda a infraestrutura + serviços
- [ ] Cada serviço tem `healthcheck` definido
- [ ] Variáveis sensíveis vêm de `${VAR}` no compose, preenchidas por secrets do GitHub
- [ ] Workflow de deploy segue o template da seção 5, com um bloco de build por serviço
- [ ] Deploy do frontend (se for repo separado) só dá `pull`/`up` do próprio serviço
- [ ] Deploy do backend dá `pull`/`up` sem filtro, respeitando as dependências do compose
- [ ] Nenhuma credencial fixa no compose — tudo via variável de ambiente/secret

---

## 9. Resumo

O processo de deploy automatizado, no seu núcleo, é sempre:

**build imagem → push imagem → copiar infraestrutura → pull + up no servidor**

Tudo que é específico de stack (Java/Spring, TS/Node, etc.) fica confinado dentro de cada `Dockerfile`.
O `deploy.yml` e o `docker-compose.prod.yml` são o "contrato" reutilizável entre projetos.