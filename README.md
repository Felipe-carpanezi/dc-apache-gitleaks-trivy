# 🔐 DevSecOps Security Lab — Gitleaks + Trivy + GitHub Actions

Laboratório prático de **DevSecOps** desenvolvido para demonstrar a integração de controles de segurança ao ciclo de desenvolvimento e CI/CD utilizando **Gitleaks, Trivy e GitHub Actions**.

O projeto utiliza uma aplicação web simples executada em **Apache HTTP Server via Docker Compose** como base para demonstrar como ferramentas de segurança podem ser incorporadas ao pipeline de desenvolvimento.

> **Objetivo principal:** demonstrar, de forma prática, onde e como ferramentas de segurança como Gitleaks e Trivy podem ser aplicadas em um projeto de infraestrutura, plataforma ou aplicação.

---

## 🎯 Objetivos

Este laboratório foi criado para estudar e aplicar os seguintes conceitos:

* DevSecOps
* Security as Code
* CI/CD com GitHub Actions
* Secret Scanning
* Vulnerability Scanning
* Infrastructure/Configuration Security
* Security Gates
* Pull Requests como ponto de controle de segurança
* Integração de segurança ao ciclo de desenvolvimento

A proposta não é apenas executar ferramentas de segurança, mas entender **onde elas entram em uma arquitetura real e qual problema cada uma resolve**.

---

# 🏗️ Arquitetura do projeto

```text
                        Developer
                            │
                            │ git push
                            ▼
                       GitHub Repository
                            │
                            │ Pull Request / Push
                            ▼
                    ┌───────────────────┐
                    │   GitHub Actions  │
                    │                   │
                    │  Security Pipeline│
                    └─────────┬─────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
              🔐 Gitleaks           🛡️ Trivy
                    │                   │
              Secret Scan          Security Scan
                    │                   │
                    │          ┌────────┼─────────┐
                    │          │        │         │
                    │        Vuln    Misconfig   Secret
                    │
                    └─────────┬─────────┘
                              │
                              ▼
                       Security Gate
                              │
                       ┌──────┴──────┐
                       │             │
                     FAIL          PASS
                       │             │
                       ▼             ▼
                  Correção       Pull Request
                                      │
                                      ▼
                                    Merge
                                      │
                                      ▼
                                Deploy / GitOps
```

---

# 📁 Estrutura do projeto

```text
dc-apache-gitleaks-trivy/
│
├── .github/
│   └── workflows/
│       └── security.yml
│
├── app/
│   ├── index.html
│   ├── script.js
│   └── style.css
│
├── .gitleaks.toml
├── trivy.yaml
├── docker-compose.yml
├── .gitignore
└── README.md
```

### Principais arquivos

| Arquivo                          | Função                                 |
| -------------------------------- | -------------------------------------- |
| `.github/workflows/security.yml` | Pipeline de segurança                  |
| `.gitleaks.toml`                 | Configuração adicional do Gitleaks     |
| `trivy.yaml`                     | Configuração do Trivy                  |
| `docker-compose.yml`             | Execução local do Apache               |
| `app/`                           | Código da aplicação                    |
| `.gitignore`                     | Arquivos que não devem ser versionados |
| `README.md`                      | Documentação do projeto                |

---

# 🔐 Gitleaks

## O que é?

**Gitleaks** é uma ferramenta de secret scanning utilizada para identificar informações sensíveis expostas em código ou histórico Git.

Entre os exemplos de informações que podem representar risco estão:

* AWS Access Keys
* API Keys
* Tokens
* Senhas
* Credenciais
* Chaves privadas
* Secrets de serviços externos

A ideia é simples:

```text
Código
   │
   ▼
Gitleaks
   │
   ├── Secret encontrado → ❌ Pipeline falha
   │
   └── Nenhum secret      → ✅ Pipeline continua
```

---

## Configuração

O projeto possui um arquivo:

```text
.gitleaks.toml
```

A configuração utiliza as regras padrão do Gitleaks e adiciona uma regra específica para o laboratório.

Exemplo:

