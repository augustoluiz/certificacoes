---
titulo: Identity and Federation
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - IAM
  - AWS Organizations
  - IAM Identity Center
  - AWS STS
  - Amazon Cognito
  - AWS Directory Service
criado-em: 2026-08-23
atualizado-em: 2026-08-23
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, identity, federation, security]
---

# Identity and Federation

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## IAM (Identity and Access Management)

- **Usuários na AWS ->** possuem "long term credentials". Isso significa dizer que as credenciais dos usuários não possuem prazo de expiração e, uma vez criadas, permanecem válidas até que ocora a sua desativação;
- É possível criar agrupamentos para os usuários; 
- **Roles** são consideradas "short term credentials" e podem ser usadas por meio do serviço AWS Securit Service Token (AWS STS):
  - Justamente por serem utilizadas por meio do "assume role" via STS, as roles são temporárias (duram enquanto a operação realizar o assume role)
  - **Exemplos de roles:**
    - EC2 Instance Roles;
    - Service Roles (API Gateway, DynamoDB, S3);
    - Cross Account roles;
- **Policies ->** Servem para definir o que as roles são capazes de fazer. Elas permitem os níveis de acesso que uma role terá. Podem ser:
  - **AWS Managed:** Políticas gerenciadas pela própria AWS e servem para garantir que os usuários não façam nada além do permitido;
  - **Customer Managed:** Políticas gerenciadas pelo próprio cliente. São criadas e podem ser compartilhadas entre roles;
  - **Inline Policies:** Políticas "de linha" que estão associadas diretamenta a uma role em específico.

### Deep Dive em IAM Policies

Uma policy do IAM possui a estrutura de um JSON com os seguintes atributos:

  - **Effect:** Representa o efeito que aquela policy terá (permissiva ou restritiva). Marcado pelos valores "Allow" ou "Deny";
  - **Action:** Detalhas as ações que sofreção o efeito de allow ou deny (ex: ec2:AttachVolume);
  - **Resource:** Recurso(s) que está sendo impactado pela ação da policy (ex: arn de um serviço);
  - **Conditions:** Serve para filtrar ou criar condicionantes à aplicação da policy;
  - **Policy Variables:** Permite a atribuição de valores dinâmicos à policy por meio de variáveis.

> **Obs¹:** Uma boa prática na concepção de policies é o princípio do **least privilege** (menor privilégio), onde liberamos somente o que é necessário.

> **Obs²:** As actions de Deny possuem maior prioridade frente às actions de allow. Isso significa que o Deny sempre será atribuído primeiro. Dessa forma, cenários em que podem haver conflitos entre Deny e Allow devem ser analisados no detalhe de modo que seja escrita da maneira lógica que funcione.

**Funcionalidades Compĺementares à Policy**

Dentro do IAM podemos encontrar duas funcionalidades que nos ajudam no processo de análise e validação das permissões de roles e policies. São estas:

  - **AWS IAM Access Advisor:** Ferramenta de segurança da Amazon que permite analisar quais serviços um usuário, grupo, role ou policy têm acessos e quando foram utilizados pela última vez.
  - **AWS IAM Access Analyser:** Ferramenta de segurança que ajuda a definir, verificar e refinar permissões com base no princípio do menor privilégio

**IAM Policies Conditions**

As condicionantes nas policies possuem a seguinte estrutua:

```json
"Condition": {
    "{condition-operator}": {
        "{condition-key}": "{condition-value}"
      }
  }
```

Os operadores podem ser do seguinte tipo:

  - **String:** StringEquals, StringNotEquals, StringLike...
  - **Numeric:** NumericEquals, NumericNotEquals, NumericLessThan...
  - **Date:** DateEquals, DateNotEquals, DateLessThan...
  - **Boolean:** Bool (True or False).
  - **(Not)IpAddress:** Referência de Ip's.
  - **Referência de ARNs:** ArnEquals, ArnLike...
  - **Null:** É o operador que testa existência de chave — não o valor dela. É o único que faz isso diretamente.

### IAM Roles vs. Resource-Based Policies

Existem **dois caminhos** para conceder acesso a um recurso, sobretudo em cenários *cross-account*. A escolha entre eles é um dos temas mais cobrados na prova.

A diferença fundamental está em **o que acontece com a identidade do principal**:

  - **IAM Role (assume role):** o principal **abandona** temporariamente suas permissões originais e passa a operar com as permissões da role. É uma **troca de identidade** — durante a sessão, ele "vira" outra entidade;
  - **Resource-Based Policy:** o principal **mantém** suas próprias permissões e, adicionalmente, ganha acesso ao recurso. **Não há troca de identidade** — ele continua sendo quem era.

