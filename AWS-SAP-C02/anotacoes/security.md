---
titulo: Security
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - CloudTrail
  - KMS
  - SSM Parameter Store
  - Secrets Manager
criado-em: 2026-10-22
atualizado-em: 2026-10-28
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, security]
---

# Security

## CloudTrail

O AWS CloudTrail fornece **governança, auditoria e compliance** para a sua conta AWS. Ele vem **habilitado por padrão** em todas as contas e registra o histórico de **eventos e chamadas de API** feitas na conta, a partir de origens como:

- Console
- AWS SDK
- CLI
- Os próprios serviços da AWS (ex: um Auto Scaling Group que lança instâncias EC2)

Cada evento responde às perguntas:

- **Quem** fez? (user, role, serviço, conta)
- **O quê?** (ação/API chamada e parâmetros)
- **Quando?** (data e hora)
- **De onde?** (IP de origem, user agent, região)
- **Qual foi o resultado?** (sucesso ou código de erro)

**Casos de uso:** auditoria e compliance, investigação de incidentes de segurança ("quem apagou este recurso?") e troubleshooting operacional.

Os eventos podem ser enviados para o **S3** e/ou para o **CloudWatch Logs** por meio de um **trail**. Um trail pode ser aplicado a **todas as regiões** (configuração padrão) ou a **uma única região**.

![Diagrama AWS CloudTrail](./img/aws-cloudtrail/aws-cloudtrail-01.png)

### CloudTrail Events

Existem 4 tipos de eventos no CloudTrail:

| Tipo | O que registra | Habilitado por padrão? |
|---|---|---|
| **Management Events** | Operações realizadas sobre os **recursos** da conta, no **control plane** (ex: `CreateBucket`, `RunInstances`, `AttachRolePolicy`, `ConsoleLogin`) | ✅ |
| **Data Events** | Operações de **data plane**, de alto volume (ex: atividade em nível de objeto no S3, como `GetObject`, `PutObject` e `DeleteObject`; `Invoke` no Lambda; operações de item no DynamoDB) | ❌ (custo adicional) |
| **Network Activity Events** | Chamadas de API feitas **através de VPC Endpoints** (inclusive as negadas pela endpoint policy) | ❌ |
| **CloudTrail Insights Events** | **Comportamento anômalo**: picos no volume de chamadas de API de escrita ou na taxa de erros, comparados com o histórico da conta | ❌ (custo adicional) |

- **Management Events** e **Data Events** podem ser separados em eventos de **leitura (Read Events)** e de **escrita (Write Events)**. Isso permite registrar só o que interessa (ex: apenas as escritas) e reduzir volume e custo.
- Os **Data Events** não são registrados por padrão justamente pelo **alto volume** de operações. Eles podem ser filtrados com **advanced event selectors** (ex: registrar apenas os eventos de escrita de um bucket específico).
- O **CloudTrail Insights** analisa continuamente os management events de escrita para estabelecer uma **linha de base** do comportamento normal da conta. Quando detecta algo fora do padrão (ex: um pico de `TerminateInstances` ou de erros `AccessDenied`), gera um **Insights event**. Esses eventos podem ser entregues ao **S3** e ao **EventBridge**, o que permite criar **fluxos e automações** a partir deles (ex: notificar o time de segurança via SNS).

![Diagrama AWS CloudTrail Insights](./img/aws-cloudtrail/aws-cloudtrail-02.png)

### CloudTrail Events Retention

Por padrão, os eventos ficam disponíveis no **Event History** do CloudTrail por **90 dias**. Para manter um histórico maior que esse período, é preciso criar um **trail** que armazene os eventos no **S3**.

- **Event History:** habilitado automaticamente em todas as contas, sem configuração. Mostra os **management events dos últimos 90 dias**, por região. Não inclui data events nem insights.
- **Trail:** configuração que **entrega continuamente** os eventos para um **bucket S3** (e, opcionalmente, para o **CloudWatch Logs**). É necessário para **reter os logs por mais de 90 dias** e para registrar data events e insights.

> **Obs:** No S3, é possível aplicar **lifecycle policies** para mover os logs antigos para classes mais baratas (ex: S3 Glacier) e consultá-los com o **Athena** quando necessário. Outra opção para retenção longa com consultas SQL é o **CloudTrail Lake**.

### Tipos de trail

- **Multi-Region:** registra os eventos de **todas as regiões** (inclusive as que forem habilitadas no futuro). É o padrão e a boa prática.
- **Single-Region:** registra os eventos de uma única região.
- **Organization trail:** criado na **management account** (ou na conta de delegated administrator) e aplicado a **todas as contas da organização**. As member accounts **conseguem ver** o trail, mas **não conseguem alterá-lo ou excluí-lo**.

