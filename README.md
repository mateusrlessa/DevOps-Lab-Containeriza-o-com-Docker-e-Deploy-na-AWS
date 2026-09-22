# DevOps Lab — Containerização com Docker e Deploy na AWS

Laboratório prático que reproduzi seguindo o conteúdo da **Maria Lazara** ([DevOps Engineer](https://www.youtube.com/@marialazaradev)), parte da série **Laboratório DevOps: Aprenda DevOps na Prática com Projetos Progressivos**.

Website estático (HTML, CSS e JavaScript) containerizado com **Docker** e publicado manualmente em uma instância **EC2** na AWS, usando o **Amazon ECR** como registro de imagens.

---

## O que eu pratiquei neste laboratório

- **Containerização** de uma aplicação web com **Docker** (imagem baseada em `nginx:alpine`)
- **Registro de imagens** privado na nuvem com **Amazon ECR**
- **Provisionamento manual de infraestrutura** com **Amazon EC2** (AMI, tipo de instância, key pair)
- **Configuração de rede e segurança** via **Security Groups** (regras de entrada para SSH e HTTP)
- **Permissões de acesso entre serviços AWS** usando **IAM Roles** (EC2 → ECR)
- **Deploy de container em produção**, incluindo restart automático (`--restart always`)

---

## Arquitetura

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Código Local   │────▶│   Docker Image  │────▶│    Amazon ECR   │
│  (HTML/CSS/JS)  │     │   (Container)   │     │   (Registry)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                          │
                                                          ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │    Browser      │◀────│    Amazon EC2   │
                        │  (User Access)  │     │   (Container)   │
                        └─────────────────┘     └─────────────────┘
```

---

## Stack utilizada

| Ferramenta | Papel no projeto |
|---|---|
| **Docker** | Containerização do website (imagem `nginx:alpine`) |
| **Amazon ECR** | Registro privado de imagens Docker |
| **Amazon EC2** | Instância que executa o container em produção |
| **Security Groups** | Firewall virtual — libera portas 22 (SSH) e 80 (HTTP) |
| **IAM Roles** | Permissão de leitura da EC2 sobre o ECR |
| **AWS CLI** | Autenticação e integração com os serviços AWS |

---

## Estrutura do repositório

```
docker-aws-deploy-lab/
│
├── website/              # Aplicação estática
│   ├── index.html
│   ├── css/style.css
│   └── js/script.js
│
└── referencia/           # Arquivos de containerização
    ├── Dockerfile        # Build da imagem (nginx:alpine + website)
    ├── docker-compose.yml
    └── .dockerignore
```

---

## Como rodar

### Pré-requisitos

| Ferramenta | Instalação |
|---|---|
| Docker | [docs.docker.com](https://docs.docker.com/engine/install/) |
| AWS CLI | [docs.aws.amazon.com/cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) |
| Conta AWS (Free Tier) | [aws.amazon.com](https://aws.amazon.com) |

### 1. Build e teste local

```bash
docker build -t meu-website:v1.0 -f referencia/Dockerfile .
docker run -d -p 8080:80 --name meu-website-container meu-website:v1.0
```

Acesse `http://localhost:8080` para conferir o site rodando localmente.

### 2. Push para o Amazon ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

docker tag meu-website:v1.0 <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
```

### 3. Deploy na EC2

```bash
# Na instância EC2, após instalar o Docker:
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

docker pull <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
docker run -d -p 80:80 --name meu-website-prod --restart always \
  <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/meu-website:v1.0
```

Acesse `http://<IP-PUBLICO-DA-EC2>` para ver o site em produção.

---

## Cenários demonstrados

**Portabilidade via containers** — a mesma imagem Docker roda de forma idêntica no ambiente local e na EC2, eliminando o clássico "funciona na minha máquina".

**Deploy resiliente** — o container é executado com `--restart always`, garantindo que ele volte automaticamente caso a instância reinicie.

**Controle de acesso em camadas** — Security Group restringindo portas expostas e IAM Role concedendo à EC2 apenas a permissão necessária (`AmazonEC2ContainerRegistryReadOnly`) para puxar imagens do ECR.

---

## Troubleshooting

**"Cannot connect to the Docker daemon"**
```bash
sudo systemctl start docker
sudo usermod -a -G docker $USER
```

**Site não abre no navegador** — verifique se a porta 80 está liberada no Security Group, se o container está rodando (`docker ps`) e se o IP público está correto.

**"No basic auth credentials" no pull do ECR** — reautentique o Docker com o ECR:
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ECR_URI>
```

---

## Limpeza de recursos

Para evitar custos na AWS após os testes:

```bash
docker stop meu-website-prod && docker rm meu-website-prod
```

Depois, no console AWS: termine a instância EC2, delete a imagem/repositório do ECR e remova o Security Group criado.

---

## Créditos

Conteúdo e ideia originais de **[Maria Lazara](https://www.youtube.com/@marialazaradev)**, DevOps Engineer, como parte da série *Laboratório DevOps: Aprenda DevOps na Prática com Projetos Progressivos*. Este repositório é a minha execução prática do laboratório proposto por ela.