| | IAM Role | Resource-Based Policy |
|---|---|---|
| Onde é anexada | À identidade (usuário, grupo, role) | Ao próprio recurso (bucket, fila, função...) |
| Elemento `Principal` | Não possui (é implícito) | **Obrigatório** (define quem pode acessar) |
| Identidade do chamador | Substituída pela da role | Preservada |
| Chamada necessária | `sts:AssumeRole` | Nenhuma (acesso direto) |
| Suporte | Todos os serviços | Apenas alguns serviços |

**Por que a preservação da identidade importa**

Considere uma Lambda na **Conta A** que precisa ler um objeto no S3 da **Conta B** e, no mesmo processamento, gravar no DynamoDB da **Conta A**:

  - **Com assume role:** ao assumir a role da Conta B, a função perde as permissões da Conta A e não consegue mais gravar no DynamoDB. Seria necessário alternar entre as credenciais;
  - **Com resource-based policy:** o bucket da Conta B autoriza diretamente a role da Lambda. A função acessa os dois recursos na mesma execução, sem trocar de credencial.

> **Decisão:** quando o workload precisa acessar recursos de **duas contas simultaneamente**, a resource-based policy é a resposta. O assume role serve quando a operação ocorre inteiramente dentro da conta de destino.

**Regra de avaliação**

  - **Mesma conta:** basta que **uma** das políticas (identity-based **ou** resource-based) conceda o Allow;
  - **Cross-account:** é necessário Allow nos **dois lados** — na identity-based policy do principal (Conta A) **e** na resource-based policy do recurso (Conta B). Um lado sozinho não é suficiente.

**Serviços que suportam resource-based policies**

S3 (bucket policy), SQS, SNS, Lambda, KMS (key policy), Secrets Manager, EFS, ECR, API Gateway, CloudWatch Logs, EventBridge, SES, Backup Vault, CodeArtifact e VPC Endpoints (endpoint policy). O DynamoDB passou a suportar a partir de 2024.

Serviços como **EC2 e RDS não possuem** resource-based policies — nesses casos, o acesso cross-account só é possível via assume role.

> **Pegadinha:** a **trust policy** de uma role é, tecnicamente, uma resource-based policy — é ela que possui o elemento `Principal` e define **quem pode assumir** a role. Não confundir com a permissions policy, que define **o que a role pode fazer**.

> **Pegadinha:** no **KMS**, a key policy é obrigatória e tem precedência. Uma IAM policy concedendo `kms:Decrypt` **não funciona sozinha** se a key policy não delegar essa autoridade à conta (via `"Principal": {"AWS": "arn:aws:iam::<account-id>:root"}`).

## IAM Access Analyzer

Serviço dentro do IAM Console que permite localizar quais recursos estão sendo compartilhados externamente. Definimos as "Zones of Trust" que podem ser aplicadas às AWS Accounts ou AWS Organizations. Quaisquer acessos fora dessas zonas de segurança serão apontados como "findings".

Além disso, essa funcionalidade também é capaz de realizar a validação de policies e também gerar policies com base nos logs que são gerados via CloudTrail. Por exemplo: podemos ter uma lambda que acessa um bucket s3 e realiza chamada ao kinesis. O IAM Access Analyzer Policy Generation então é capaz de analisar os logs do cloudtrail nos últimos 90 dias para gerar uma policy com base nos acessos que a lambda executa.

## AWS Security Token Service (STS)

Serviço da AWS que permite com que seja feita o assume role entre diferentes contas, mesmas contas e também realizar o processo de Identity Federation. O proceso de utilização do STS para assume role pode ser descrito da seguinte maneira:

- Definição de uma IAM Role para acesso na mesma conta ou via cross-account;
- Definição de quais principals podem acessar essa role;
- Uso do AWS STS para realizar o retrieve de credenciais para realização do assume role;
- O grant de credenciais temporárias pode durar entre 15 minutos até 12 horas.

*Obs: O processo para efetuar o uso da role é feito via "AssumeRole API", funcionalidade a qual interage com o STS. O STS por sua vez realiza a validação se as permissões para assumir aquela role são válidas. Um vez válida, retorna as credenciais de segurança temporárias.

![Diagrama STS Assume Role](./img/sts/sts-01.png)