> **Obs:** Os serviços globais (ex: IAM, STS, CloudFront) registram seus eventos em **us-east-1**. Um trail multi-region captura esses eventos automaticamente.

**Custos:** a **primeira cópia** dos management events entregue por um trail é gratuita. Cópias adicionais (outros trails registrando os mesmos management events), data events, network activity events e insights são cobrados.

### Segurança dos logs

Como os logs do CloudTrail são a base da auditoria, eles também precisam ser protegidos:

- **Criptografia:** por padrão os logs são criptografados com **SSE-S3**. É possível usar **SSE-KMS** com uma chave gerenciada pelo cliente, para controlar quem pode ler os logs.
- **Log file integrity validation:** o CloudTrail gera, a cada hora, um **digest file** com hashes (SHA-256) dos arquivos de log, assinado digitalmente (RSA). Com ele é possível provar que os logs **não foram alterados ou excluídos** após a entrega (`aws cloudtrail validate-logs`).
- **Proteção do bucket:** centralizar os logs em uma **conta separada** (ex: Log Archive do Control Tower), restringir a bucket policy, habilitar **S3 Object Lock** (WORM) e/ou **MFA Delete**.
- **Impedir a desativação:** usar uma **SCP** que bloqueie `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail` e `cloudtrail:UpdateTrail` nas member accounts.

> **Obs:** A entrega dos logs no S3 **não é em tempo real**. Os eventos costumam ser entregues em até cerca de **15 minutos** após a chamada da API.

### Integrações e análise dos logs

- **Amazon EventBridge:** recebe os eventos do CloudTrail e permite **reagir em tempo quase real**. Exemplo: quando alguém chamar `DeleteTrail` ou criar um IAM user, disparar uma Lambda de remediação ou uma notificação SNS.
- **CloudWatch Logs:** o trail pode enviar os eventos para um log group. Com **metric filters** + **alarms** é possível alertar sobre eventos específicos (ex: login do root user, falhas de `ConsoleLogin`, alterações em security groups).
- **Amazon Athena:** consultas SQL diretamente sobre os logs armazenados no S3. Uma opção barata para investigações pontuais.
- **CloudTrail Lake:** data lake gerenciado que armazena os eventos em **event data stores** (com retenção configurável por anos) e permite consultas **SQL** direto no CloudTrail, sem precisar montar a estrutura com S3 + Athena. Pode também receber eventos de fontes **fora da AWS**.

> **Obs:** Por padrão, as regras do EventBridge recebem os eventos de **escrita**. Para receber também os eventos de leitura (`Get*`, `List*`, `Describe*`), é preciso habilitar isso explicitamente na regra.

### CloudTrail vs. Config vs. CloudWatch

| Serviço | Responde | Exemplo |
|---|---|---|
| **CloudTrail** | **Quem fez o quê** e quando (chamadas de API) | "Quem alterou o security group às 14h?" |
| **AWS Config** | **Como está (e como estava)** a configuração dos recursos e se ela está em conformidade | "Este security group permite 0.0.0.0/0 na porta 22? Desde quando?" |
| **CloudWatch** | **Como está o desempenho** (métricas, logs de aplicação, alarmes) | "A CPU da instância passou de 80%?" |

### Dicas para a prova

- "**Quem** apagou/alterou um recurso?" → **CloudTrail**.
- "Auditar a atividade de **todas as contas** da organização de forma centralizada" → **Organization trail** entregando em um bucket de uma **conta de log separada**.
- "Garantir que os logs **não foram adulterados**" → **log file integrity validation** (+ S3 Object Lock).
- "**Reagir automaticamente** a uma chamada de API específica" → **CloudTrail + EventBridge** (+ Lambda/SNS).
- "Detectar **volume anormal** de chamadas de API ou de erros" → **CloudTrail Insights**.
- "Auditar **quem acessou objetos** de um bucket S3" → **data events** do CloudTrail (ou S3 server access logs).
- "Reter logs de auditoria por **anos**" → trail para o **S3** (com lifecycle para Glacier) ou **CloudTrail Lake**. O Event History guarda apenas **90 dias**.
- "Fazer consultas **SQL** nos eventos" → **Athena** sobre o S3 ou **CloudTrail Lake**.
- "Impedir que as member accounts desativem o CloudTrail" → **Organization trail** e/ou **SCP**.


## AWS KMS (Key Management Service)

O AWS Key Management Service (KMS) é o serviço gerenciado para **criar e controlar as chaves criptográficas** usadas para proteger os dados. Ele é integrado à grande maioria dos serviços da AWS (S3, EBS, RDS, DynamoDB, Secrets Manager, CloudTrail etc.), o que torna a criptografia de dados em repouso praticamente transparente.