```toml
[extend]
useDefault = true

[[rules]]
id = "aws-access-key-id-lab"
description = "AWS Access Key ID - laboratório"
regex = '''AKIA[0-9A-Z]{16}'''
keywords = ["AKIA"]
```

Isso demonstra um conceito importante:

> As regras padrão podem ser utilizadas como base e regras específicas podem ser adicionadas conforme as necessidades da organização.

---

# 🧪 Teste prático realizado com Gitleaks

Para validar o Security Gate, foi criado propositalmente um arquivo `.env` contendo uma **credencial AWS fictícia**:

```text
```

> ⚠️ A credencial utilizada neste laboratório é fictícia e não representa uma credencial real.

O Gitleaks identificou o valor:

```text
Finding:

RuleID:
aws-access-key-id-lab
```

Resultado:

```text
WRN leaks found: 1
```

O comando retornou código de saída diferente de zero, indicando falha no scan.

Isso demonstra como uma ferramenta de segurança pode interromper um pipeline quando uma condição considerada insegura é encontrada.

---

# 🚨 Teste através de Pull Request

O teste também foi realizado através do GitHub Actions.

O fluxo foi:

```text
Secret fictício
      │
      ▼
Commit
      │
      ▼
Pull Request
      │
      ▼
GitHub Actions
      │
      ▼
Gitleaks
      │
      ▼
🚨 Secret detectado
      │
      ▼
❌ Security Check FAILED
```

O GitHub Actions identificou o secret no histórico analisado pelo Pull Request e o check de segurança falhou.

---

# 🧠 Um aprendizado importante sobre Git

Durante o laboratório foi observado um comportamento importante:

> **Apagar um secret em um commit posterior não significa necessariamente que ele desapareceu do histórico Git.**

O cenário foi:

```text
Commit A
   │
   └── Secret fictício introduzido
          │
          ▼
Commit B
   │
   └── Arquivo removido
```

O secret ainda fazia parte do histórico analisado pelo Pull Request.

Foi necessário corrigir o histórico do branch de laboratório utilizando:

```text
git rebase -i
```

e posteriormente:

```text
git push --force-with-lease
```

Após a remoção do commit que continha o secret, o Security Check passou novamente.

### Aprendizado profissional

Em um ambiente real, se uma credencial verdadeira for exposta:

1. **Revogar/rotacionar a credencial imediatamente**
2. Remover o secret do código
3. Corrigir o histórico quando apropriado
4. Investigar possíveis acessos
5. Avaliar o impacto do vazamento

> Reescrever o histórico Git não torna uma credencial real comprometida novamente segura.

---

# 🛡️ Trivy

## O que é?

**Trivy** é uma ferramenta de segurança utilizada para identificar diferentes tipos de problemas em projetos e ambientes de software.

Neste laboratório, o Trivy foi configurado para analisar:

```text
Vulnerabilities
Misconfigurations
Secrets
```

A configuração está em:

```text
trivy.yaml
```

```yaml
scan:
  scanners:
    - vuln
    - misconfig
    - secret

severity:
  - HIGH
  - CRITICAL

exit-code: 1
```

---

## Por que utilizar Trivy?

Enquanto o Gitleaks responde principalmente:

> 🔐 "Existe algum secret exposto?"

O Trivy responde:

> 🛡️ "Existe alguma vulnerabilidade ou configuração insegura?"

Podemos visualizar assim:

| Ferramenta   | Principal objetivo                                 |
| ------------ | -------------------------------------------------- |
| **Gitleaks** | Detectar secrets                                   |
| **Trivy**    | Detectar vulnerabilidades e problemas de segurança |

Eles são complementares.

---

# ⚙️ GitHub Actions

O pipeline de segurança está definido em:

```text
.github/workflows/security.yml
```

O workflow é executado em:

```text
push → main
```

e:

```text
pull_request → main
```

Isso permite que a segurança seja verificada tanto durante alterações no código quanto antes de um Pull Request ser integrado à branch principal.

---

## Pipeline

O pipeline executa:

```text
Checkout
   │
   ├───────────────┐
   ▼               ▼
Gitleaks          Trivy
   │               │
   ▼               ▼
Secrets        Vulnerabilities
                  │
             Misconfigurations
                  │
                  ▼
               Secrets
```

### Gitleaks

```yaml
- name: Gitleaks
  uses: gitleaks/gitleaks-action@v3
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Trivy

```yaml
- name: Trivy
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: fs
    scan-ref: .
    trivy-config: trivy.yaml
```

---

# 🚦 Security Gate

Um dos conceitos mais importantes deste laboratório é o **Security Gate**.

A ideia é simples:

```text
              Código
                 │
                 ▼
          GitHub Actions
                 │
        ┌────────┴────────┐
        ▼                 ▼
     Gitleaks            Trivy
        │                 │
        └────────┬────────┘
                 ▼
          Security Gate
                 │
          ┌──────┴──────┐
          ▼             ▼
        FAIL           PASS
          │             │
          ▼             ▼
       Corrigir       Continuar
```

Um problema de segurança não deve ser tratado apenas como um alerta visual.

O pipeline pode utilizar o resultado da ferramenta para determinar se a alteração pode continuar no processo de entrega.

---

# 🔄 Fluxo DevSecOps

O laboratório demonstra o conceito de inserir segurança **dentro do ciclo de desenvolvimento**, e não apenas depois que a aplicação está em produção.

```text
Developer
    │
    ▼
Código
    │
    ▼
Git
    │
    ▼
Pull Request
    │
    ▼
GitHub Actions
    │
    ├── Gitleaks
    │
    └── Trivy
    │
    ▼
Security Gate
    │
    ├── FAIL → Corrigir
    │
    └── PASS
          │
          ▼
        Merge
          │
          ▼
       Delivery
```

Esse conceito é conhecido como **Shift Left Security**:

> Encontrar problemas de segurança o mais cedo possível no ciclo de desenvolvimento.

---

# ☁️ Relação com uma Internal Developer Platform

Este laboratório foi construído de forma independente, mas os mesmos conceitos podem ser aplicados a uma **Internal Developer Platform (IDP)** baseada em:

```text
AWS
 │
 └── Kubernetes
      │
      ├── Terraform / Terragrunt
      ├── ArgoCD
      ├── Rancher
      ├── Observabilidade
      └── Aplicações
```

Em uma plataforma maior, o pipeline poderia analisar diferentes partes do repositório:

```text
                    IDP Repository
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
   Terraform          Kubernetes         Application
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
                  Security Pipeline
                          │
                  ┌───────┴────────┐
                  ▼                ▼
               Gitleaks          Trivy
                  │                │
                  │          ┌─────┴─────┐
                  │          │           │
                  │        Image        IaC
                  │        Scan         Scan
                  │
                  ▼
             Security Gate