### Cenários em que podemos utilizar o Assume Role via STS

- Providenciar o acesso de um usuário IAM em uma conta AWS aos recursos de em outra conta;
- Providenciar o acesso de usuários terceiros à conta AWS;
- Liberar acesso aos serviços da AWS para recursos da prórpria AWS.
- Providenciar acesso para usuários externos autenticados (identify federation)

## Identity Federation & Cognito

- Principais conceitos: 
  - Faz com que usuários de fora da AWS tenham acesso aos recursos na sua conta.
  - Não é preciso criar IAM Users (usuários são gerenciados fora da AWS)

- Casos de uso:
  - Uma empresa possui um provedor de identidade externo (ex: Microsoft EntraID)
  - Aplicações Web/Mobile que necessitam acessar recursos da conta AWS

### Fluxo do Identity Federation

O processo de acesso aos recursos em uma conta AWS por meio de um provedor de identidade externo ocorre da seguinte maneira:

1. O Provedor de Identidade (ex: AD, EntraID) realiza o "trust relationship" com a AWS;
2. Usuários efetuam o Login no provedor de identidade;
3. O provedor retorna as credenciais temporárias de acesso aos usuários;
4. Os usuários utilizam as credenciais temporárias fornecidas para realizar o acesso à AWS

![Diagrama Fluxo Identity Federation](./img/identity-federation/identity-federation-01.png) 

A comunicação do Identity Federation pode ser realizada por meio de:

- SAML 2.0;
- Customer Identity Broker;
- Web Identity Federation With(out) Amazon Cognito;
- IAM Identity Center

#### Conexão via SAMl 2.0

SAML 2.0 (Security Assertion Markup Language) é um padrão aberto, baseado em XML, para trocar dados de autenticação e autorização entre duas partes. É o protocolo que viabiliza o SSO (Single Sign-On) corporativo: o usuário faz login uma vez no IdP da empresa e acessa vários sistemas (AWS, Salesforce, Office 365...) sem digitar a senha de novo. Foi publicado pela OASIS em 2005.

Este padrão permite aplicar o Identity Federation com diversos IdP's que possuam o suporte ao SAML 2.0. O processo é representado pelas seguintes etapas:

1. Usuários realizam a autenticação no IdP;
2. O IdP realiza a autenticação no LDAP-based Identity Store;
3. É retornado o SAML Assertion ao usuário;
4. O usuário realiza o **AssumeRoleWithSAML** no AWS Security Service Token (STS)
5. O STS retorna as credenciais de acesso temporário ao usuário;
6. O usuário realiza o acesso aos recuros/console da conta por meio das credenciais;

![Diagrama Fluxo Identity Federation - SAML 2.0](./img/identity-federation/identity-federation-02.png) 

### Usando o Cognito

É possível realizar a integração do Identity Federation na AWS com e sem o Cognito.

> **Mas atenção!** Atualmente a AWS **não** recomenda mais o cenário sem o Cognito.

A opção antiga (não recomendada) sem o Cognito consiste basicamente na utilização de um 3ʳᵈ Party Identity Provider como, por exemplo, Google, Facebook ou qualquer OpenID Connect compatível com IdP.

![Diagrama Fluxo Identity Federation - Without Cognito](./img/identity-federation/identity-federation-03.png) 

Para o cenário recomendado, a diferença principal está na utilização do Cognito como intermediador após o processo de autenticação no 3ʳᵈ Party Identity Provider. O Cognito é capaz de criar uma camada de abstração e segurança entre o dispositivo móvel/web e os recursos na AWS. Sua principal função é gear um token que deve ser utilizado em conjunto com o STS.

![Diagrama Fluxo Identity Federation - With Cognito](./img/identity-federation/identity-federation-04.png) 

## AWS Directory Services

Este serviço está fortemente relacionado com o serviço fornecido pela Microsoft chamado "Microsoft Active Directory" ou popularmente conhecido como AD. Em resumo, o AD é:

- Encontrado em qualquer Windows Server que possua o AD Domain Services;
- Um banco de dados de "objetos" que podem representar usuários, contas, computadores, impressoras, File Shares, Security Groups, etc;
- Uma forma de realizar a gestão de segurança centralizada, criação de contas, associação de políticas e permissões, etc;
- Os objetos são organizados em "trees" e o conjunto de "trees" é chamado de "forest".