- As chaves ficam armazenadas em **HSMs** (*Hardware Security Modules*) gerenciados pela AWS e validados pelo **FIPS 140-3**. O material das chaves **nunca sai do KMS sem criptografia**.
- O acesso às chaves é controlado por **key policies** e **IAM policies**.
- Todas as chamadas de API ao KMS (ex: `Encrypt`, `Decrypt`, `GenerateDataKey`) são **registradas no CloudTrail**, o que permite auditar quem usou cada chave.
- As chaves são **regionais**: uma chave existe apenas na região em que foi criada (exceto as multi-region keys).

### Tipos de chave

**Por tipo de criptografia:**

| Tipo | Como funciona | Casos de uso |
|---|---|---|
| **Symmetric** (padrão) | Uma única chave **AES-256** para criptografar e descriptografar. A chave nunca sai do KMS: é preciso chamar a API do KMS para usá-la | Criptografia de dados em repouso nos serviços AWS. É o tipo usado pela maioria das integrações |
| **Asymmetric** | Par de chaves **RSA** ou **ECC**. A **chave pública pode ser baixada**, e a privada nunca sai do KMS | Criptografia por quem **não pode chamar a API do KMS** (ex: clientes fora da AWS), **assinatura digital** e verificação |
| **HMAC** | Chave usada para gerar e verificar códigos HMAC | Verificar integridade e autenticidade de mensagens e tokens |

**Por propriedade/gerenciamento:**

| Tipo | Quem gerencia | Rotação | Custo |
|---|---|---|---|
| **AWS owned keys** | A AWS, usadas internamente em várias contas. Não aparecem na sua conta | Gerenciada pela AWS | Gratuita |
| **AWS managed keys** | A AWS, mas **ficam na sua conta** (ex: `aws/s3`, `aws/ebs`, `aws/rds`). A key policy **não pode ser alterada** | **Automática e obrigatória** (anual) | Sem custo mensal (paga-se pelas chamadas de API) |
| **Customer managed keys** | **Você**: cria, define a key policy, habilita/desabilita e agenda a exclusão | **Opcional**: automática (período configurável de 90 a 2560 dias, padrão de 1 ano) ou **on-demand** | **US$ 1/mês** por chave + chamadas de API |

> **Obs:** Na rotação automática, a chave mantém o **mesmo key ID e ARN**. O KMS guarda o material das versões anteriores para continuar **descriptografando** os dados antigos, e usa o material novo para as novas criptografias. Não é preciso recriptografar nada.

**Imported key material (BYOK):** é possível criar uma customer managed key **sem material** e **importar o seu próprio material** de chave, gerado fora da AWS.
- Você é responsável pela **durabilidade** do material (deve manter uma cópia fora da AWS) e pode definir uma **data de expiração**.
- **Rotação manual:** cria-se uma nova chave com o novo material e atualiza-se o **alias** para apontar para ela.

> **Obs:** Um **alias** (ex: `alias/app-prod`) é um nome amigável que aponta para uma chave. As aplicações devem referenciar o alias, e não o key ID. Assim, a troca da chave (rotação manual) não exige mudança no código.

### Envelope Encryption

A API `Encrypt` do KMS criptografa diretamente no máximo **4 KB** de dados. Para dados maiores, usa-se o **envelope encryption**:

1. A aplicação chama `GenerateDataKey`. O KMS retorna uma **data key** (DEK) em duas versões: **em texto claro** e **criptografada** pela KMS key.
2. A aplicação **criptografa os dados localmente** com a data key em texto claro e a descarta da memória.
3. A aplicação armazena a **data key criptografada junto com os dados** (o "envelope").
4. Para descriptografar, a aplicação envia a data key criptografada ao KMS (`Decrypt`), recebe a data key em texto claro e descriptografa os dados localmente.

Benefícios: os dados **não trafegam até o KMS** (apenas a data key), há melhor desempenho e a KMS key nunca sai do KMS. É assim que os serviços AWS (S3, EBS etc.) usam o KMS por baixo.

> **Obs:** O **AWS Encryption SDK** implementa o envelope encryption e oferece **data key caching**, que reutiliza data keys por um período e **reduz o número de chamadas** ao KMS.

### Key Policies e controle de acesso

Toda KMS key tem **exatamente uma key policy**, que é a principal forma de controlar o acesso a ela. **Sem permissão na key policy, nenhum principal consegue usar a chave**, nem mesmo o root user da conta.

