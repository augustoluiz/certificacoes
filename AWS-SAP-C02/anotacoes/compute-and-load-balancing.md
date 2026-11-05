---
titulo: Compute and Load Balancing
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - Amazon EC2
criado-em: 2026-11-05
atualizado-em: 2026-11-05
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, compute, load-balancing]
---

# Compute and Load Balancing

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## Índice

- **Compute:** [Amazon EC2](#amazon-elastic-compute-cloud-ec2)
- **Revisão:** [Qual serviço usar](#revisão-qual-serviço-usar)

## Amazon Elastic Compute Cloud (EC2)

O Amazon EC2 oferece **servidores virtuais** (instâncias) sob demanda na AWS. Na SAP-C02, o foco não é como criar uma instância, e sim **escolher** o tipo de instância, a **opção de compra**, o **posicionamento** e a estratégia de **resiliência** certos para cada cenário, com o menor custo e esforço operacional.

- As instâncias rodam no **AWS Nitro System**: hardware dedicado para rede, armazenamento e segurança, que entrega desempenho próximo ao bare metal e isola a instância até mesmo dos operadores da AWS.
- Cada instância roda em uma **Availability Zone**. A alta disponibilidade vem de distribuir instâncias em **várias AZs** (normalmente com **Auto Scaling** e **load balancer**).
- O **modelo de responsabilidade compartilhada** coloca o sistema operacional, os patches, as aplicações e os security groups sob responsabilidade do **cliente**.

### Tipos de instância

O nome de um tipo de instância segue o padrão **família + geração + atributos + tamanho**. Exemplo: `m7gd.2xlarge` = família **m** (uso geral), geração **7**, **g** (processador **Graviton**), **d** (disco **NVMe** local, instance store), tamanho **2xlarge**.

| Família | Otimizada para | Exemplos de uso |
|---|---|---|
| **General purpose** (M, T) | Equilíbrio de CPU, memória e rede | Servidores web, aplicações em geral. **T** = burstable (créditos de CPU) |
| **Compute optimized** (C) | CPU | Processamento em lote, encoding de mídia, servidores de jogos, HPC |
| **Memory optimized** (R, X, U, z) | Memória | Bancos em memória, cache, SAP HANA, analytics em tempo real |
| **Accelerated computing** (P, G, Inf, Trn) | GPU / chips de ML | Treinamento e inferência de ML, renderização gráfica |
| **Storage optimized** (I, D, Im/Is) | I/O de disco local muito alto | Bancos NoSQL, data warehouses, sistemas de arquivos distribuídos |
| **HPC optimized** (Hpc) | Computação científica com MPI | Simulações, modelagem, CFD |

**Atributos comuns no nome:** `g` = Graviton (ARM), `a` = AMD, `i` = Intel, `d` = instance store NVMe, `n` = rede aprimorada, `e` = memória/armazenamento extra, `z` = alta frequência.

> **Obs:** As instâncias **Graviton** (processadores ARM da AWS) oferecem até **40% melhor preço/desempenho** que as equivalentes x86, mas exigem que a aplicação e suas dependências sejam compatíveis com **ARM64**. Em questões de redução de custo com aplicações interpretadas ou em containers multi-arquitetura, Graviton costuma ser uma boa resposta.

**Instâncias burstable (T):** acumulam **créditos de CPU** quando ficam abaixo da linha de base e os gastam em picos. No modo **unlimited** (padrão nas T3/T4g), a instância pode passar do limite de créditos, pagando pelo excedente. Para cargas com uso de CPU **constantemente alto**, uma família fixa (M, C) é mais barata.

### Opções de compra

| Opção | Desconto (aprox.) | Compromisso | Quando usar |
|---|---|---|---|
| **On-Demand** | — | Nenhum | Cargas imprevisíveis, curtas, testes. Cobrança por segundo (Linux/Windows) |
| **Savings Plans** | Até ~72% | US$/hora por **1 ou 3 anos** | Uso constante. **Compute SP** vale para qualquer família, região, SO, **Fargate** e **Lambda**; **EC2 Instance SP** fixa a família e a região (desconto maior) |
| **Reserved Instances** | Até ~72% | Tipo de instância por **1 ou 3 anos** | Uso constante. **Standard** (maior desconto, pode ser vendida no Marketplace) ou **Convertible** (permite trocar a família) |
| **Spot** | Até ~90% | Nenhum (pode ser **interrompida** com aviso de 2 minutos) | Cargas **tolerantes a falhas** e **flexíveis**: batch, big data, CI/CD, render, containers stateless |
| **Dedicated Hosts** | — | On-Demand ou reserva | Servidor físico **inteiro** dedicado, com visibilidade de **sockets e cores**: licenças **BYOL** por core/socket (Windows Server, SQL Server, Oracle) e compliance |
| **Dedicated Instances** | — | On-Demand ou reserva | Hardware **não compartilhado com outros clientes**, mas **sem controle** do host físico |
| **On-Demand Capacity Reservations** | — (combina com Savings Plans/RIs para desconto) | Nenhum (paga enquanto a reserva existir, usada ou não) | **Garantir capacidade** em uma AZ específica (eventos, DR, cargas críticas) |
| **Capacity Blocks for ML** | — | Período definido | Reservar **clusters de GPU** por dias ou semanas para treinamento de ML |

**Pontos que caem na prova:**

- **Savings Plans vs. Reserved Instances:** os Savings Plans são o modelo **recomendado** hoje. O **Compute Savings Plan** é o mais flexível (cobre mudanças de família, região, SO e até Fargate/Lambda).
- **Desconto ≠ reserva de capacidade:** Savings Plans e RIs **regionais** dão desconto, mas **não garantem capacidade**. Para garantir capacidade, use **On-Demand Capacity Reservations** (que recebem o desconto de um Savings Plan ou de RIs regionais aplicáveis). **RIs zonais** dão desconto **e** reservam capacidade em uma AZ.
- **Multi-account:** com **AWS Organizations**, os descontos de Savings Plans e RIs são **compartilhados** entre as contas (pode ser desligado por conta). Capacity Reservations podem ser **compartilhadas via RAM**.
- **Licenças por core/socket** (BYOL) → **Dedicated Hosts** (com **License Manager** para controlar o uso).

### Spot Instances

- A AWS pode **recuperar** uma instância Spot quando precisa da capacidade, com um aviso de **2 minutos** (*Spot Instance Interruption Warning*), entregue via **instance metadata** e **EventBridge**.
- A **EC2 Instance Rebalance Recommendation** avisa **antes**, quando a instância está com **risco elevado** de interrupção, dando mais tempo para reagir.
- O comportamento na interrupção pode ser **terminate** (padrão), **stop** ou **hibernate**.
- O preço Spot muda **gradualmente** conforme a oferta e a demanda de longo prazo. Não há mais leilão: paga-se o preço Spot vigente, limitado opcionalmente por um preço máximo.

**Boas práticas:**

- Ser **flexível**: usar **vários tipos de instância** e **várias AZs** para acessar mais **pools de capacidade Spot**. Ferramenta útil: **attribute-based instance type selection** (escolher por vCPU e memória, em vez de listar tipos).
- Usar a allocation strategy **`price-capacity-optimized`** (recomendada): escolhe os pools com **maior capacidade disponível** e menor preço, reduzindo interrupções.
- Combinar **On-Demand** (capacidade base) e **Spot** (escala) no mesmo Auto Scaling group com uma **mixed instances policy**.
- Habilitar o **Capacity Rebalancing** no ASG: ao receber a rebalance recommendation, o ASG lança uma substituta **antes** da interrupção.
- Projetar a aplicação para ser **stateless** ou fazer **checkpoints** do processamento (S3, DynamoDB).

**Spot Fleet vs. EC2 Fleet vs. Auto Scaling group:** o **Auto Scaling group** com mixed instances policy é a forma recomendada para a maioria dos casos. **EC2 Fleet** atende a lançamentos únicos (`instant`) ou mantém uma capacidade-alvo combinando On-Demand e Spot; o **Spot Fleet** é a API mais antiga.

![Diagrama Amazon EC2 - Spot com Auto Scaling](./img/amazon-ec2/amazon-ec2-02.png)

### Placement groups

Controlam **como as instâncias são distribuídas** no hardware físico de uma AZ.

| Estratégia | Como distribui | Uso | Limitações |
|---|---|---|---|
| **Cluster** | Instâncias **próximas**, no mesmo segmento de rede de alta bisseção em **uma AZ** | **HPC**, cargas com **baixa latência** e **alto throughput** entre nós | Uma falha de hardware/rack pode afetar **todas**. Recomenda-se o mesmo tipo de instância e lançar todas de uma vez (evita *insufficient capacity*) |
| **Spread** | Cada instância em um **rack diferente** (hardware, rede e energia separados) | **Poucas instâncias críticas** que não podem falhar juntas | Máximo de **7 instâncias por AZ** por grupo. Pode abranger **várias AZs** |
| **Partition** | Grupos de instâncias (**partições**) em racks separados | Sistemas distribuídos com consciência de topologia: **HDFS, HBase, Cassandra, Kafka** | Até **7 partições por AZ**, com **centenas** de instâncias. A instância sabe sua partição pelos **metadados** |

![Diagrama Amazon EC2 - Placement groups](./img/amazon-ec2/amazon-ec2-01.png)

> **Obs:** Em caso de *insufficient capacity* ao adicionar instâncias a um **cluster placement group**, a recomendação é **parar e iniciar** todas as instâncias do grupo (para que sejam realocadas juntas) ou tentar novamente mais tarde.

### Rede

- **Elastic Network Interface (ENI):** placa de rede virtual com IPs privados, IP público/Elastic IP, MAC e security groups. Pode ser **movida** entre instâncias da mesma AZ, o que permite **failover** rápido de IP (padrão comum em appliances e licenças atreladas ao MAC).
- **Elastic IP:** IPv4 público **fixo**, associado à conta até ser liberado. Desde 2024, **todo IPv4 público é cobrado**, inclusive Elastic IPs em uso.
- **Enhanced networking (ENA):** SR-IOV para maior **largura de banda** (até centenas de Gbps nas instâncias maiores), mais **PPS** e menor latência. Habilitado por padrão nas instâncias Nitro.
- **Elastic Fabric Adapter (EFA):** ENA com **OS-bypass** (no Linux), permitindo que aplicações **HPC** e de **ML distribuído** com **MPI** ou **NCCL** se comuniquem com latência muito baixa. Usado com **cluster placement groups**.
- **Jumbo frames:** MTU de **9001** dentro da VPC. Tráfego para a Internet, VPN e entre VPCs via peering usa MTU 1500 (o Transit Gateway suporta 8500).

### Armazenamento

- **EBS:** volumes de **rede**, persistentes, presos a **uma AZ** (snapshots no S3 permitem copiar entre AZs e regiões). Detalhes na seção de storage.
- **Instance store:** discos **NVMe locais** ao host físico, com **altíssimo IOPS** e sem custo extra, mas **efêmeros**: os dados se perdem ao **parar, hibernar ou terminar** a instância ou em falha do host. Usado para cache, buffers, dados temporários e sistemas que **replicam** os dados entre nós (ex: Cassandra, Kafka).
- **EBS-optimized:** largura de banda dedicada entre a instância e o EBS (padrão nas instâncias atuais).

### AMIs, launch templates e user data

- **AMI (Amazon Machine Image):** imagem com o SO, a configuração e o software. É **regional**: para usar em outra região, é preciso **copiar** a AMI. Pode ser **compartilhada** com outras contas ou com a Organization.
  - Para compartilhar uma AMI **criptografada**, a chave KMS precisa ser uma **customer managed key** compartilhada com a conta de destino (chaves AWS managed não podem ser compartilhadas). A conta de destino normalmente **copia** a AMI e a recriptografa com sua própria chave.
  - **EC2 Image Builder** automatiza a criação, o teste e a distribuição de AMIs (golden images) entre contas e regiões.
  - **Allowed AMIs** (configuração da conta, aplicável via **declarative policy**) restringe quais AMIs podem ser usadas.
- **Golden AMI vs. user data:** a AMI pré-configurada reduz o **tempo de boot** (bom para escalar rápido); o **user data** (script executado no primeiro boot) é mais flexível, mas aumenta o tempo até a instância ficar pronta. Muitas soluções combinam os dois.
- **Launch templates:** definem a configuração de lançamento (AMI, tipo, rede, IAM, user data), com **versionamento**. São **obrigatórios** para recursos novos, como mixed instances policies no ASG. As antigas **launch configurations** estão **descontinuadas**.

### Instance metadata (IMDS) e IAM

- **Instance profile:** associa uma **IAM role** à instância. As aplicações obtêm **credenciais temporárias**, rotacionadas automaticamente, pelo **IMDS** (`169.254.169.254`). Nunca guardar access keys na instância.
- **IMDSv2:** exige um **token de sessão** obtido com um `PUT` antes de ler os metadados, protegendo contra ataques de **SSRF**. O **hop limit** (TTL) igual a **1** impede que containers na instância alcancem o IMDS.
- Para **exigir o IMDSv2** em toda a organização: **declarative policy** (EC2), **SCP** com a condition `ec2:MetadataHttpTokens`, ou a configuração padrão da conta.

![Diagrama Amazon EC2 - Instance profile e IMDSv2](./img/amazon-ec2/amazon-ec2-03.png)

### Ciclo de vida e recuperação

- **Stop/start:** a instância normalmente vai para **outro host físico** (útil para fugir de um host degradado). O IP público muda (o Elastic IP não), e os dados do instance store são perdidos.
- **Hibernate:** grava a **RAM** no volume EBS raiz (que precisa ser **criptografado** e ter espaço suficiente) e retoma os processos de onde pararam. Útil para aplicações com **inicialização demorada** (cache em memória). Limite de **60 dias** hibernada.
- **Status checks:**
  - **System status check:** problemas na infraestrutura da AWS (host, rede, energia). Solução: **recuperar** a instância em outro host.
  - **Instance status check:** problemas na própria instância (SO, configuração de rede, memória). Solução: **reboot** ou correção pelo cliente.
  - **Attached EBS status check:** problemas nos volumes EBS anexados.
- **Recuperação automática:** a **simplified automatic recovery** (padrão em tipos compatíveis) ou um **alarme do CloudWatch** com a ação **recover** move a instância para um novo host mantendo **instance ID, IPs privados, Elastic IP e metadados**. Não funciona com **instance store**.
- **Termination protection** e **stop protection** evitam encerramentos acidentais (não impedem o término por Auto Scaling ou pelo SO).

### Acesso e monitoramento

- **Systems Manager Session Manager:** acesso ao shell **sem SSH/RDP**, **sem portas abertas** e **sem bastion host**, com auditoria no CloudTrail e logs da sessão no S3/CloudWatch. Exige o **SSM Agent** e acesso aos endpoints do SSM (Internet/NAT ou **VPC endpoints**).
- **EC2 Instance Connect:** chaves SSH temporárias enviadas via API. O **EC2 Instance Connect Endpoint** permite SSH/RDP para instâncias em **subnets privadas** sem bastion e sem IP público.
- **CloudWatch:** métricas a cada **5 minutos** (básico) ou **1 minuto** (detailed monitoring). **Memória** e **espaço em disco** não são coletados por padrão: exigem o **CloudWatch agent**.

### Quotas

- As quotas de **On-Demand** e de **Spot** são por **região** e medidas em **vCPUs** por família de instância (não em número de instâncias). Antes de eventos de escala ou migrações grandes, solicitar aumento via **Service Quotas**.

### Dicas para a prova

- "Carga **constante** por 1 ou 3 anos com **máxima flexibilidade** (trocar família, região, ou usar Fargate/Lambda)" → **Compute Savings Plan**.
- "Garantir **capacidade** em uma AZ para um evento ou para DR, **sem compromisso de longo prazo**" → **On-Demand Capacity Reservation**.
- "Cargas **tolerantes a interrupção** com **menor custo**" → **Spot** (vários tipos de instância e AZs, `price-capacity-optimized`).
- "Base estável + picos com menor custo" → ASG com **mixed instances policy**: On-Demand/Savings Plans para a base e **Spot** para o excedente.
- "Reagir **antes** da interrupção Spot" → **rebalance recommendation** + **Capacity Rebalancing** no ASG.
- "Licenças **por socket/core** (BYOL) ou exigência de servidor físico dedicado" → **Dedicated Hosts**.
- "Comunicação de **baixíssima latência** entre nós de HPC" → **cluster placement group** + **EFA**.
- "Instâncias críticas que **não podem falhar juntas**" → **spread placement group** (máx. 7 por AZ).
- "**Cassandra, HDFS, Kafka** com consciência de rack" → **partition placement group**.
- "Mover rapidamente um **IP/MAC** para uma instância standby" → mover a **ENI** secundária.
- "Aplicação com **inicialização muito lenta** (cache em memória) precisa voltar rápido" → **hibernate**.
- "Instância com falha de **hardware** do host deve voltar com o **mesmo IP e ID**" → **automatic recovery** / alarme do CloudWatch com ação **recover**.
- "Proteger as credenciais da instância contra **SSRF**" → **IMDSv2** (obrigatório) com **hop limit 1**.
- "Acessar instâncias em subnets **privadas** sem bastion e sem abrir a porta 22" → **Session Manager** (ou **EC2 Instance Connect Endpoint**).
- "Monitorar o uso de **memória**" → **CloudWatch agent**.
- "Reduzir o **tempo de boot** das instâncias no Auto Scaling" → **golden AMI** (e/ou **warm pools** no ASG).
- "Compartilhar uma AMI **criptografada** com outra conta" → usar uma **customer managed KMS key** e conceder acesso à conta de destino.
- "Erro `InsufficientInstanceCapacity`" → tentar **outra AZ** ou **outro tipo** de instância (ou reservar capacidade). "Erro `VcpuLimitExceeded`" → aumentar a **quota de vCPUs**.

[↑ Índice](#índice)

## Revisão: qual serviço usar

Resumo das comparações que mais aparecem nas questões. Os detalhes estão nas seções de cada serviço.

### Opções de compra do EC2

| Pergunta do cenário | Solução |
|---|---|
| Carga constante, máxima flexibilidade (família, região, Fargate, Lambda) | [Compute Savings Plan](#opções-de-compra) |
| Carga constante em uma família e região, maior desconto | [EC2 Instance Savings Plan / Standard RI](#opções-de-compra) |
| Carga tolerante a interrupções, menor custo | [Spot](#spot-instances) |
| Garantir capacidade em uma AZ sem compromisso longo | [On-Demand Capacity Reservation](#opções-de-compra) |
| Licenças por socket/core (BYOL) | [Dedicated Hosts](#opções-de-compra) |
| Carga curta, imprevisível ou de teste | [On-Demand](#opções-de-compra) |

### Posicionamento das instâncias

| Pergunta do cenário | Solução |
|---|---|
| Latência mínima entre nós (HPC) | [Cluster placement group](#placement-groups) + EFA |
| Poucas instâncias críticas isoladas umas das outras | [Spread placement group](#placement-groups) |
| Sistemas distribuídos com consciência de rack (HDFS, Cassandra, Kafka) | [Partition placement group](#placement-groups) |

[↑ Índice](#índice)