A gestão dos objetos dentro do AD é realizada por meio do Domain Controller, cuja responsabilidade é ser o servidor que roda o AD DS (Active Directory Domain Services). Ele guarda a base de identidades do domínio e responde por toda autenticação e autorização dentro dele. Na prática, "o AD" que os clientes e aplicações acessam é um ou mais DCs.

![Diagrama Microsoft Active Directory](./img/aws-directory-services/aws-directory-services-01.png) 

### AD Federation Service (ADFS)

O ADFS é o serviço da microsoft que permite a integração Sigle Sign-On entre aplicações e o Active Directory. Além disso, também permite realizar a integração via SAML entre sistemas terceiros (ex: AWS, Dropbox, Office365, etc).

![Diagrama AD Federation Service](./img/aws-directory-services/aws-directory-services-02.png) 

### Tipos de AWS Directory Services

O AWS Directory Service tem três tipos principais: **AWS Managed Microsoft AD, AD Connector e Simple AD.**

#### AWS Managed Microsoft AD

Um **Active Directory real (Windows Server)** gerenciado pela AWS.

- Provisiona **2 Domain Controllers em AZs diferentes**, e você pode adicionar mais. A AWS cuida de patching, backup, replicação e monitoramento.
- Tem duas edições:
  - **Standard:** até cerca de 30 mil objetos.
  - **Enterprise:** até cerca de 500 mil objetos. Suporta **multi-Region replication.**
- Suporta **trust** (one-way ou two-way, forest ou external) com o **AD on-premises**. Os usuários continuam on-prem e acessam recursos na AWS.
- Suporta MFA via **RADIUS, LDAPS**, extensões de schema e **compartilhamento do diretório com outras contas**.
- Você **não tem Domain Admin**. Recebe um grupo de administração delegada.
- Integrações: **RDS for SQL Server, FSx for Windows, WorkSpaces, IAM Identity Center**, QuickSight, seamless domain join de EC2 etc.

**Quando usar:** você precisa de um AD de verdade na AWS, quer trust com o AD on-prem ou quer que a autenticação **funcione mesmo se o link com o datacenter cair.**

![Diagrama AWS  Managed Microsoft AD](./img/aws-directory-services/aws-directory-services-03.png) 

**Integrações Possíveis no AWS Managed Microsoft AD**

![Diagrama AWS  Managed Microsoft AD - Integrações](./img/aws-directory-services/aws-directory-services-04.png)

**Conexão com On Premises AD**: A conexão com o AD que se encontra na rede interna (On Premises) de uma empresa com o AWS Managed Microsoft AD pode ocorrer por meio de duas formas:

- VPN Site-to-Site;
- AWS Direct Connect.

Neste processo, temos 3 tipos de trust's possíveis:

- **One-way trust:** AWS -> On-Premise
- **One-way trust:** On-Premise -> AWS
- **Two-way trust:** AWS <-> On-Premise

![Diagrama AWS  Managed Microsoft AD - Conectividade com On-Premise](./img/aws-directory-services/aws-directory-services-05.png)

#### AD Connector

Um **proxy** (gateway) que redireciona as requisições para o **AD on-premises existente.**

- **Não armazena nem faz cache** de nada na AWS. Toda autenticação vai para os DCs on-prem.
- Exige **VPN ou Direct Connect**. Se o link cair, a autenticação para.
- Tamanhos: **Small e Large**, que definem a capacidade de requisições e não um limite rígido de usuários.
- Suporta MFA via **RADIUS** on-prem.
- **Não suporta trust** e não é um diretório "de verdade".
- Integrações: **IAM Identity Center, WorkSpaces**, console da AWS, seamless domain join de EC2. **Não funciona com RDS for SQL Server**, que exige Managed AD.

**Quando usar:** os usuários devem continuar **só no AD on-prem**, sem sincronizar ou replicar nada para a AWS, e com **o menor esforço possível.**

![Diagrama AWS AD Conector](./img/aws-directory-services/aws-directory-services-06.png)

#### Simple AD

Um **diretório leve e barato, compatível com AD**, baseado em **Samba 4.**

- Tamanhos: **Small** (até cerca de 500 usuários) e **Large** (até cerca de 5 mil usuários).
- Suporta usuários, grupos, Kerberos, GPOs básicas e domain join de EC2.
- **Não suporta** trust com outros domínios, **MFA**, extensões de schema e várias integrações (RDS SQL Server e IAM Identity Center, por exemplo).

**Quando usar:** um ambiente pequeno, **standalone** na AWS, que precisa de recursos básicos de AD e onde o **custo** é o principal critério.