- **Default key policy:** concede acesso total à **conta** (`arn:aws:iam::<account-id>:root`). Isso **habilita o uso de IAM policies** para controlar o acesso à chave dentro da conta.
- **Custom key policy:** define explicitamente quem administra a chave (*key administrators*) e quem pode usá-la (*key users*). É necessária para **acesso cross-account**.
- **Grants:** permissões **temporárias e programáticas** para usar uma chave, muito usadas pelos serviços AWS. Exemplo: o EBS cria um grant para que o EC2 descriptografe um volume.

**Condições úteis:**
- `kms:ViaService`: permite usar a chave **somente por meio de um serviço específico** (ex: só via S3).
- `kms:EncryptionContext`: o **encryption context** é um conjunto de pares chave-valor não secretos enviados na criptografia. Ele precisa ser **idêntico na descriptografia** e aparece no CloudTrail, o que ajuda na auditoria e no controle de acesso.
- `kms:CallerAccount`: restringe o uso da chave a uma conta específica.

**Acesso cross-account** exige duas configurações:
1. A **key policy** da chave (conta A) deve permitir o principal da conta B.
2. Uma **IAM policy** na conta B deve permitir que o principal use a chave da conta A.

### Compartilhamento de recursos criptografados

**Entre contas (ex: snapshot EBS ou RDS):**
1. O snapshot precisa estar criptografado com uma **customer managed key**. Snapshots criptografados com **AWS managed keys não podem ser compartilhados** com outras contas.
2. Na conta de origem, compartilhar o snapshot e **liberar a chave na key policy** para a conta de destino.
3. Na conta de destino, **copiar o snapshot**, recriptografando-o com uma chave **própria**.

**Entre regiões:** como as chaves são regionais, ao **copiar um snapshot ou AMI criptografado para outra região** é preciso informar uma **chave da região de destino**. A AWS recriptografa os dados durante a cópia.

### Multi-Region Keys

São chaves com o **mesmo key ID e o mesmo material** replicadas em várias regiões. Assim, um dado criptografado em uma região pode ser **descriptografado em outra sem chamadas cross-region** e sem recriptografia.

- Existe uma chave **primary** e uma ou mais **replicas**. Cada uma é um recurso **independente**, com **sua própria key policy**.
- A rotação automática é feita na primary e **propagada** para as replicas.
- **Não são chaves globais**: cada réplica é gerenciada separadamente em sua região.

Casos de uso:
- **Client-side encryption** de dados replicados entre regiões (ex: atributos criptografados em **DynamoDB Global Tables** ou **Aurora Global Database**).
- **Disaster recovery** e aplicações **ativo-ativo** multi-região.
- **Assinatura digital** com chaves assimétricas, verificada em várias regiões.

> **Obs:** A AWS **não recomenda** usar multi-region keys por padrão. Elas devem ser usadas apenas quando houver real necessidade de descriptografar os mesmos dados em várias regiões, já que ampliam a superfície de exposição da chave.

### Exclusão de chaves

- Uma chave **não é excluída imediatamente**: é preciso agendar a exclusão com um **período de espera de 7 a 30 dias** (padrão de 30). Durante esse período, a chave fica **inutilizável** e a exclusão pode ser **cancelada**.
- Depois de excluída, **todos os dados criptografados com ela ficam irrecuperáveis**.
- Alternativa mais segura: **desabilitar** a chave, o que é reversível.

> **Boa prática:** criar um alarme (CloudTrail + EventBridge ou CloudWatch metric filter) que dispare se alguém **tentar usar uma chave com exclusão agendada**. Isso indica que a chave ainda está em uso e a exclusão deve ser cancelada.

### Limites de requisições

O KMS tem **quotas de requisições por segundo** (compartilhadas entre as operações criptográficas, por conta e região). Ao ultrapassá-las, as chamadas recebem `ThrottlingException`.

Soluções:
- **Exponential backoff** nas chamadas.
- **Data key caching** (AWS Encryption SDK).
- **S3 Bucket Keys:** para SSE-KMS no S3, gera uma chave no nível do bucket e **reduz as chamadas ao KMS em até 99%**, o que também diminui o custo.
- Solicitar **aumento da quota** via Service Quotas.

### Custom Key Stores

Para requisitos de compliance que exigem **controle total do hardware** onde as chaves ficam:

- **AWS CloudHSM key store:** as chaves do KMS ficam em um **cluster CloudHSM** que você controla, mas continuam sendo usadas pela API do KMS e pelas integrações com os serviços AWS.
- **External key store (XKS):** as chaves ficam em um **HSM fora da AWS** (ex: on-premises). O KMS chama o seu HSM externo a cada operação. Usado em requisitos regulatórios do tipo "*hold your own key*".

### KMS vs. CloudHSM

