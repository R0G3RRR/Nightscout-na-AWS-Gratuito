# 🚀 Nightscout AWS CloudFormation Deploy

Automação completa para implantar o [**Nightscout**](https://github.com/nightscout/cgm-remote-monitor) em uma instância **EC2 da AWS**, utilizando **AWS CloudFormation**.  

O modelo cria automaticamente toda a infraestrutura necessária (VPC, Subnet, Security Group e EC2) ou reutiliza recursos existentes — tudo configurado para ser compartilhado com segurança e simplicidade.

---

## 🧭 Sumário
- [Visão Geral](#-visão-geral)
- [Recursos Criados](#-recursos-criados)
- [Arquitetura](#-arquitetura)
- [Pré-Requisitos](#-pré-requisitos)
- [Criação de Conta no Dynu](#-criação-de-conta-no-dynu)
- [Criação de Conta no MongoDB Atlas](#-criação-de-conta-no-mongodb-atlas)
- [Parâmetros do Template](#-parâmetros-do-template)
- [Arquivos do Projeto](#-arquivos-do-projeto)
- [Como Implantar via Console AWS](#-como-implantar-via-console-aws)
- [Validação e Logs](#-validação-e-logs)
- [Acesso ao Nightscout](#-acesso-ao-nightscout)
- [Boas Práticas de Segurança](#-boas-práticas-de-segurança)
- [Licença](#-licença)
- [Autor](#-autor)

---

## 🌐 Visão Geral

Este projeto automatiza **todo o deploy do Nightscout** em uma instância EC2 da AWS, com:

- **Criação condicional de rede:** Cria VPC/Subnet se não existirem, ou usa as informadas.  
- **Configuração automática de DNS dinâmico (Dynu)** usando o cliente `ddclient`.  
- **Instalação completa do Nightscout via Docker**, incluindo **Certbot configurado** para HTTPS gratuito.  
- **Validação automática** de parâmetros obrigatórios.  
- **Outputs limpos e informativos** ao final do deploy.  

---

## 🏗️ Recursos Criados

| Recurso | Descrição |
|----------|------------|
| **VPC** | Criada automaticamente se não informada. |
| **Subnet Pública** | Criada se a VPC for nova. |
| **Internet Gateway + Route Table** | Configura acesso à internet. |
| **Security Group** | Permite acesso nas portas 22 (SSH), 80 (HTTP) e 443 (HTTPS). |
| **Instância EC2 (Amazon Linux 2023)** | Hospeda o Nightscout via Docker. |
| **Serviço ddclient** | Atualiza o IP público no Dynu automaticamente. |
| **Certbot** | Configura HTTPS gratuito com Let's Encrypt. |
| **Serviço systemd do Nightscout** | Garante que o app inicie com o sistema. |

---

## 🗺️ Arquitetura

```text
┌───────────────────────────────┐
│        AWS CloudFormation     │
│-------------------------------│
│  Cria VPC/Subnet se necessário│
│  Configura Security Group     │
│  Provisiona EC2 (Amazon Linux)│
│  Instala Nightscout + Dynu    │
│  Configura Certbot (HTTPS)    │
└───────────────────────────────┘
             │
             ▼
┌───────────────────────────────┐
│      Instância EC2            │
│-------------------------------│
│ Docker + Nightscout container │
│ ddclient → Dynu DNS dinâmico  │
│ HTTPS automático via Certbot  │
└───────────────────────────────┘
```

---

## ⚙️ Pré-Requisitos

Antes de começar, você precisará:

1. **Conta na AWS** com permissão para criar recursos (EC2, VPC e CloudFormation).  
   👉 [Crie uma conta AWS aqui](https://aws.amazon.com/pt/free/)

2. **Par de chaves EC2 (KeyPair)** existente para acessar a instância se desejar via SSH.  
   - Acesse **EC2 > Par de Chaves** e crie uma chamada `Publica`.

3. **Conta no Dynu** (para gerenciar o domínio dinâmico DDNS).  
4. **Conta no MongoDB Atlas** (banco de dados necessário para o Nightscout).

---

## 🌍 Criação de Conta no Dynu

O [Dynu](https://www.dynu.com/) é o serviço gratuito de DNS dinâmico que manterá seu domínio atualizado com o IP da instância AWS.

### 🪜 Passos:

1. Acesse [https://www.dynu.com/](https://www.dynu.com/)  
2. Clique em **Sign Up** e crie uma conta gratuita.  
3. Após confirmar o e-mail, acesse o painel e vá em:
   ```
   Control Panel → DDNS Services → Add
   ```
4. Escolha um domínio gratuito (exemplo: `seudominio.ddnsfree.com`).  
5. Anote:
   - **Domínio:** `seudominio.ddnsfree.com`
   - **Usuário:** o login criado
   - **Senha:** a senha da conta Dynu  

Essas informações serão usadas como parâmetros no deploy do CloudFormation.

---

## 🍃 Criação de Conta no MongoDB Atlas

O [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) hospeda o banco de dados do Nightscout gratuitamente na nuvem.

### 🪜 Passos:

1. Acesse [https://www.mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)  
2. Crie uma conta gratuita.  
3. Crie um novo **Cluster gratuito** (Plano M0).  
4. Clique em **Database Access** → adicione um novo usuário e senha.  
5. Vá em **Network Access** → adicione o IP `0.0.0.0/0` (para permitir acesso de qualquer lugar).  
6. Copie sua **string de conexão** em:
   ```
   Clusters → Connect → Drivers → Connection String
   ```
   Exemplo:
   ```
   mongodb+srv://usuario:senha@cluster0.xxxxxx.mongodb.net/nightscout
   ```

Essa string será usada no parâmetro `MongoConnection`.

---

## 🧩 Parâmetros do Template

| Parâmetro | Descrição | Obrigatório | Exemplo |
|------------|------------|--------------|----------|
| `VpcId` | ID da VPC existente (deixe vazio para criar nova) | ❌ | `vpc-12345678` |
| `SubnetId` | ID da Subnet existente (deixe vazio para criar nova) | ❌ | `subnet-87654321` |
| `DynuDomain` | Domínio Dynu configurado | ✅ | `seudominio.ddnsfree.com` |
| `DynuLogin` | Login da conta Dynu | ✅ | `meuusuario` |
| `DynuPassword` | Senha da conta Dynu | ✅ | `minhasenhaSegura` |
| `ApiSecret` | API_SECRET do Nightscout | ✅ | `segredo123` |
| `MongoConnection` | String de conexão MongoDB | ✅ | `mongodb+srv://usuario:senha@cluster.mongodb.net/nightscout` |

---

## 📁 Arquivos do Projeto

| Arquivo | Descrição |
|----------|------------|
| **modelo_AW23_param.yaml** | Template principal do CloudFormation. |
| **nightscout-params.json** | Exemplo de parâmetros para quem usar CLI. |
| **README.md** | Este guia completo de uso. |

---

## 🧠 Como Implantar via Console AWS

### 🪜 Passo a Passo

1. Acesse o **Console AWS** → [CloudFormation](https://console.aws.amazon.com/cloudformation/home).  
2. Clique em **Criar pilha** → **Com novos recursos (padrão)**.  
3. Em **Especificar modelo**, selecione **Fazer upload de um arquivo de modelo**.  
4. Faça upload do arquivo `modelo_AW23_param.yaml`.  
5. Clique em **Próximo**.  
6. Dê um nome à pilha (ex: `NightscoutStack`).  
7. Preencha os parâmetros:  
   - DynuDomain  
   - DynuLogin  
   - DynuPassword  
   - ApiSecret  
   - MongoConnection  
   (Deixe VPC/Subnet vazios para criar automaticamente)  
8. Clique em **Próximo** → **Próximo** → marque a opção:
   ```
   Reconheço que o AWS CloudFormation poderá criar recursos IAM
   ```
9. Clique em **Criar Pilha**.

⏳ Aguarde de **5 a 10 minutos** para o processo completar.

---

## 🔍 Validação e Logs

Durante o primeiro boot da instância EC2:

- O script valida automaticamente se todos os parâmetros foram informados.  
- Caso algum esteja vazio, o deploy será abortado com uma mensagem no log.  

### 🔧 Verificar logs de inicialização
Após o deploy, você pode acessar os logs:
1. Vá em **EC2 > Instâncias > Sua instância > Monitoramento > Logs do sistema**.  
2. Procure por mensagens começando com:
   ```
   ❌ Erro: Parâmetro obrigatório ausente. Abortando inicialização.
   ```

---

## 🌐 Acesso ao Nightscout

Após o deploy, vá em:

**CloudFormation → Pilhas → [Sua Pilha] → Aba "Saídas" (Outputs)**

Você verá:
- **InstancePublicIP** → IP público da instância EC2  
- **NightscoutURL** → domínio Dynu configurado  

Acesse pelo navegador:
```
https://seudominio.ddnsfree.com
```

> ⚠️ Pode levar até **2 minutos** após o boot inicial para o Dynu sincronizar o IP.

---

## 🔒 Boas Práticas de Segurança

- Nunca compartilhe senhas reais no repositório (`nightscout-params.json`).  
- O template usa `NoEcho: true` para esconder valores sensíveis.  
- Limite o acesso SSH ao seu IP fixo.  
- Use **chaves SSH** seguras e guarde o `.pem` com cuidado.  
- Faça **backup periódico** do banco MongoDB Atlas.  

---

## 📜 Licença

Este projeto é disponibilizado sob a **licença MIT**.  
Você pode usá-lo, modificá-lo e redistribuí-lo livremente.

---

## ✨ Autor

**Automação e Template por:** Roger  
💬 Contato: [roger.uesc@live.com](mailto:roger.uesc@live.com)  
📦 GitHub: [https://github.com/R0G3RRR/Nightscout-na-AWS-Gratuito](https://github.com/R0G3RRR/Nightscout-na-AWS-Gratuito)

---

> 💡 *Este projeto oferece uma implantação gratuita, automática e segura do Nightscout com HTTPS e DNS dinâmico, sem necessidade de Elastic IP ou custos adicionais.*