```

Dessa forma, segurança passa a ser uma capacidade da própria plataforma.

---

# 🧩 Onde cada ferramenta se encaixa

Uma forma simples de memorizar:

| Tecnologia             | Responsabilidade                  |
| ---------------------- | --------------------------------- |
| **Git**                | Versionamento                     |
| **GitHub**             | Repositório e colaboração         |
| **GitHub Actions**     | Automação do CI/CD                |
| **Gitleaks**           | Secret scanning                   |
| **Trivy**              | Vulnerability/security scanning   |
| **Terraform**          | Provisionamento de infraestrutura |
| **Kubernetes**         | Orquestração                      |
| **ArgoCD**             | GitOps / Continuous Delivery      |
| **Prometheus/Grafana** | Observabilidade                   |

O objetivo não é substituir uma ferramenta pela outra.

Cada uma possui uma responsabilidade dentro do fluxo.

---

# 🐳 Aplicação

A aplicação utilizada neste laboratório é um site HTML simples executado através do Apache HTTP Server.

O ambiente local pode ser iniciado utilizando:

```bash
docker compose up -d
```

Verificação:

```bash
docker ps
```

A aplicação fica disponível em:

```text
http://localhost:8080
```

O objetivo da aplicação não é ser complexa.

Ela funciona como um **artefato de software** sobre o qual podemos aplicar os controles de segurança do pipeline.

---

# 📚 Principais aprendizados

Durante o desenvolvimento deste laboratório foram praticados conceitos importantes de DevSecOps:

### 1. Secret Scanning

Aprendizado sobre como identificar secrets expostos no código e no histórico Git utilizando Gitleaks.

### 2. Vulnerability Scanning

Entendimento do papel do Trivy na identificação de vulnerabilidades e configurações inseguras.

### 3. Security as Code

As regras e configurações de segurança são armazenadas junto ao código:

```text
.gitleaks.toml
trivy.yaml
.github/workflows/security.yml
```

### 4. CI/CD Security

A segurança é executada automaticamente pelo GitHub Actions.

### 5. Pull Request Security

O Pull Request pode funcionar como um ponto de controle antes da integração do código.

### 6. Security Gate

Falhas de segurança podem interromper o fluxo de entrega.

### 7. Git History

Um secret removido do arquivo ainda pode permanecer no histórico Git.

### 8. Shift Left Security

Problemas devem ser encontrados o mais cedo possível no ciclo de desenvolvimento.

---

# 💼 Aplicabilidade profissional

Este laboratório representa uma versão simplificada de um controle que pode ser incorporado a ambientes corporativos.

Em uma plataforma maior, o mesmo conceito pode ser utilizado para proteger:

* Código de aplicações
* Dockerfiles
* Imagens de containers
* Terraform
* Kubernetes manifests
* Helm charts
* Configurações de infraestrutura
* Repositórios Git
* Pipelines CI/CD

O objetivo é transformar segurança em uma **capacidade automatizada da plataforma**, reduzindo a dependência de verificações manuais.

---

# 🚀 Possíveis evoluções

Este laboratório pode evoluir para uma arquitetura mais completa, incluindo:

* Trivy em Docker Images
* Trivy em Infrastructure as Code
* Terraform security scanning
* Kubernetes security scanning
* Container image scanning
* SBOM
* Dependency scanning
* SAST
* DAST
* Branch Protection / Rulesets
* Security Gates obrigatórios
* ArgoCD
* Kubernetes
* AWS EKS
* Observabilidade
* DevSecOps completo no IDP

Essas evoluções não fazem parte da implementação mínima deste laboratório, mas representam caminhos naturais para ampliar a plataforma.

---

# 🎓 Conclusão

Este projeto demonstra uma implementação prática e simplificada de **DevSecOps**, integrando segurança diretamente ao processo de desenvolvimento.

O principal aprendizado foi entender que ferramentas como Gitleaks e Trivy não precisam ficar isoladas ou ser executadas manualmente sem contexto.

Elas podem fazer parte do próprio fluxo de engenharia:

```text
Code
  ↓
Git
  ↓
Pull Request
  ↓
CI/CD
  ↓
Security Scanning
  ├── Gitleaks
  └── Trivy
  ↓
Security Gate
  ↓
Merge
  ↓
Delivery
```

### Em uma frase:

> **Gitleaks ajuda a impedir que secrets sejam expostos; Trivy ajuda a identificar vulnerabilidades e problemas de segurança; GitHub Actions automatiza esses controles dentro do CI/CD.**

Este laboratório representa a base de uma abordagem **DevSecOps aplicada a uma Internal Developer Platform**, na qual segurança deixa de ser uma etapa manual e passa a fazer parte do processo automatizado de entrega de software.

---

## 👨‍💻 Autor

**Felipe Carpanezi**

DevOps / Cloud / Platform Engineering

Foco de estudos:

* AWS
* Kubernetes
* Terraform
* GitOps
* CI/CD
* DevSecOps
* Observabilidade
* Infrastructure as Code
* Automation

---

## ⭐ Tecnologias utilizadas

```text
Docker
Apache HTTP Server
Git
GitHub
GitHub Actions
Gitleaks
Trivy
YAML
HTML
CSS
JavaScript
```