| | AWS KMS | AWS CloudHSM |
|---|---|---|
| Modelo | Serviço gerenciado, **multi-tenant** | HSM **dedicado** (single-tenant) na sua VPC |
| Controle das chaves | AWS gerencia o hardware, você controla o acesso | **Você gerencia** as chaves e os usuários do HSM. A AWS **não tem acesso** às chaves |
| Integração com serviços AWS | Nativa | Via **custom key store** do KMS |
| Padrões | PKCS#11 não suportado | Suporta **PKCS#11, JCE, CNG/KSP** (ex: Oracle TDE, SSL offload) |
| Alta disponibilidade | Gerenciada pela AWS | Você monta o **cluster em várias AZs** |
| Custo | Baixo (por chave e por chamada) | Alto (por HSM por hora) |

### Dicas para a prova

- "Criptografar dados em repouso nos serviços AWS com **menor esforço operacional**" → **KMS** (AWS managed key ou customer managed key).
- "Controlar **quem pode usar** a chave, **auditar** o uso e **rotacionar** com período definido" → **Customer managed key**.
- "A chave precisa ser **gerada fora da AWS** / a empresa precisa **controlar o material** da chave" → **Imported key material (BYOK)**. Lembrar que a rotação é **manual** (nova chave + alias).
- "**Hardware dedicado**, single-tenant, FIPS 140 nível 3, AWS sem acesso às chaves, PKCS#11" → **CloudHSM** (ou CloudHSM key store, se precisar integrar com os serviços AWS).
- "Chaves devem ficar **fora da AWS**" → **External key store (XKS)**.
- "Criptografar arquivos **maiores que 4 KB**" → **envelope encryption** (`GenerateDataKey`).
- "Clientes **fora da AWS** precisam criptografar dados ou verificar assinaturas" → **chave assimétrica** (chave pública baixável).
- "Compartilhar um **snapshot criptografado** com outra conta" → snapshot com **customer managed key** + **key policy** liberando a outra conta. AWS managed keys **não** podem ser compartilhadas.
- "Copiar snapshot criptografado para **outra região**" → informar uma **chave da região de destino**.
- "Descriptografar os mesmos dados em **várias regiões** sem recriptografar" → **Multi-Region Keys**.
- "`ThrottlingException` no KMS com muitas requisições ao S3" → **S3 Bucket Keys** (ou data key caching / aumento de quota).
- "Evitar perder dados ao excluir uma chave" → **desabilitar** em vez de excluir, ou usar o **período de espera** + alarme de uso.

## AWS SSM Parameter Store

O AWS Systems Manager Parameter Store é um serviço **serverless** para **armazenar de forma centralizada e segura** dados de **configuração** e **secrets** (ex: URLs de endpoints, nomes de buckets, feature flags, senhas de banco de dados, API keys). Ele faz parte do **AWS Systems Manager (SSM)**.

- Os parâmetros podem ser guardados em **texto claro** ou **criptografados com o KMS**.
- O acesso é controlado com **IAM policies**, inclusive por **caminho** da hierarquia.
- Mantém o **histórico de versões** de cada parâmetro.
- Todas as chamadas de API são **auditadas no CloudTrail**, e as alterações nos parâmetros geram eventos no **EventBridge**.
- É integrado a vários serviços: **Lambda, ECS, EC2, CodeBuild, CodePipeline, CloudFormation** etc.

**Casos de uso:** separar configuração do código (*externalized configuration*), guardar parâmetros diferentes por ambiente (dev, hom, prod) e armazenar secrets simples sem custo adicional.

### Tipos de parâmetro

| Tipo | Descrição | Exemplo |
|---|---|---|
| **String** | Texto simples | `https://api.empresa.com` |
| **StringList** | Lista de valores separados por vírgula | `us-east-1,sa-east-1` |
| **SecureString** | Valor **criptografado com o KMS** | Senha do banco de dados |

Detalhes do **SecureString**:
- Pode usar a AWS managed key `aws/ssm` (padrão) ou uma **customer managed key**.
- Para ler o valor descriptografado, a chamada precisa usar `WithDecryption=true`, e o principal precisa de permissão `ssm:GetParameter` **e** `kms:Decrypt` na chave.
- Com uma customer managed key, é possível controlar **quem pode descriptografar** o parâmetro pela key policy.

### Hierarquia

Os parâmetros podem ser organizados em uma **hierarquia de caminhos**, como pastas:

```
/minha-app/dev/db-url
/minha-app/dev/db-password
/minha-app/prod/db-url
/minha-app/prod/db-password
/outra-app/...
```

- `GetParametersByPath` retorna todos os parâmetros de um caminho (ex: toda a configuração de `/minha-app/prod/`) em uma única chamada.
- As **IAM policies** podem liberar acesso por caminho. Por exemplo, a role da aplicação de dev só lê `arn:aws:ssm:<region>:<account>:parameter/minha-app/dev/*`.

### Tiers: Standard vs. Advanced

| | Standard | Advanced |
|---|---|---|
| Quantidade de parâmetros (por conta/região) | 10.000 | 100.000 |
| Tamanho máximo do valor | **4 KB** | **8 KB** |
| Parameter Policies | ❌ | ✅ |
| Compartilhamento com outras contas (via RAM) | ❌ | ✅ |
| Custo de armazenamento | **Gratuito** | **US$ 0,05** por parâmetro/mês |

> **Obs:** Um parâmetro Standard pode ser **convertido para Advanced** a qualquer momento, mas o contrário **não é possível**: seria preciso excluir e recriar o parâmetro.

**Throughput:** por padrão, a quota é de cerca de **40 transações por segundo**. É possível habilitar o **higher throughput**, que aumenta o limite (até 10.000 TPS) mediante cobrança por chamada de API.

### Parameter Policies (somente Advanced)

Permitem definir regras de ciclo de vida para um parâmetro:

- **Expiration:** **exclui** o parâmetro em uma data/hora definida (ex: uma senha temporária).
- **ExpirationNotification:** envia um evento ao **EventBridge** antes da expiração (ex: 15 dias antes).
- **NoChangeNotification:** envia um evento ao **EventBridge** se o parâmetro **não for alterado** por um período (ex: lembrar de trocar uma senha a cada 90 dias).

> **Obs:** O Parameter Store **não tem rotação automática** de secrets. As parameter policies apenas **notificam**. Para rotacionar, é preciso montar uma automação própria (ex: EventBridge + Lambda) ou usar o **Secrets Manager**.

### Versionamento e labels

- Cada alteração em um parâmetro cria uma **nova versão**. É possível ler uma versão específica (ex: `/minha-app/prod/db-url:3`).
- **Labels** são apelidos para uma versão (ex: `producao`, `v1`), referenciados como `/minha-app/prod/db-url:producao`.

### Parâmetros públicos e referência ao Secrets Manager

- **Public parameters:** a AWS publica parâmetros com informações atualizadas, como o **ID da AMI mais recente** de cada sistema operacional:
  `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`
  Muito útil em templates CloudFormation e Launch Templates para sempre usar a AMI mais recente.
- **Referência ao Secrets Manager:** um secret do Secrets Manager pode ser lido pela API do Parameter Store usando o caminho `/aws/reference/secretsmanager/<nome-do-secret>`.

### Integração com CloudFormation

- **Parameter types:** um parâmetro do template pode ser do tipo `AWS::SSM::Parameter::Value<String>`, e o CloudFormation resolve o valor no Parameter Store durante o deploy (ex: a AMI mais recente via public parameter).
- **Dynamic references:** valores podem ser referenciados diretamente nas propriedades dos recursos:
  - `{{resolve:ssm:/minha-app/prod/db-url}}` para String/StringList.
  - `{{resolve:ssm-secure:/minha-app/prod/db-password}}` para SecureString (suportado apenas por alguns recursos e propriedades).

### Parameter Store vs. Secrets Manager

| | SSM Parameter Store | AWS Secrets Manager |
|---|---|---|
| Foco | **Configuração** em geral e secrets simples | **Secrets** (credenciais, API keys) |
| Criptografia | Opcional (SecureString) | **Sempre** criptografado com KMS |
| **Rotação automática** | ❌ (precisa de automação própria) | ✅ Nativa (Lambda), com integração pronta para **RDS, Aurora, Redshift, DocumentDB** |
| Replicação multi-região | ❌ | ✅ |
| Tamanho máximo | 4 KB (Standard) / 8 KB (Advanced) | 64 KB |
| Custo | **Gratuito** (Standard) | US$ 0,40 por secret/mês + chamadas de API |

### Dicas para a prova

- "Armazenar **configurações** da aplicação de forma centralizada, por ambiente, com **menor custo**" → **Parameter Store** (Standard, hierarquia por caminho).
- "Armazenar uma senha **criptografada** sem custo adicional" → **SecureString** no Parameter Store.
- "Credenciais de banco de dados com **rotação automática**" → **Secrets Manager**, e não o Parameter Store.
- "Garantir que cada aplicação só leia a **sua própria configuração**" → hierarquia + **IAM policy por caminho**.
- "Ser **notificado** quando um parâmetro estiver para expirar ou não for alterado há muito tempo" → **Parameter Policies** (Advanced tier) + **EventBridge**.
- "Parâmetro **maior que 4 KB**" → **Advanced tier** (até 8 KB). Acima disso, Secrets Manager (até 64 KB) ou S3.
- "Sempre usar a **AMI mais recente** do Amazon Linux no CloudFormation" → **public parameter** `/aws/service/ami-amazon-linux-latest/...`.
- "`ThrottlingException` ao ler parâmetros com alto volume" → habilitar **higher throughput** ou fazer **cache** dos valores na aplicação.
- "Compartilhar parâmetros com outras contas" → **Advanced tier** + **AWS RAM**.

## AWS Secrets Manager

O AWS Secrets Manager é o serviço para **armazenar, gerenciar e rotacionar secrets**, como credenciais de banco de dados, API keys, tokens OAuth e chaves SSH. A ideia é **eliminar secrets fixos no código** (*hardcoded*): a aplicação busca o secret via API em tempo de execução.

- Os secrets são **sempre criptografados com o KMS**: com a AWS managed key `aws/secretsmanager` (padrão) ou com uma **customer managed key**.
- Oferece **rotação automática** nativa, com integração pronta para **RDS, Aurora, Redshift e DocumentDB**.
- O acesso é controlado por **IAM policies** e por **resource-based policies** no próprio secret.
- Suporta **replicação multi-região**.
- Todas as chamadas de API são **auditadas no CloudTrail**.
- Um secret pode ter até **64 KB** e normalmente é armazenado como **JSON** (ex: `{"username": "admin", "password": "...", "host": "..."}`).

**Custo:** **US$ 0,40** por secret/mês + **US$ 0,05** a cada 10.000 chamadas de API.

### Rotação automática

A rotação troca o valor do secret **periodicamente e sem intervenção manual**, atualizando ao mesmo tempo o secret e a credencial no banco ou serviço de destino. O agendamento pode ser feito por **frequência** (ex: a cada 30 dias) ou por **expressão cron**, com intervalo mínimo de **4 horas**.

Existem duas formas de rotação:

- **Managed rotation:** a rotação é feita **pelo próprio serviço** dono da credencial, sem Lambda. Exemplo: o **RDS/Aurora** pode gerenciar a senha do usuário master no Secrets Manager (`ManageMasterUserPassword`), com rotação automática a cada **7 dias** por padrão.
- **Rotação via Lambda:** o Secrets Manager invoca uma **função Lambda** que executa a troca. A AWS fornece **templates prontos** para RDS, Aurora, Redshift e DocumentDB. Para outros casos (ex: API key de um serviço terceiro), é preciso escrever uma Lambda própria.

**Etapas da Lambda de rotação:**

1. **`createSecret`:** gera um novo valor e o grava como uma nova versão com o label `AWSPENDING`.
2. **`setSecret`:** aplica o novo valor no banco/serviço de destino.
3. **`testSecret`:** testa se a nova credencial funciona.
4. **`finishSecret`:** move o label `AWSCURRENT` para a nova versão. A versão antiga recebe o label `AWSPREVIOUS`.

> **Obs:** Os **staging labels** identificam as versões do secret: `AWSCURRENT` (versão atual, retornada por padrão), `AWSPENDING` (em rotação) e `AWSPREVIOUS` (anterior, útil para rollback).

**Estratégias de rotação:**

| Estratégia | Como funciona | Quando usar |
|---|---|---|
| **Single user** | Troca a senha do **mesmo usuário** | Mais simples. Pode haver uma breve janela em que conexões com a senha antiga falham |
| **Alternating users** | Mantém **dois usuários** e alterna entre eles a cada rotação. Um terceiro secret (de um superusuário) é usado para alterar as senhas | Aplicações com **alta disponibilidade**, que não toleram falhas de conexão durante a rotação |

> **Obs:** A Lambda de rotação precisa de **acesso de rede** ao banco de dados (normalmente rodando **na mesma VPC**) **e** ao endpoint do Secrets Manager (via **VPC Interface Endpoint** ou **NAT Gateway**). Esse é um ponto comum de falha da rotação.

### Acesso cross-account

Para que uma conta B leia um secret da conta A:

1. **Resource policy** no secret (conta A) permitindo o principal da conta B (`secretsmanager:GetSecretValue`).
2. O secret precisa estar criptografado com uma **customer managed key**, com a **key policy** permitindo `kms:Decrypt` para a conta B. Secrets criptografados com a `aws/secretsmanager` **não podem** ser lidos por outras contas.
3. **IAM policy** na conta B permitindo o principal acessar o secret e a chave KMS da conta A.

> **Obs:** A resource policy também pode ser usada para **bloquear** acessos. Por exemplo, negar o acesso a qualquer principal que não venha da VPC da aplicação (condition `aws:SourceVpc` ou `aws:SourceVpce`).

### Replicação Multi-Region

Um secret pode ser **replicado para outras regiões**. O Secrets Manager mantém as réplicas **sincronizadas** com o secret primário, inclusive após as rotações.

- As réplicas são **somente leitura**. Cada uma é criptografada com uma chave KMS da sua própria região.
- Em um cenário de **disaster recovery**, uma réplica pode ser **promovida a secret independente** (*standalone*).

**Casos de uso:** aplicações **multi-região** (ex: com **Aurora Global Database** ou **DynamoDB Global Tables**) e DR, em que a aplicação na região secundária precisa das mesmas credenciais sem depender da região primária.

### Como as aplicações consomem os secrets

- **SDK/API:** `GetSecretValue` (ou `BatchGetSecretValue` para vários secrets de uma vez). As **bibliotecas de cache** do Secrets Manager (Java, Python, .NET, Go) reduzem a latência e o número de chamadas.
- **Lambda:** a **AWS Parameters and Secrets Lambda Extension** faz cache dos secrets localmente na função.
- **ECS:** a task definition referencia o secret no campo `secrets` (`valueFrom` com o ARN). O valor é injetado como **variável de ambiente** no container. A **task execution role** precisa de permissão para ler o secret.
- **EKS:** **Secrets Store CSI Driver** com o provider da AWS, montando os secrets como volumes ou sincronizando com Kubernetes Secrets.
- **CloudFormation:**
  - **Dynamic reference:** `{{resolve:secretsmanager:MeuSecret:SecretString:password}}`.
  - `AWS::SecretsManager::Secret` com `GenerateSecretString` para **gerar uma senha aleatória** durante o deploy, sem expô-la no template.
  - `AWS::SecretsManager::SecretTargetAttachment` para associar o secret a uma instância RDS.

### Exclusão de secrets

- A exclusão tem um **recovery window** de **7 a 30 dias** (padrão de 30). Durante esse período, o secret não pode ser usado, mas a exclusão pode ser **cancelada**.
- Com `ForceDeleteWithoutRecovery`, o secret é excluído **imediatamente**, sem possibilidade de recuperação.

### Monitoramento e compliance

- **CloudTrail:** registra quem acessou ou alterou cada secret.
- **AWS Config:** regras gerenciadas como `secretsmanager-rotation-enabled-check` (rotação habilitada), `secretsmanager-secret-periodic-rotation` (rotação ocorrendo no prazo) e `secretsmanager-secret-unused` (secrets sem uso).
- **EventBridge:** eventos de rotação (sucesso ou falha) podem disparar notificações ou automações.

### Secrets Manager vs. Parameter Store

| | AWS Secrets Manager | SSM Parameter Store |
|---|---|---|
| Foco | **Secrets** (credenciais, API keys) | **Configuração** em geral e secrets simples |
| Criptografia | **Sempre** criptografado com KMS | Opcional (SecureString) |
| **Rotação automática** | ✅ Nativa (managed rotation ou Lambda) | ❌ (precisa de automação própria) |
| Replicação multi-região | ✅ | ❌ |
| Cross-account | ✅ Resource policy | Advanced tier + RAM |
| Tamanho máximo | 64 KB | 4 KB (Standard) / 8 KB (Advanced) |
| Custo | US$ 0,40 por secret/mês + chamadas de API | **Gratuito** (Standard) |

### Dicas para a prova

- "Credenciais de banco de dados com **rotação automática**" → **Secrets Manager**.
- "Rotacionar a senha do master user do **RDS/Aurora** com **menor esforço operacional**" → **managed rotation** do RDS com Secrets Manager (`ManageMasterUserPassword`).
- "Rotacionar uma **API key de um serviço terceiro**" → Secrets Manager com **Lambda de rotação customizada**.
- "A rotação está **falhando** com timeout" → a Lambda de rotação não tem **acesso de rede** ao banco ou ao endpoint do Secrets Manager (falta **VPC endpoint** ou **NAT Gateway**).
- "A aplicação **não pode ter falhas de conexão** durante a rotação" → estratégia **alternating users**.
- "Remover senhas **hardcoded** do código ou das variáveis de ambiente" → Secrets Manager (ou Parameter Store SecureString, se não houver necessidade de rotação).
- "Compartilhar um secret com **outra conta**" → **resource policy** no secret + **customer managed key** com key policy liberando a outra conta.
- "A aplicação na **região de DR** precisa das mesmas credenciais" → **replicação multi-região** do secret.
- "Injetar o secret em um **container ECS**" → campo `secrets` da task definition + permissão na **task execution role**.
- "Gerar a senha do banco no **CloudFormation** sem expô-la no template" → `GenerateSecretString` + **dynamic reference**.
- "Armazenar configurações **simples e baratas**, sem rotação" → **Parameter Store**, e não Secrets Manager.
