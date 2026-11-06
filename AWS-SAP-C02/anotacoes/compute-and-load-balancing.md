---
titulo: Compute and Load Balancing
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - Amazon EC2
  - AWS ParallelCluster
  - AWS Parallel Computing Service (PCS)
  - AWS Batch
  - Amazon FSx for Lustre
  - Amazon File Cache
  - Amazon DCV
criado-em: 2026-11-05
atualizado-em: 2026-11-06
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, compute, load-balancing]
---

# Compute and Load Balancing

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## Índice

- **Compute:** [Amazon EC2](#amazon-elastic-compute-cloud-ec2) · [High Performance Computing (HPC)](#high-performance-computing-hpc)
- **Revisão:** [Qual serviço usar](#revisão-qual-serviço-usar)

## Amazon Elastic Compute Cloud (EC2)

O Amazon EC2 oferece **servidores virtuais** (instâncias) sob demanda na AWS. Em vez de comprar um servidor físico, você escolhe uma "configuração" de CPU, memória, rede e disco, liga a máquina em minutos e paga apenas pelo tempo de uso. Por baixo, a AWS mantém data centers com milhares de servidores físicos (**hosts**). Cada host é dividido em várias máquinas virtuais por um **hypervisor**, e cada uma delas é uma **instância** que você controla como se fosse um computador seu: escolhe o sistema operacional, instala software, define quem acessa.

**Vindo do Lambda:** a melhor forma de entender o EC2 é pensar em tudo o que o Lambda faz por você e que, no EC2, passa a ser **sua responsabilidade**. No Lambda, você entrega o código e a AWS cuida do servidor, da escala, dos patches e da disponibilidade. No EC2, você recebe o **servidor** e decide todo o resto. Em troca, ganha controle total: sem limite de 15 minutos de execução, escolha de SO, GPU, disco local, processos que ficam rodando o tempo todo e licenças de software tradicionais.

| No Lambda... | No EC2... |
|---|---|
| Você configura a **memória** e a CPU vem proporcional | Você escolhe o **tipo de instância** (combinação fixa de vCPU, memória, rede e disco) |
| Escala automática por requisição, **até zero** | Escala com um **Auto Scaling group**, em instâncias inteiras, e normalmente com um mínimo de instâncias sempre ligadas |
| Paga por **invocação e duração** (ms) | Paga por **segundo de instância ligada**, mesmo ociosa |
| Alta disponibilidade **multi-AZ automática** | Você distribui as instâncias em **várias AZs** e coloca um **load balancer** na frente |
| **Execution role** entrega credenciais à função | **Instance profile** entrega credenciais à instância (via IMDS) |
| Pacote de deploy / **container image** | **AMI** (imagem do disco) + **user data** (script de boot) |
| Configuração da função (memória, timeout, VPC, variáveis) | **Launch template** |
| Ambiente efêmero, `/tmp` descartável | Instância **persistente**: o disco EBS sobrevive a reinícios. O **instance store** se comporta como o `/tmp` |
| **Cold start** | **Tempo de boot** (de dezenas de segundos a minutos) |
| Concorrência reservada / limite de concorrência | **Capacity Reservations** / **quotas de vCPU** |
| Logs e métricas no CloudWatch automaticamente | Métricas básicas automáticas; **logs, memória e disco** exigem o **CloudWatch agent** |
| A AWS aplica os patches do runtime | **Você** aplica os patches do SO (ex: **SSM Patch Manager**) |

Na SAP-C02, o foco não é como criar uma instância, e sim **escolher** o tipo de instância, a **opção de compra**, o **posicionamento** e a estratégia de **resiliência** certos para cada cenário, com o menor custo e esforço operacional.

- As instâncias rodam no **AWS Nitro System**: hardware dedicado para rede, armazenamento e segurança, que entrega desempenho próximo ao bare metal e isola a instância até mesmo dos operadores da AWS.
- Cada instância roda em uma **Availability Zone**. A alta disponibilidade vem de distribuir instâncias em **várias AZs** (normalmente com **Auto Scaling** e **load balancer**).
- O **modelo de responsabilidade compartilhada** coloca o sistema operacional, os patches, as aplicações e os security groups sob responsabilidade do **cliente**.

### Tipos de instância

Um **tipo de instância** é o "tamanho e formato" da máquina virtual: quantas **vCPUs** (CPUs virtuais, explicadas a seguir), quanta **memória RAM**, quanta **largura de banda de rede** e se ela tem **disco local**. As aplicações têm gargalos diferentes: um servidor web precisa de equilíbrio, um banco em memória precisa de muita RAM, um treinamento de ML precisa de GPU. Por isso, a AWS agrupa os tipos em **famílias**, cada uma otimizada para um recurso. Escolher a família certa é a primeira alavanca de custo: pagar por memória que a aplicação não usa é desperdício.

> **Paralelo com o Lambda:** no Lambda, só há um controle, a **memória**, e a CPU acompanha. No EC2, você escolhe a **proporção** entre CPU e memória pela família. As famílias **C** têm mais CPU por GB, as **R** têm mais memória por vCPU. A arquitetura **arm64 (Graviton)** do Lambda é o mesmo processador das instâncias com `g` no nome, com a mesma vantagem de preço.

#### O que é uma vCPU

A **vCPU** (*virtual CPU*) é a unidade de processamento que a AWS entrega para uma instância. Ela é o que o sistema operacional enxerga como um "processador": o comando `nproc` no Linux ou o Gerenciador de Tarefas no Windows mostram uma CPU para cada vCPU. Para entender o que existe por trás dela, é preciso olhar o hardware em camadas:

1. **Host:** o servidor físico no data center da AWS.
2. **Socket:** cada chip de processador encaixado na placa-mãe. Um host costuma ter 1 ou 2.
3. **Núcleo (core):** cada processador tem vários núcleos, e cada núcleo é uma unidade independente capaz de executar instruções.
4. **Thread de hardware:** em processadores **x86** (Intel e AMD), a tecnologia **SMT** (*Simultaneous Multithreading*, chamada de **Hyper-Threading** pela Intel) faz cada núcleo executar **2 threads** ao mesmo tempo. Elas compartilham as unidades de execução e o cache do núcleo, aproveitando os momentos em que uma delas fica esperando dados da memória.

Na maioria das instâncias x86, **1 vCPU = 1 thread de hardware**, ou seja, **meio núcleo físico**. O **Nitro Hypervisor** reparte as threads do host entre as instâncias. Uma `m7i.large` (2 vCPUs) recebe as duas threads de **um** núcleo, e uma `m7i.2xlarge` (8 vCPUs) recebe **quatro** núcleos.

![Diagrama Amazon EC2 - Do servidor físico à vCPU](./img/amazon-ec2/amazon-ec2-04.png)

**Por que isso importa:**

- **x86 vs. Graviton:** os processadores **Graviton** não usam SMT, então **1 vCPU = 1 núcleo físico inteiro**. Duas vCPUs de uma `m7g.large` não disputam o mesmo núcleo como as de uma `m7i.large`, o que deixa o desempenho por vCPU mais previsível. Algumas famílias AMD recentes (ex: `m7a`) também entregam 1 núcleo por vCPU.
- **Licenciamento:** softwares como **Oracle** e **SQL Server** podem ser licenciados por **núcleo** ou **vCPU**. A opção **Optimize CPUs** permite lançar a instância com **menos núcleos** (`CoreCount`) e/ou **sem SMT** (`ThreadsPerCore = 1`), mantendo a mesma memória. Assim, uma carga que precisa de muita RAM não paga licença para CPUs que não usa. O preço da instância **não** muda. Para controlar licenças por **socket**, o caminho é o **Dedicated Host**, que mostra os sockets e núcleos físicos.
- **HPC:** algumas cargas de computação científica rodam melhor **sem SMT**, cada thread com um núcleo só para ela. Por isso as instâncias **Hpc** já vêm com SMT desligado.
- **Burstable (T):** a "linha de base" das instâncias T é um **percentual de cada vCPU** (ex: 20%). Acima disso, a instância gasta créditos de CPU.
- **Quotas:** os limites On-Demand e Spot por região são contados em **vCPUs**, não em número de instâncias.

![Diagrama Amazon EC2 - x86, Graviton, Optimize CPUs e Lambda](./img/amazon-ec2/amazon-ec2-05.png)

> **Paralelo com o Lambda:** no Lambda, você nunca escolhe vCPUs. A CPU é alocada **proporcionalmente à memória**: cerca de **1 vCPU a cada 1.769 MB**, chegando a **6 vCPUs com 10.240 MB**. Por isso, aumentar a memória de uma função com CPU como gargalo a deixa mais rápida mesmo sem precisar de RAM. Se o código não for **multi-thread**, a partir de ~1.769 MB ele não aproveita as vCPUs extras, e o mesmo vale para o EC2: uma aplicação de thread única não fica mais rápida em uma instância com 16 vCPUs.

#### Nomenclatura e famílias

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

A mesma instância pode custar valores muito diferentes dependendo de **como você a compra**. A lógica é uma troca entre **compromisso** e **desconto**:

- quem paga o preço cheio (**On-Demand**) tem liberdade total para ligar e desligar quando quiser;
- quem se compromete a usar por **1 ou 3 anos** (**Savings Plans** e **Reserved Instances**) ganha desconto, porque a AWS consegue planejar sua capacidade;
- quem aceita usar a **capacidade ociosa** da AWS, sabendo que pode perdê-la a qualquer momento (**Spot**), ganha o maior desconto.

Há ainda opções que não são sobre preço, e sim sobre **isolamento** (hardware dedicado, para licenças e compliance) ou **garantia de capacidade** (ter certeza de que a instância vai estar disponível quando você precisar). Na prova, a pergunta-chave é: a carga é **previsível**? **Tolera interrupção**? Precisa de **garantia** de capacidade ou de **hardware dedicado**?

> **Paralelo com o Lambda:** o **Compute Savings Plan**, que talvez você já conheça, é o mesmo para EC2, Fargate e Lambda. Você se compromete com um gasto em US$/hora, e o desconto vale para qualquer um dos três. Uma diferença importante: o EC2 cobra enquanto a instância está **ligada**, mesmo sem tráfego. Por isso, desligar instâncias ociosas e escolher o tamanho certo (*rightsizing*) pesam tanto quanto o modelo de compra.

#### On-Demand

É o modelo **padrão**: você liga a instância, paga um preço fixo **por segundo** enquanto ela está ligada (mínimo de 60 segundos, em Linux e Windows) e desliga quando quiser. Não há compromisso, pagamento antecipado nem desconto.

- **Como pensar:** é como alugar um carro por diária. É o mais caro por dia, mas você devolve quando quiser.
- **Quando usar:** cargas **imprevisíveis** ou **de curta duração**, testes, picos que ninguém sabe quando vão acontecer, e o período inicial de uma aplicação nova, antes de conhecer o uso real.
- **Ponto de atenção:** a instância cobra enquanto está **running**, mesmo sem nenhuma requisição. Uma instância parada (**stopped**) não cobra computação, mas os volumes EBS e os Elastic IPs continuam cobrando.

> **Paralelo com o Lambda:** o Lambda também é "on-demand", mas cobra por **execução** (requisições + milissegundos de duração). O EC2 On-Demand cobra pelo **tempo ligado**. Uma instância ociosa 80% do tempo paga 100% do tempo, e é aí que o serverless costuma ser mais barato para tráfego irregular.

#### Savings Plans

Você se compromete a gastar um **valor fixo por hora** (ex: **US$ 10/hora**) durante **1 ou 3 anos**. Em troca, todo uso que se encaixa no plano é cobrado com preço **reduzido**. Funciona assim:

- O uso, já com o preço reduzido, é **abatido** do compromisso a cada hora, automaticamente. Você não "atribui" o plano a nenhuma instância.
- Se em uma hora você usa **mais** do que o compromisso, o excedente é cobrado em **On-Demand**.
- Se usa **menos**, você paga os US$ 10/hora **mesmo assim**. O compromisso é pago, usado ou não.
- **Formas de pagamento:** **All Upfront** (tudo adiantado, maior desconto), **Partial Upfront** ou **No Upfront** (mensal, menor desconto). Isso vale também para as RIs.

Há tipos diferentes, com troca entre **flexibilidade** e **desconto**:

| Tipo | O que cobre | Flexibilidade | Desconto |
|---|---|---|---|
| **Compute Savings Plan** | EC2 de **qualquer família, tamanho, região, SO e tenancy**, mais **Fargate** e **Lambda** | Máxima: trocar de `m5` para `c7g`, mudar de região ou migrar para containers continua coberto | Menor (até ~66%) |
| **EC2 Instance Savings Plan** | EC2 de **uma família em uma região** (ex: `m7g` em us-east-1), qualquer tamanho, SO e AZ | Média: pode mudar o tamanho, mas não a família nem a região | Maior (até ~72%) |
| **SageMaker Savings Plan** | Instâncias do SageMaker | Só SageMaker | Até ~64% |

- **Como pensar:** é um plano de celular com franquia mensal. Você paga a franquia de qualquer jeito, e o que passar dela sai pelo preço normal.
- **Como dimensionar:** o **Cost Explorer** recomenda o valor do compromisso com base no uso histórico. A prática comum é cobrir a **base estável** do consumo com o plano, e não o pico.

> **Paralelo com o Lambda:** se você usa Lambda, o **Compute Savings Plan** é o único jeito de ter desconto em Lambda, e o mesmo compromisso vale para EC2 e Fargate. Uma empresa migrando de EC2 para serverless não perde o desconto ao trocar de serviço.

#### Reserved Instances (RIs)

É o modelo **anterior** aos Savings Plans. Você se compromete com uma **configuração específica de instância** (tipo, região, SO e tenancy) por **1 ou 3 anos**. Apesar do nome, a RI **não é uma instância**: é um **desconto de cobrança** que a AWS aplica automaticamente a qualquer instância em execução que **coincida** com a configuração reservada. Se nenhuma instância coincidir, o desconto é desperdiçado, mas você paga a RI mesmo assim.

As RIs variam em duas dimensões:

**Classe**, ou seja, o quanto ela pode mudar depois da compra:

- **Standard:** maior desconto. Não pode trocar de família, mas pode ser **vendida** no **Reserved Instance Marketplace** se você não precisar mais dela.
- **Convertible:** desconto menor. Pode ser **trocada** por outra RI de família, SO ou tenancy diferente, desde que de valor igual ou maior. Não pode ser vendida.

**Escopo**, ou seja, onde o desconto vale:

- **Regional:** o desconto vale em **qualquer AZ** da região e, para Linux, em **qualquer tamanho** da mesma família (*size flexibility*: uma RI de `m5.xlarge` cobre duas `m5.large`). **Não reserva capacidade.**
- **Zonal:** o desconto vale só em **uma AZ** e no tamanho exato. Em troca, **reserva capacidade** nessa AZ.

Hoje, a AWS recomenda Savings Plans no lugar das RIs para EC2, porque dão o mesmo desconto com menos gestão. As RIs continuam relevantes em dois casos: para **reservar capacidade** (RI zonal) e para serviços que **não têm Savings Plan**, como **RDS**, **ElastiCache**, **OpenSearch** e **Redshift**.

#### Spot

Você usa a **capacidade ociosa** da AWS com desconto de até ~90%, mas a AWS pode **retomar** a instância com **2 minutos de aviso**. Não há compromisso: o preço varia com a oferta e a demanda, e você paga o preço Spot vigente. Os detalhes estão na seção [Spot Instances](#spot-instances).

- **Como pensar:** é comprar uma passagem de avião de última hora com desconto em um voo com lugares vagos. É barato, mas você pode ser remanejado se a companhia precisar do assento.

#### Dedicated Instances e Dedicated Hosts

Por padrão, a sua instância roda em um host físico **compartilhado** com instâncias de **outros clientes**. O isolamento entre elas é garantido pelo Nitro, e isso é seguro para quase todos os casos. Essa configuração se chama **tenancy** (*default*, *dedicated* ou *host*). Duas opções colocam a instância em hardware **exclusivo** da sua conta, mas resolvem problemas diferentes:

- **Dedicated Instances:** as suas instâncias rodam em hardware que **nenhum outro cliente** usa, mas você **não escolhe nem enxerga** o host. A AWS pode mover a instância para outro host dedicado (ex: depois de um stop/start). Resolve requisitos de **compliance** do tipo "não podemos compartilhar hardware com terceiros". Há uma **taxa regional adicional** por hora.
- **Dedicated Hosts:** você aluga o **servidor físico inteiro** e decide quais instâncias rodam nele. Você **enxerga** os **sockets e núcleos** físicos e pode fixar uma instância sempre no mesmo host (*host affinity*). A cobrança é **por host**, não por instância: dentro dele, você roda quantas instâncias couberem.
  - Resolve o **licenciamento BYOL** (*Bring Your Own License*) de softwares licenciados por **socket ou núcleo físico**, como Windows Server, SQL Server, Oracle e alguns softwares de virtualização. Com o host visível, você consegue provar quantos núcleos está usando.
  - O **AWS License Manager** controla o uso das licenças e pode alocar e liberar hosts automaticamente (*host resource groups*).

**Regra prática:** se a questão menciona **licença por socket/núcleo**, **BYOL** ou **visibilidade do hardware**, a resposta é **Dedicated Host**. Se menciona apenas "**não compartilhar hardware**", a resposta mais barata e simples é **Dedicated Instance**.

> **Obs:** Se a VPC for criada com **tenancy dedicated**, todas as instâncias lançadas nela serão dedicadas, mais caras, mesmo sem necessidade. A tenancy da VPC pode ser alterada de *dedicated* para *default* depois, mas não o contrário.

#### On-Demand Capacity Reservations (ODCR)

Garantem que haverá **capacidade disponível** para um tipo de instância em uma **AZ específica**, pelo tempo que você quiser. Isso protege contra o erro `InsufficientInstanceCapacity`, que pode acontecer quando muitos clientes pedem o mesmo tipo de instância na mesma AZ ao mesmo tempo (ex: Black Friday, ou um failover de DR em massa para a mesma região).

- **Não há prazo nem compromisso:** você cria a reserva quando quiser e cancela quando quiser.
- Você paga o **preço On-Demand** da capacidade reservada **enquanto a reserva existir**, com instâncias rodando nela ou não. Se você reserva 10 instâncias e usa 6, paga as 10.
- **Não dá desconto por si só**, mas pode ser **combinada** com Savings Plans ou RIs regionais, que reduzem o preço da capacidade reservada.
- Pode ser **compartilhada entre contas** via **AWS RAM** e agrupada em **resource groups**.
- **Como pensar:** é reservar uma mesa no restaurante. A mesa fica guardada para você, mas a reserva tem custo mesmo que você chegue com menos pessoas.

> **Paralelo com o Lambda:** é parecido com a **provisioned concurrency**: você paga para manter capacidade **pronta**, usada ou não, em troca de ter certeza de que ela estará lá quando a demanda chegar.

#### Capacity Blocks for ML

Permitem **reservar instâncias com GPU** (ex: família P, com GPUs NVIDIA de alto desempenho) por um **período definido**, de alguns dias a algumas semanas, com data de início **no futuro**. As instâncias ficam em **UltraClusters**, com rede de baixa latência (EFA) entre elas, prontas para **treinamento distribuído de modelos de ML**. O pagamento é feito **antecipadamente** na reserva.

- **Por que existe:** GPUs de ponta são escassas, e um treinamento de 2 semanas não justifica um compromisso de 1 ano. Com o Capacity Block, você tem a garantia das GPUs só pelo período do treinamento.

#### Resumo comparativo

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

A AWS precisa manter capacidade sobrando em cada AZ para atender picos de demanda dos clientes On-Demand. Em vez de deixar esses servidores parados, ela os **vende com grande desconto** como instâncias **Spot**. O preço baixo vem com uma condição: quando a AWS precisar dessa capacidade de volta, ela **interrompe** a sua instância. Por isso, Spot é ideal para trabalho que pode ser **pausado, repetido ou distribuído**, como processar um lote de arquivos, renderizar vídeos ou rodar testes. Não serve para algo que precisa ficar no ar sem interrupção, como um banco de dados único.

Cada combinação de **tipo de instância + AZ** forma um **pool de capacidade Spot** separado. Se um pool fica sem capacidade, as instâncias dele são interrompidas, mas os outros pools não são afetados. Daí vem a principal boa prática: **diversificar** entre muitos pools.

> **Paralelo com o Lambda:** pense em uma carga que você faria com Lambda + SQS. Ela é **idempotente**, processa mensagens de uma fila e, se uma execução falhar, a mensagem volta para a fila e outra execução a processa. Esse mesmo desenho é o que torna uma carga **segura para Spot**: um worker interrompido apenas deixa a mensagem voltar para a fila. Se o processamento passar de 15 minutos ou precisar de GPU, workers em Spot consumindo a fila são a alternativa natural ao Lambda.

- A AWS pode **recuperar** uma instância Spot quando precisa da capacidade, com um aviso de **2 minutos** (*Spot Instance Interruption Warning*), entregue via **instance metadata** e **EventBridge**.
- A **EC2 Instance Rebalance Recommendation** avisa **antes**, quando a instância está com **risco elevado** de interrupção, dando mais tempo para reagir.
- O comportamento na interrupção pode ser **terminate** (padrão), **stop** ou **hibernate**.
- O preço Spot muda **gradualmente** conforme a oferta e a demanda de longo prazo. Não há mais leilão: paga-se o preço Spot vigente, limitado opcionalmente por um preço máximo.

O diagrama abaixo mostra o processo completo. O ASG pede capacidade em vários pools (①) e a estratégia de alocação escolhe os mais disponíveis (②). A instância roda pagando o preço Spot (③). Quando o pool fica sob pressão, a AWS primeiro emite a **rebalance recommendation** (④), depois o **aviso de 2 minutos** (⑤) e, por fim, interrompe a instância (⑥). Os dois avisos chegam pelo **IMDS**, de dentro da instância, e pelo **EventBridge**, de fora dela. O ASG repõe a capacidade **antes** da interrupção, se o Capacity Rebalancing estiver ligado, ou **depois** dela.

![Diagrama Amazon EC2 - Ciclo de vida de uma instância Spot](./img/amazon-ec2/amazon-ec2-06.png)

**Boas práticas:**

- Ser **flexível**: usar **vários tipos de instância** e **várias AZs** para acessar mais **pools de capacidade Spot**. Ferramenta útil: **attribute-based instance type selection** (escolher por vCPU e memória, em vez de listar tipos).
- Usar a allocation strategy **`price-capacity-optimized`** (recomendada): escolhe os pools com **maior capacidade disponível** e menor preço, reduzindo interrupções.
- Combinar **On-Demand** (capacidade base) e **Spot** (escala) no mesmo Auto Scaling group com uma **mixed instances policy**.
- Habilitar o **Capacity Rebalancing** no ASG: ao receber a rebalance recommendation, o ASG lança uma substituta **antes** da interrupção.
- Projetar a aplicação para ser **stateless** ou fazer **checkpoints** do processamento (S3, DynamoDB).

**Spot Fleet vs. EC2 Fleet vs. Auto Scaling group:** o **Auto Scaling group** com mixed instances policy é a forma recomendada para a maioria dos casos. **EC2 Fleet** atende a lançamentos únicos (`instant`) ou mantém uma capacidade-alvo combinando On-Demand e Spot; o **Spot Fleet** é a API mais antiga.

![Diagrama Amazon EC2 - Spot com Auto Scaling](./img/amazon-ec2/amazon-ec2-02.png)

### Placement groups

Dentro de uma AZ, os servidores físicos ficam organizados em **racks**: armários que compartilham a mesma alimentação de energia e o mesmo switch de rede. Normalmente, a AWS decide sozinha em qual host e rack cada instância vai rodar. Os **placement groups** permitem influenciar essa decisão. Há duas necessidades opostas:

- **Desempenho:** instâncias **fisicamente próximas** conversam mais rápido, porque há menos saltos de rede entre elas.
- **Resiliência:** instâncias **fisicamente separadas** não caem juntas, porque uma falha de energia ou de switch em um rack só afeta as instâncias daquele rack.

Os placement groups controlam **como as instâncias são distribuídas** nesse hardware físico. A estratégia **partition** é um meio-termo entre as duas necessidades.

> **Paralelo com o Lambda:** no serverless, não existe esse controle, e nem é preciso: a AWS espalha as execuções entre AZs e hosts por você. No EC2, você só pensa nisso em cenários específicos: HPC, que precisa de proximidade, ou poucas instâncias críticas, que precisam de separação.

| Estratégia | Como distribui | Uso | Limitações |
|---|---|---|---|
| **Cluster** | Instâncias **próximas**, no mesmo segmento de rede de alta bisseção em **uma AZ** | **HPC**, cargas com **baixa latência** e **alto throughput** entre nós | Uma falha de hardware/rack pode afetar **todas**. Recomenda-se o mesmo tipo de instância e lançar todas de uma vez (evita *insufficient capacity*) |
| **Spread** | Cada instância em um **rack diferente** (hardware, rede e energia separados) | **Poucas instâncias críticas** que não podem falhar juntas | Máximo de **7 instâncias por AZ** por grupo. Pode abranger **várias AZs** |
| **Partition** | Grupos de instâncias (**partições**) em racks separados | Sistemas distribuídos com consciência de topologia: **HDFS, HBase, Cassandra, Kafka** | Até **7 partições por AZ**, com **centenas** de instâncias. A instância sabe sua partição pelos **metadados** |

![Diagrama Amazon EC2 - Placement groups](./img/amazon-ec2/amazon-ec2-01.png)

> **Obs:** Em caso de *insufficient capacity* ao adicionar instâncias a um **cluster placement group**, a recomendação é **parar e iniciar** todas as instâncias do grupo (para que sejam realocadas juntas) ou tentar novamente mais tarde.

### Rede

Toda instância se conecta à VPC por pelo menos uma **placa de rede virtual** (a ENI), que carrega os endereços IP e os security groups. Os demais recursos desta seção tratam de dois temas:

- **endereçamento:** como manter um IP fixo ou movê-lo entre instâncias;
- **desempenho:** quanto tráfego a instância consegue enviar e receber, e com que latência.

Os recursos de desempenho só importam para cargas que trafegam **muitos dados entre instâncias**, como HPC e ML distribuído.

> **Paralelo com o Lambda:** quando você coloca uma função Lambda **em uma VPC**, ela também recebe ENIs (as *Hyperplane ENIs*, gerenciadas pela AWS) nas subnets escolhidas. No EC2, a ENI é um recurso que **você** gerencia, e por isso pode ser movida entre instâncias.

- **Elastic Network Interface (ENI):** placa de rede virtual com IPs privados, IP público/Elastic IP, MAC e security groups. Pode ser **movida** entre instâncias da mesma AZ, o que permite **failover** rápido de IP (padrão comum em appliances e licenças atreladas ao MAC).
- **Elastic IP:** IPv4 público **fixo**, associado à conta até ser liberado. Desde 2024, **todo IPv4 público é cobrado**, inclusive Elastic IPs em uso.
- **Enhanced networking (ENA):** o recurso de rede de alto desempenho das instâncias, implementado pela placa **ENA** (*Elastic Network Adapter*). Usa **SR-IOV** (a instância acessa a placa de rede física quase diretamente, sem passar pelo hypervisor a cada pacote) para maior **largura de banda** (até centenas de Gbps nas instâncias maiores), mais **PPS** e menor latência. Habilitado por padrão nas instâncias Nitro.
- **Elastic Fabric Adapter (EFA):** ENA com **OS-bypass** (no Linux), em que a aplicação fala direto com a placa de rede, **sem passar pelo kernel** do sistema operacional. Isso permite que aplicações **HPC** e de **ML distribuído** se comuniquem com latência muito baixa. Essas aplicações usam bibliotecas como **MPI** e **NCCL**, que coordenam muitos processos rodando em paralelo em várias máquinas. Usado com **cluster placement groups**.
- **Jumbo frames:** o **MTU** é o tamanho máximo de um pacote de rede. Pacotes maiores significam menos pacotes para transferir o mesmo volume de dados, o que aumenta o throughput. O MTU é de **9001** dentro da VPC, no VPC peering **na mesma região** e no Direct Connect (VIF privada). O Transit Gateway suporta **8500**. Tráfego para a **Internet**, peering **entre regiões** e **Site-to-Site VPN** usam **1500** ou menos. Quando o pacote passa por um caminho com MTU menor, ele precisa ser fragmentado ou descartado, o que reduz o desempenho.

O diagrama resume a seção:
1. **ENI:** failover movendo uma ENI secundária entre instâncias;
2. **Caminho do pacote:** as camadas que cada tecnologia contorna. O ENA pula o hypervisor, e o EFA pula também o kernel;
3. **MTU:** o valor máximo de cada caminho de rede.

![Diagrama Amazon EC2 - Rede: ENI, ENA, EFA e MTU](./img/amazon-ec2/amazon-ec2-07.png)

### Armazenamento

Uma instância pode usar dois tipos de disco, e a diferença entre eles é **onde o disco está fisicamente**:

- **EBS:** fica em **outro lugar** da AZ e é acessado **pela rede**, como um "pendrive de rede". Por isso sobrevive quando a instância é desligada e pode ser desanexado e anexado a outra instância.
- **Instance store:** está **fisicamente dentro do host** onde a instância roda. Isso o torna muito rápido, mas quando a instância sai daquele host (stop, falha, término), os dados ficam para trás e são apagados.

A regra prática: dados que **precisam persistir** vão para o **EBS**; dados **temporários** ou que **já são replicados** pela aplicação podem usar o **instance store**.

> **Paralelo com o Lambda:** o **instance store** se comporta como o `/tmp` do Lambda: rápido, local e descartável. A diferença no EC2 é que existe também um disco **persistente** (o EBS), que no Lambda não existe. No serverless, o estado sempre vai para fora da função, em S3, DynamoDB ou EFS. Essa continua sendo a melhor prática no EC2, porque deixa as instâncias **descartáveis** e fáceis de escalar.

- **EBS:** volumes de **rede**, persistentes, presos a **uma AZ** (snapshots no S3 permitem copiar entre AZs e regiões). Detalhes na seção de storage.
- **Instance store:** discos **NVMe locais** ao host físico, com **altíssimo IOPS** e sem custo extra, mas **efêmeros**: os dados se perdem ao **parar, hibernar ou terminar** a instância ou em falha do host. Usado para cache, buffers, dados temporários e sistemas que **replicam** os dados entre nós (ex: Cassandra, Kafka).
- **EBS-optimized:** largura de banda dedicada entre a instância e o EBS (padrão nas instâncias atuais).
- **DeleteOnTermination:** atributo de cada volume EBS que define se ele é apagado quando a instância é terminada. É **verdadeiro** por padrão para o volume **raiz** e **falso** para volumes anexados depois.

O diagrama mostra:
1. **Onde cada disco fica:** o instance store dentro do host, o EBS na rede da AZ e os snapshots, que são regionais e permitem restaurar o volume em outra AZ;
2. **O que acontece com os dados** em cada evento do ciclo de vida;
3. **Qual usar:** instance store, EBS ou EFS/FSx.

![Diagrama Amazon EC2 - Armazenamento: instance store, EBS e snapshots](./img/amazon-ec2/amazon-ec2-08.png)

### AMIs, launch templates e user data

Para lançar uma instância, a AWS precisa saber duas coisas:

- **O que vai no disco:** sistema operacional e software. Isso é a **AMI**, uma "foto" de um disco pronta para ser copiada em cada nova instância.
- **Com que configuração a instância vai rodar:** tipo de instância, rede, security groups, role IAM. Isso fica no **launch template**, uma "receita" reutilizável que o Auto Scaling usa para criar instâncias idênticas.

O **user data** é um script opcional que roda no **primeiro boot** para terminar a configuração. Por exemplo, instalar a versão mais recente da aplicação. O ponto de decisão na prova é **quanto deixar pronto na AMI** e **quanto fazer no boot**.

> **Paralelo com o Lambda:**
> - A **AMI** equivale à **container image** da função: um artefato imutável e versionado com tudo o que é preciso para rodar.
> - O **launch template** equivale à configuração da função: memória, VPC, role e variáveis. Cada alteração gera uma **nova versão**, como as versões publicadas do Lambda.
> - O **user data** lembra o código de **inicialização** fora do handler: roda uma vez quando o ambiente sobe. Quanto mais ele faz, maior o "cold start" da instância.

- **AMI (Amazon Machine Image):** imagem com o SO, a configuração e o software. É **regional**: para usar em outra região, é preciso **copiar** a AMI. Pode ser **compartilhada** com outras contas ou com a Organization.
  - Para compartilhar uma AMI **criptografada**, a chave KMS precisa ser uma **customer managed key** compartilhada com a conta de destino (chaves AWS managed não podem ser compartilhadas). A conta de destino normalmente **copia** a AMI e a recriptografa com sua própria chave.
  - **EC2 Image Builder** automatiza a criação, o teste e a distribuição de AMIs (golden images) entre contas e regiões.
  - **Allowed AMIs** (configuração da conta, aplicável via **declarative policy**) restringe quais AMIs podem ser usadas.
- **Golden AMI vs. user data:** a AMI pré-configurada reduz o **tempo de boot** (bom para escalar rápido); o **user data** (script executado no primeiro boot) é mais flexível, mas aumenta o tempo até a instância ficar pronta. Muitas soluções combinam os dois.
- **Launch templates:** definem a configuração de lançamento (AMI, tipo, rede, IAM, user data), com **versionamento**. São **obrigatórios** para recursos novos, como mixed instances policies no ASG. As antigas **launch configurations** estão **descontinuadas**.
- **Versões e atualização:** cada alteração no launch template gera uma **nova versão**, e o ASG aponta para uma versão específica, para a `$Default` ou para a `$Latest`. Instâncias que já estão rodando **não mudam** quando o template muda. Para trocá-las pela nova versão, por exemplo com uma golden AMI atualizada, usa-se o **instance refresh** do ASG. Ele substitui as instâncias aos poucos, respeitando um percentual mínimo de instâncias saudáveis. O **rollback** é apontar o ASG para a versão anterior e fazer um novo refresh.

O diagrama mostra:
1. **O fluxo completo:** pipeline do Image Builder → golden AMI criptografada → cópia para outra região e compartilhamento com contas, sob controle do Allowed AMIs → launch template → ASG;
2. **Quanto deixar pronto na AMI** e quanto fazer no boot;
3. **Versionamento** do launch template com instance refresh.

![Diagrama Amazon EC2 - AMIs e launch templates](./img/amazon-ec2/amazon-ec2-09.png)

### Instance metadata (IMDS) e IAM

Uma aplicação rodando no EC2 frequentemente precisa chamar outros serviços da AWS, como ler um arquivo no S3 ou gravar no DynamoDB. Para isso, precisa de **credenciais**. A forma errada é gravar access keys em arquivos ou variáveis de ambiente: elas não expiram e podem vazar. A forma certa é associar uma **IAM role** à instância, para que a aplicação receba **credenciais temporárias** automaticamente.

Essas credenciais são entregues pelo **Instance Metadata Service (IMDS)**, um endereço interno (`169.254.169.254`) que só a própria instância consegue acessar. Ele funciona como um "balcão de informações" da instância: responde qual é o instance ID, a AZ, o IP e as credenciais da role. Os SDKs da AWS consultam esse endereço sozinhos, sem código extra.

> **Paralelo com o Lambda:** é exatamente o papel da **execution role**. No Lambda, as credenciais temporárias da role chegam como **variáveis de ambiente** (`AWS_ACCESS_KEY_ID`, `AWS_SESSION_TOKEN`...), e o SDK as lê sem você perceber. No EC2, o **instance profile** faz a mesma coisa, mas o SDK busca as credenciais no IMDS. Para o código, não muda nada: a mesma chamada `s3.getObject()` funciona nos dois.

O risco é que, se um atacante conseguir fazer a aplicação acessar uma URL arbitrária, ele pode ler as credenciais pelo IMDS. Esse ataque se chama **SSRF** (*Server-Side Request Forgery*): o atacante manipula um parâmetro para o servidor fazer a requisição por ele. O **IMDSv2** existe para bloquear esse caminho.

- **Instance profile:** associa uma **IAM role** à instância. As aplicações obtêm **credenciais temporárias**, rotacionadas automaticamente, pelo **IMDS** (`169.254.169.254`). Nunca guardar access keys na instância.
- **IMDSv2:** exige um **token de sessão** obtido com um `PUT` antes de ler os metadados, protegendo contra ataques de **SSRF**. A maioria das vulnerabilidades de SSRF só permite um `GET` simples, sem controlar o método nem os headers, então o atacante não consegue obter o token. O **hop limit** (TTL) é o número de "saltos" de rede que a resposta pode dar. Com valor **1**, a resposta não passa da própria instância e não chega a containers rodando em redes virtuais dentro dela.
- Para **exigir o IMDSv2** em toda a organização: **declarative policy** (EC2), **SCP** com a condition `ec2:MetadataHttpTokens`, ou a configuração padrão da conta.

![Diagrama Amazon EC2 - Instance profile e IMDSv2](./img/amazon-ec2/amazon-ec2-03.png)

### Ciclo de vida e recuperação

Uma instância passa por estados: **pending** (iniciando), **running**, **stopping/stopped** (desligada, sem cobrança de computação, mas os volumes EBS continuam cobrados) e **terminated** (apagada definitivamente). O ponto que mais confunde é o que acontece com o **hardware**. Ao **desligar e ligar** a instância, ela quase sempre volta em **outro host físico**. Já um **reboot** mantém o mesmo host.

Isso explica três comportamentos:

- os dados do instance store se perdem no stop;
- o stop/start resolve problemas de hardware;
- a AWS consegue "recuperar" uma instância movendo-a para um host saudável.

> **Paralelo com o Lambda:** no Lambda, se o host de um ambiente de execução falha, a AWS simplesmente cria outro, e você nem fica sabendo. No EC2, a instância é um recurso **de longa duração**, e lidar com falhas de hardware é parte do seu desenho. Por isso, a prática recomendada é tratar instâncias como **descartáveis** ("gado, não animal de estimação"): um **Auto Scaling group** detecta a instância não saudável e cria outra, e o estado fica fora dela. É o modelo mental mais próximo do serverless.

- **Stop/start:** a instância normalmente vai para **outro host físico** (útil para fugir de um host degradado). O IP público muda (o Elastic IP não), e os dados do instance store são perdidos.
- **Hibernate:** grava a **RAM** no volume EBS raiz (que precisa ser **criptografado** e ter espaço suficiente) e retoma os processos de onde pararam. Útil para aplicações com **inicialização demorada** (cache em memória). Limite de **60 dias** hibernada.
- **Status checks:**
  - **System status check:** problemas na infraestrutura da AWS (host, rede, energia). Solução: **recuperar** a instância em outro host.
  - **Instance status check:** problemas na própria instância (SO, configuração de rede, memória). Solução: **reboot** ou correção pelo cliente.
  - **Attached EBS status check:** problemas nos volumes EBS anexados.
- **Recuperação automática:** a **simplified automatic recovery** (padrão em tipos compatíveis) ou um **alarme do CloudWatch** com a ação **recover** move a instância para um novo host mantendo **instance ID, IPs privados, Elastic IP e metadados**. Não funciona com **instance store**.
- **Termination protection** e **stop protection** evitam encerramentos acidentais (não impedem o término por Auto Scaling ou pelo SO).

### Acesso e monitoramento

Tradicionalmente, administrar um servidor Linux exige **SSH** (porta 22) e um Windows exige **RDP** (porta 3389). Para instâncias em subnets privadas, isso exigia também um **bastion host**, uma instância pública usada como "ponte". Cada porta aberta e cada chave SSH é uma superfície de ataque e algo a mais para gerenciar. A AWS oferece alternativas que **eliminam portas abertas e chaves de longa duração**, e a prova costuma preferi-las.

> **Paralelo com o Lambda:** no Lambda, você nunca "entra" no ambiente de execução. Tudo é observado por **logs e métricas** que vão para o CloudWatch automaticamente. No EC2, os logs da aplicação **não** vão para o CloudWatch por padrão: é preciso instalar o **CloudWatch agent**. Também é possível abrir um shell na máquina, o que é útil para diagnóstico, mas deve ser controlado e auditado.

- **Systems Manager Session Manager:** acesso ao shell **sem SSH/RDP**, **sem portas abertas** e **sem bastion host**, com auditoria no CloudTrail e logs da sessão no S3/CloudWatch. Exige o **SSM Agent** e acesso aos endpoints do SSM (Internet/NAT ou **VPC endpoints**).
- **EC2 Instance Connect:** chaves SSH temporárias enviadas via API. O **EC2 Instance Connect Endpoint** permite SSH/RDP para instâncias em **subnets privadas** sem bastion e sem IP público.
- **CloudWatch:** métricas a cada **5 minutos** (básico) ou **1 minuto** (detailed monitoring). **Memória** e **espaço em disco** não são coletados por padrão: exigem o **CloudWatch agent**.

### Quotas

A AWS limita quantos recursos cada conta pode usar por região. Isso protege contra gastos acidentais e ajuda a AWS a planejar a capacidade. Muitas migrações e eventos de escala falham não por falta de capacidade na AWS, mas porque a conta **atingiu a quota**.

> **Paralelo com o Lambda:** é o equivalente ao limite de **concurrent executions** por região (1.000 por padrão). No EC2, a quota é contada em **vCPUs** em execução.

- As quotas de **On-Demand** e de **Spot** são por **região** e medidas em **vCPUs** por família de instância (não em número de instâncias). Antes de eventos de escala ou migrações grandes, solicitar aumento via **Service Quotas**.

### Dicas para a prova

- "Carga **constante** por 1 ou 3 anos com **máxima flexibilidade** (trocar família, região, ou usar Fargate/Lambda)" → **Compute Savings Plan**.
- "Garantir **capacidade** em uma AZ para um evento ou para DR, **sem compromisso de longo prazo**" → **On-Demand Capacity Reservation**.
- "Cargas **tolerantes a interrupção** com **menor custo**" → **Spot** (vários tipos de instância e AZs, `price-capacity-optimized`).
- "Banco com licença **por núcleo/vCPU** que precisa de **muita memória** mas pouca CPU" → **Optimize CPUs** (reduzir `CoreCount` e/ou desligar o SMT), mantendo a memória da instância.
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

## High Performance Computing (HPC)

**HPC** (*High Performance Computing*, computação de alto desempenho) é usar **muitos computadores trabalhando juntos** para resolver um problema grande demais para uma máquina só. O problema é dividido em milhares de pedaços, cada núcleo calcula um pedaço, e os resultados são combinados no final. Exemplos: previsão do tempo, simulação de colisão de carros, aerodinâmica (**CFD**, *Computational Fluid Dynamics*), sequenciamento genômico, descoberta de medicamentos, análise sísmica, risco financeiro e renderização de filmes.

HPC não é um serviço da AWS, e sim um **tipo de carga de trabalho** que combina vários serviços: instâncias EC2 otimizadas, rede de baixa latência (EFA), sistemas de arquivos paralelos (FSx for Lustre) e orquestradores de jobs (ParallelCluster, PCS, Batch). On-premises, um cluster HPC é caro, fica **ocioso** entre projetos e cria **filas de espera** nos picos. Na AWS, o cluster cresce para milhares de núcleos quando há jobs na fila e volta a **zero** quando a fila esvazia.

> **Paralelo com o Lambda:** você provavelmente já faz uma forma de HPC: dividir um trabalho grande em muitos pedaços **independentes** e processá-los em paralelo, com Lambda + SQS ou com o **Step Functions Distributed Map**. Isso é HPC **fracamente acoplado**. O que o serverless não atende são as cargas em que os pedaços precisam **conversar entre si o tempo todo**, que exigem máquinas fixas, fisicamente próximas e com rede especial. Também não atende tarefas que passam de **15 minutos** ou que precisam de **GPU**.

### Como funciona um cluster HPC

Antes dos serviços, vale entender como um cluster HPC **tradicional** funciona. Eles existem há décadas em universidades, centros de pesquisa e indústrias, e os serviços da AWS reproduzem exatamente essa estrutura. Um cluster tem cinco componentes:

1. **Login/head node:** a "porta de entrada". Os usuários se conectam a ele por SSH, preparam os arquivos e **submetem jobs**. Ninguém roda cálculos pesados ali.
2. **Job scheduler** (agendador): roda no head node e funciona como um **maître de restaurante**. Recebe os pedidos (jobs), coloca-os em **fila**, decide a ordem de acordo com prioridades e cotas de cada grupo, e reserva os nós para cada job. O mais usado é o **Slurm**.
3. **Compute nodes:** dezenas a milhares de servidores **idênticos**, que só executam jobs. Cada job recebe nós **exclusivos** enquanto roda.
4. **Interconnect:** a **rede especial** que liga os compute nodes, com latência muito menor do que a de uma rede comum. On-premises, a tecnologia clássica é o **InfiniBand**; na AWS, é o **EFA**.
5. **Sistema de arquivos paralelo:** um storage **compartilhado** que todos os nós enxergam como uma pasta comum e que aguenta milhares de leituras e escritas simultâneas. O clássico é o **Lustre**; na AWS, é o **FSx for Lustre**.

**O ciclo de vida de um job:**

1. O usuário escreve um **script de job**, que diz o que rodar e quantos recursos usar (ex: 16 nós, 64 núcleos cada, por até 12 horas), e o submete.
2. O job entra na **fila** e espera.
3. O agendador **reserva** os nós quando há recursos livres e chega a vez do job.
4. O programa roda em todos os nós ao mesmo tempo, lendo e gravando no sistema de arquivos compartilhado e trocando mensagens pelo interconnect.
5. Ao terminar, os nós são **liberados** para o próximo job.

**O que muda na nuvem:** on-premises, o número de nós é **fixo**. Quando a fila cresce, os jobs esperam dias; quando ela esvazia, máquinas caras ficam paradas. Na AWS, o passo 3 muda: em vez de esperar um nó livre, o agendador **cria** as instâncias necessárias e as **destrói** quando o job termina. A fila praticamente desaparece, e o custo acompanha o uso.

![Diagrama HPC - Anatomia de um cluster HPC e equivalentes na AWS](./img/aws-hpc/aws-hpc-04.png)

| Componente do cluster | Equivalente na AWS |
|---|---|
| Login/head node + scheduler | Instância EC2 com Slurm (**ParallelCluster**), controlador gerenciado (**PCS**) ou fila gerenciada (**Batch**) |
| Compute nodes | Instâncias EC2 (**Hpc**, C, P/G), criadas e destruídas conforme a fila |
| Interconnect (InfiniBand) | **EFA** + **cluster placement group** |
| Sistema de arquivos paralelo (Lustre) | **FSx for Lustre**, ligado ao **S3** |
| Arquivos de usuário (home) | **EFS** ou FSx for NetApp ONTAP/OpenZFS |
| Estação de visualização | Instância com GPU + **Amazon DCV** |

**Por que levar HPC para a nuvem, e o que pesa contra:**

- **A favor:** sem fila de espera, hardware sempre atualizado, nenhum investimento inicial (CapEx), capacidade para picos (ex: entrega de projeto) e a possibilidade de rodar **vários experimentos em paralelo** em clusters separados.
- **Contra, ou o que exige planejamento:**
  - **gravidade dos dados**: mover petabytes leva tempo e custa;
  - **licenças** de software comercial que dependem de um servidor de licenças on-premises;
  - cargas que rodam **24×7** a 100% de uso, para as quais um cluster próprio já amortizado pode ser mais barato.

[↑ Índice](#índice)

### Conceitos básicos

#### Como um problema é dividido

Imagine simular o ar passando em volta de um carro. O espaço ao redor dele é dividido em uma **malha** com milhões de pequenas células. Em cada **passo de tempo** (ex: um milésimo de segundo simulado), o programa calcula a pressão e a velocidade do ar em cada célula, com base nas **células vizinhas**. Essa conta é feita milhares de vezes até cobrir alguns segundos de simulação.

Uma máquina só levaria semanas. Por isso, a malha é dividida em pedaços (**decomposição de domínio**), e cada processo, em um núcleo, cuida de um pedaço. Surge então um problema: as células na **borda** de cada pedaço dependem de células que estão **no pedaço de outro nó**. Por isso, cada passo da simulação segue sempre o mesmo ciclo:

1. **Calcular:** cada processo atualiza as células do seu pedaço.
2. **Trocar bordas:** cada processo envia aos vizinhos os valores da sua borda e recebe os deles (*halo exchange*). Essa troca é feita com **MPI**.
3. **Sincronizar:** ninguém começa o próximo passo até **todos** terminarem a troca. Isso é uma **barreira**.

O ponto crítico está no passo 3: **o processo mais lento dita o ritmo de todos**. Se um nó demora mais para receber uma mensagem, os outros milhares de núcleos ficam **parados esperando**. Por isso, em HPC fortemente acoplado, a rede precisa de latência **baixa** e também **consistente**, todos os nós devem ser **iguais** e não pode haver "vizinho barulhento" disputando recursos. Isso explica as escolhas de arquitetura: núcleos físicos dedicados (SMT desligado), cluster placement group, EFA, uma única AZ e um único tipo de instância.

Já em uma carga **fracamente acoplada**, como renderizar frames de um filme, cada tarefa é independente: não há troca de bordas nem barreira. Um nó lento atrasa só a própria tarefa. Por isso essas cargas aceitam Spot, várias AZs e tipos de instância misturados.

![Diagrama HPC - Decomposição de domínio, troca de bordas e sincronização](./img/aws-hpc/aws-hpc-05.png)

#### Escalabilidade: mais nós nem sempre é mais rápido

Dobrar o número de nós dobra a capacidade de cálculo, mas cada nó fica com um pedaço **menor** da malha. A conta de cada passo fica mais rápida, e a troca de bordas **não diminui na mesma proporção**. A partir de certo ponto, o tempo de comunicação domina e adicionar nós **não acelera mais**, ou até piora. É a [Lei de Amdahl](https://pt.wikipedia.org/wiki/Lei_de_Amdahl) aplicada à rede. Duas formas de medir:

- **Strong scaling:** o **mesmo problema** com mais nós. Mede quanto o resultado sai mais rápido.
- **Weak scaling:** um **problema maior** (malha mais fina) com mais nós, mantendo o mesmo trabalho por nó. Mede se dá para resolver problemas maiores no mesmo tempo.

Na nuvem, isso vira uma decisão de **custo**. Um job em 64 nós por 1 hora custa o mesmo que em 128 nós por 30 minutos, **se** a escala for perfeita. Se não for, os nós extras são dinheiro gasto em núcleos esperando a rede. Testar o ponto de escala ideal antes de rodar a produção é uma boa prática.

#### Fortemente vs. fracamente acoplado

A distinção mais importante em HPC é entre cargas **fortemente** e **fracamente acopladas**, porque ela define toda a arquitetura:

| | **Fortemente acoplado** (*tightly coupled*) | **Fracamente acoplado** (*loosely coupled*, *embarrassingly parallel*) |
|---|---|---|
| Como funciona | Os nós **trocam dados o tempo todo** durante o cálculo: cada pedaço depende dos vizinhos | Cada tarefa é **independente** e não conversa com as outras |
| Exemplo | Simulação de clima ou de CFD: cada região do modelo depende das vizinhas a cada passo | Renderizar 10.000 frames, analisar 1 milhão de genomas, simulação de Monte Carlo |
| Gargalo | **Latência de rede** entre os nós | Quantidade de CPU disponível e vazão de leitura dos dados |
| Comunicação | **MPI** | Nenhuma: uma fila de tarefas basta |
| Se um nó cai | O job **inteiro** para (ou volta ao último checkpoint) | Só aquela tarefa é refeita |
| Arquitetura na AWS | **Cluster placement group** + **EFA** + **FSx for Lustre** em **uma AZ**, com ParallelCluster ou PCS | **AWS Batch** com **Spot**, em **várias AZs** e com vários tipos de instância |

#### Glossário

- **MPI** (*Message Passing Interface*): o padrão de biblioteca que os programas HPC usam para trocar mensagens entre processos rodando em máquinas diferentes. É o "idioma" das cargas fortemente acopladas. Implementações comuns: Open MPI e Intel MPI. Em ML distribuído, o papel equivalente é da **NCCL** (NVIDIA), entre GPUs.
- **Rank:** cada processo de um programa MPI recebe um número, o *rank* (0, 1, 2...), usado para saber qual pedaço do problema é dele e com quem trocar mensagens. Normalmente há **um rank por núcleo físico**.
- **Core-hours** (núcleo-hora): a unidade de consumo em HPC. 1.000 núcleos por 1 hora = 1.000 core-hours. Centros de pesquisa distribuem cotas nessa unidade, e alguns softwares comerciais de simulação são **licenciados por núcleo**.
- **Interconnect:** a rede entre os compute nodes de um cluster. InfiniBand on-premises, EFA na AWS.
- **POSIX:** o padrão de sistema de arquivos do Unix/Linux: pastas, `open`/`read`/`write`, permissões e travas (locks). Programas HPC esperam um sistema de arquivos POSIX, e o S3 não é um.
- **Job:** uma unidade de trabalho submetida ao cluster, por exemplo "rodar esta simulação com 1.024 núcleos".
- **Job scheduler** (agendador): o software que recebe os jobs, coloca-os em **fila** e decide quando e em quais nós cada um roda. O mais usado em HPC é o **Slurm**.
- **Head node** (nó principal): a máquina que roda o agendador e onde os usuários entram para submeter jobs. Os **compute nodes** (nós de computação) executam os jobs.
- **Checkpoint:** um "save game" periódico do estado da simulação em disco. Se algo falhar, o job continua do último checkpoint em vez de recomeçar. É o que torna viável usar Spot em jobs longos.
- **Scratch:** armazenamento **temporário** e rápido, usado durante o job e descartado depois.

[↑ Índice](#índice)

### Computação

Escolher a máquina para HPC é diferente de escolher para uma aplicação web. Quatro características pesam mais do que o número de vCPUs:

- **Núcleos físicos dedicados:** cada rank MPI quer um núcleo inteiro. Duas threads SMT no mesmo núcleo disputam recursos e criam exatamente o "nó mais lento" que trava todos os outros.
- **Largura de banda de memória:** muitos códigos de simulação (CFD, clima) passam mais tempo **buscando dados na memória** do que calculando. São chamados de *memory-bound*. Para eles, colocar mais núcleos em um nó não ajuda se a memória não acompanha. Às vezes, usar **menos núcleos por nó** em mais nós é mais rápido.
- **Rede:** a largura de banda e o suporte a EFA definem o quanto o job escala para muitos nós.
- **GPU ou CPU:** códigos reescritos para GPU (dinâmica molecular, alguns solvers de CFD, ML) podem rodar muitas vezes mais rápido. Código que não foi portado para GPU não ganha nada com ela.

Há também o fator **licença**. Softwares comerciais de simulação costumam ser licenciados **por núcleo**. Nesse caso, compensa usar **menos núcleos, mais rápidos**, mesmo pagando mais pela instância.

- **Famílias de instância:**
  - **Hpc** (ex: `hpc7g` com Graviton, `hpc7a` com AMD, `hpc6id` com disco NVMe local): feitas para HPC fortemente acoplado. Têm muitos núcleos físicos, **SMT desligado**, EFA de alta largura de banda e o melhor preço por núcleo para essas cargas. Estão disponíveis em poucas regiões e AZs.
  - **C** (compute optimized): alternativa geral para HPC, com mais opções de tamanho e de região.
  - **P e G** (GPUs NVIDIA) e **Trn/Inf** (chips da AWS para ML): simulações aceleradas por GPU e treinamento de modelos.
  - **R e X** (memory optimized): simulações que precisam de muita memória por núcleo, como bancos genômicos e análise de elementos finitos.
- **SMT desligado:** em HPC, normalmente cada processo MPI deve ter um núcleo físico **só para ele**. Nas famílias que usam SMT, desliga-se com **Optimize CPUs** (`ThreadsPerCore = 1`). As Hpc e o Graviton já vêm assim. Ver [O que é uma vCPU](#o-que-é-uma-vcpu).
- **Modelo de compra:**
  - **Fracamente acoplado:** **Spot**, porque uma tarefa interrompida é apenas refeita.
  - **Fortemente acoplado:** **On-Demand**, porque perder um nó derruba o job inteiro. Para garantir que todos os nós vão subir juntos na mesma AZ, usar **On-Demand Capacity Reservations**. Spot só é viável com **checkpoints** frequentes.
  - **Cluster ligado o ano todo:** **Savings Plans**.
  - **GPUs para treinamentos de dias ou semanas:** **Capacity Blocks for ML**.
- **Quotas:** clusters HPC usam milhares de vCPUs de uma vez. Pedir aumento de quota antes de um projeto grande. Ver [Quotas](#quotas).

[↑ Índice](#índice)

### Rede: ENA e Elastic Fabric Adapter (EFA)

Uma rede tem duas medidas de desempenho, e é comum confundi-las:

- **Latência:** quanto tempo uma mensagem leva para **chegar** do outro lado, mesmo que seja minúscula. É o tempo de entrega de uma carta.
- **Largura de banda:** quanto dado passa **por segundo**. É o tamanho do caminhão.

Uma aplicação web ou um backup precisam de **largura de banda**: são poucos arquivos grandes. Já a troca de bordas em uma simulação gera **milhões de mensagens pequenas**, uma por vizinho a cada passo, e cada passo só termina quando a **última** mensagem chega (ver [Como um problema é dividido](#como-um-problema-é-dividido)). Por isso, em HPC fortemente acoplado, importam a **latência** e, principalmente, a sua **consistência**. Uma única mensagem atrasada (*tail latency*) deixa todos os núcleos esperando.

Em uma rede comum, cada mensagem passa pela pilha TCP/IP do kernel do sistema operacional, o que acrescenta atraso e variação. Clusters on-premises resolvem isso com **InfiniBand**, uma rede especializada em que a aplicação fala direto com a placa. Na AWS, há dois níveis de interface de rede, e é importante saber quando cada um basta.

#### Enhanced networking (ENA)

É o mesmo **enhanced networking** descrito na seção [Rede do EC2](#rede). *Enhanced networking* é o nome do recurso, e o **ENA** (*Elastic Network Adapter*) é a placa de rede virtual que o implementa. Ele vem habilitado por padrão em todas as instâncias atuais (Nitro). Com **SR-IOV**, a instância acessa a placa de rede quase diretamente, sem o hypervisor no caminho de cada pacote. Isso entrega muita largura de banda (de alguns Gbps nas instâncias pequenas a centenas de Gbps nas maiores), muitos pacotes por segundo e latência baixa **para uma rede TCP/IP**. Mas o tráfego continua passando pelo **kernel** do sistema operacional. O diagrama daquela seção mostra as camadas que o ENA e o EFA contornam.

Para HPC, o ENA é suficiente em vários casos:

- **Cargas fracamente acopladas:** as tarefas não conversam entre si, e a rede serve só para ler e gravar dados (S3, FSx).
- **Acesso ao storage:** a leitura e a escrita no FSx for Lustre passam pelo ENA, e é aqui que a **largura de banda** da instância importa.
- **Jobs MPI pequenos ou pouco sensíveis à latência:** poucos nós, ou códigos que trocam mensagens com pouca frequência.

**Limites de banda por fluxo:** a banda anunciada de uma instância (ex: 100 Gbps) é o **total** da placa, não o de uma única conexão. Um único **fluxo TCP** (uma conexão entre dois IPs e portas) é limitado a cerca de **5 Gbps**, ou cerca de **10 Gbps** dentro de um **cluster placement group**. Para usar toda a banda, a aplicação precisa abrir **várias conexões em paralelo**, como fazem o AWS CLI no S3, o DataSync e os clientes Lustre. Esse limite costuma aparecer em questões do tipo "a instância tem 100 Gbps, mas a cópia não passa de 5 Gbps".

**ENA Express:** um recurso do ENA que usa o protocolo **SRD** (o mesmo do EFA, descrito abaixo) para o tráfego **TCP e UDP comum**, sem mudar a aplicação. Ele aumenta a banda de um **único fluxo** (até cerca de 25 Gbps) e reduz a latência de cauda (*tail latency*). Funciona apenas entre instâncias compatíveis na **mesma AZ** e precisa ser habilitado nas duas pontas. É útil para aplicações que não usam MPI, mas sofrem com o limite por fluxo, como replicação de bancos de dados, transferência de arquivos grandes e storage distribuído.

#### EFA (Elastic Fabric Adapter)

O **EFA** é um ENA com uma capacidade a mais: o **OS-bypass**. Ele é a resposta da AWS ao InfiniBand e o nível necessário para MPI fortemente acoplado em escala:

- **OS-bypass:** a biblioteca MPI ou NCCL fala **direto com a placa de rede**, sem passar pelo kernel do sistema operacional. Isso corta boa parte da latência de cada mensagem. Funciona por meio da biblioteca **libfabric**, que as implementações de MPI já suportam.
- **SRD** (*Scalable Reliable Datagram*): o protocolo de transporte do EFA, criado pela AWS. Ele espalha os pacotes por **vários caminhos** da rede ao mesmo tempo e retransmite rapidamente o que se perder. Isso dá latência baixa e **consistente**, mesmo com milhares de nós.
- O EFA também funciona como uma **ENA comum** para o tráfego TCP/IP normal (SSH, acesso ao storage, internet).

| | **ENA** | **ENA Express** | **EFA** |
|---|---|---|---|
| O que é | Interface padrão (enhanced networking, SR-IOV) | ENA + transporte SRD para TCP/UDP | ENA + OS-bypass + SRD para MPI/NCCL |
| Passa pelo kernel? | Sim | Sim | **Não** (no tráfego MPI/NCCL) |
| Muda a aplicação? | Não | Não (só habilitar nas duas pontas) | Usa MPI/NCCL com libfabric |
| Alcance | Qualquer destino | Mesma AZ | **Mesma subnet** |
| Uso típico em HPC | Fracamente acoplado, acesso ao storage | Transferências grandes entre instâncias | MPI fortemente acoplado, ML distribuído |

**Restrições que caem na prova:**

- O tráfego **OS-bypass** do EFA **não é roteável**: ele só funciona entre instâncias na **mesma subnet**, e portanto na **mesma AZ**. Um cluster MPI que precisa de EFA não pode ser distribuído entre AZs.
- O **security group** precisa permitir **todo o tráfego de entrada e saída para ele mesmo** (uma regra que referencia o próprio security group). Sem isso, o EFA não funciona.
- Só alguns tipos de instância suportam EFA (em geral, os tamanhos maiores das famílias C, M, R, Hpc, P e Trn), e o EFA precisa ser **anexado no lançamento** da instância.
- Combinar com **cluster placement group**, para colocar os nós fisicamente próximos.

> **Obs:** Para HPC fortemente acoplado, **uma única AZ** é a resposta certa, mesmo que isso pareça contradizer as boas práticas de alta disponibilidade. A latência entre AZs prejudica o MPI, e o EFA nem funciona entre subnets. A resiliência vem de **checkpoints** no S3 ou no FSx e da possibilidade de **recriar o cluster** em outra AZ, e não de distribuir os nós.

[↑ Índice](#índice)

### Armazenamento para HPC

Um job HPC típico tem três fases de dados: **ler** um grande volume de entrada (malhas de simulação, genomas, imagens), **escrever** resultados intermediários e checkpoints durante a execução, e **gravar** a saída final. Milhares de núcleos lendo e escrevendo ao mesmo tempo esgotam rapidamente um storage comum. Por isso, HPC usa **sistemas de arquivos paralelos**, em que os dados ficam espalhados por muitos servidores e cada cliente lê de vários deles ao mesmo tempo.

**Por que um NFS comum não basta:** em um servidor de arquivos tradicional (NFS), **um único servidor** atende todos os clientes. Ele funciona bem para dezenas de máquinas, mas com mil nós lendo os mesmos arquivos de entrada no início de um job, esse servidor vira um **funil**. Os nós ficam esperando dados em vez de calcular.

**Como um sistema de arquivos paralelo resolve isso:** o Lustre separa duas funções e distribui os dados.

- **Servidor de metadados (MDS):** guarda os **nomes**, as pastas, as permissões e o **mapa** de onde está cada pedaço de cada arquivo. É consultado só ao abrir o arquivo.
- **Servidores de armazenamento (OSS), com seus discos (OSTs):** guardam os **dados** de fato.
- **Striping:** cada arquivo grande é **fatiado** em blocos (*stripes*) espalhados por vários servidores de armazenamento, como um RAID 0 em rede. Para ler um arquivo de 100 GB, o cliente busca as fatias em **vários servidores ao mesmo tempo**, e mil clientes se distribuem por todos eles.
- **Resultado:** a vazão total cresce com o número de servidores, e é por isso que o desempenho do FSx for Lustre **cresce com a capacidade provisionada**. Mais TB significa mais servidores por trás e mais GB/s.

![Diagrama HPC - NFS tradicional vs. sistema de arquivos paralelo (Lustre)](./img/aws-hpc/aws-hpc-06.png)

Os programas veem tudo isso como uma **pasta comum** (POSIX), por exemplo `/fsx/projeto/malha.dat`, sem saber que o arquivo está fatiado em dezenas de servidores.

**Amazon FSx for Lustre:** o Lustre é o sistema de arquivos paralelo mais usado em supercomputadores. O FSx for Lustre é a versão **gerenciada** dele, com throughput de centenas de GB/s, milhões de IOPS e latência abaixo de 1 ms. Os pontos-chave são:

- **Tipos de implantação:**
  - **Scratch:** sem replicação, mais barato, para dados **temporários** de jobs curtos. Se um servidor de arquivos falhar, os dados se perdem.
  - **Persistent:** os dados são **replicados dentro da AZ** e o servidor com falha é substituído automaticamente. Para armazenamento de longa duração ou jobs longos.
- **Integração com o S3 (*data repository association*):** o sistema de arquivos é **ligado a um bucket S3**. Os arquivos do bucket aparecem no Lustre como se estivessem lá, mas são carregados **sob demanda** no primeiro acesso (*lazy loading*). Os resultados podem ser **exportados** de volta para o S3, automaticamente ou com um comando. Assim, o S3 guarda os dados de forma durável e barata, e o Lustre é só a camada rápida enquanto o job roda. Um padrão comum é criar o FSx scratch no início do projeto e **apagá-lo** no fim.
- O FSx for Lustre fica em **uma única AZ**, a mesma do cluster.

| Storage | Papel em HPC |
|---|---|
| **Amazon S3** | Repositório **durável** de entradas e saídas. Barato, mas não é um sistema de arquivos POSIX: os programas HPC não o leem diretamente com bom desempenho |
| **FSx for Lustre** | Sistema de arquivos **paralelo** de alto desempenho para o processamento, ligado ao S3 |
| **Instance store NVMe** (ex: `hpc6id`) | Scratch **local** de cada nó: o mais rápido, mas efêmero e não compartilhado |
| **Amazon EFS** | Diretórios **home** dos usuários, scripts e software compartilhado. Não é feito para o throughput de um job HPC |
| **EBS** | Disco de sistema do head node e dos nós |

> **Paralelo com o Lambda:** o padrão "S3 como fonte da verdade + uma camada rápida temporária" é o mesmo de uma função que baixa um arquivo do S3 para o `/tmp`, processa e envia o resultado de volta ao S3. A diferença é que o FSx for Lustre faz isso de forma **compartilhada** entre milhares de núcleos, e de forma transparente para o programa.

[↑ Índice](#índice)

### Orquestração: ParallelCluster, PCS e Batch

Em um cluster compartilhado por dezenas de pesquisadores, alguém precisa decidir **qual job roda onde e quando**. Esse é o papel do **agendador** (*job scheduler*), que:

- mantém **filas** (no Slurm, *partitions*), por exemplo uma para nós com GPU e outra para nós com muita memória;
- aplica **prioridades** e cotas por grupo (*fair-share*), para que um usuário não monopolize o cluster;
- **reserva nós inteiros** para cada job, já que jobs MPI precisam de todos os nós ao mesmo tempo;
- faz ***backfill***: encaixa jobs pequenos nos "buracos" enquanto um job grande espera nós suficientes ficarem livres.

O usuário descreve o job em um script e o envia com `sbatch`:

```bash
#!/bin/bash
#SBATCH --job-name=simulacao-asa
#SBATCH --partition=hpc7g        # fila (tipo de nó)
#SBATCH --nodes=16               # 16 nós
#SBATCH --ntasks-per-node=64     # 64 ranks MPI por nó = 1.024 núcleos
#SBATCH --time=12:00:00          # tempo máximo
srun ./solver /fsx/projeto/malha.dat   # roda o programa MPI em todos os nós
```

Na nuvem, o agendador ganha mais uma função: **criar e destruir as instâncias** conforme a fila. Quando esse job chega, 16 instâncias `hpc7g` são lançadas; quando termina, são encerradas. A AWS oferece três caminhos para isso:

- **AWS ParallelCluster:** ferramenta **open source** (CLI e interface web) que cria um cluster HPC "tradicional" a partir de um arquivo **YAML**. Ela gera a infraestrutura com **CloudFormation**: head node, compute nodes, rede, storage compartilhado e agendador **Slurm**. Os compute nodes **escalam automaticamente**, de zero até o máximo, conforme a fila de jobs. É gratuita (paga-se só pelos recursos), mas **você** opera o cluster: atualizações, versões e o head node.
- **AWS Parallel Computing Service (PCS):** serviço **gerenciado** de Slurm. A AWS opera o controlador do Slurm (o equivalente ao head node), as atualizações e a disponibilidade. Você define **grupos de nós** de computação e **filas**. É o caminho para quem quer Slurm **sem gerenciar** a ferramenta.
- **AWS Batch:** serviço **gerenciado** de filas de jobs, **sem Slurm**. Você define **job definitions** (uma imagem de container + recursos), **job queues** e **compute environments** (EC2 On-Demand, **Spot** ou **Fargate**). O Batch escolhe as instâncias, executa os containers e escala até zero. Recursos importantes:
  - **Array jobs:** um único envio cria até **10.000 jobs filhos**. Cada um recebe um índice (`AWS_BATCH_JOB_ARRAY_INDEX`) para saber qual pedaço dos dados processar. É o padrão do fracamente acoplado.
  - **Multi-node parallel jobs:** um job que usa **vários nós ao mesmo tempo**, com suporte a MPI e EFA. Permite rodar cargas fortemente acopladas no Batch.
  - **Dependências** entre jobs e integração com **Step Functions** para montar pipelines (ex: alinhar genomas → chamar variantes → gerar relatório).

| Critério | ParallelCluster | PCS | Batch |
|---|---|---|---|
| Modelo | Ferramenta open source; você opera | Slurm gerenciado | Filas de jobs gerenciadas, com containers |
| Agendador | Slurm | Slurm | Próprio do Batch |
| Público típico | Equipes de HPC que já usam Slurm on-premises e querem controle total | Equipes de HPC que usam Slurm e querem menos operação | Desenvolvedores e pipelines de dados; cargas fracamente acopladas |
| Força | Flexibilidade e personalização | Menor esforço operacional com Slurm | Integração com containers, Spot e Step Functions |

> **Paralelo com o Lambda:** o **AWS Batch** é o mais próximo do serverless: você envia um container, não gerencia servidores e só paga enquanto os jobs rodam. Pense nele como um "Lambda para jobs longos". Ele não tem limite de 15 minutos, aceita GPU e muita memória, e tem uma fila nativa. Para tarefas curtas e muito numerosas, **Step Functions Distributed Map + Lambda** continua sendo uma opção, com até 10.000 execuções em paralelo.

[↑ Índice](#índice)

### Arquiteturas de referência

**HPC fortemente acoplado com Slurm:**

- **Acesso:** os usuários entram no **head node**, que fica na subnet pública ou atrás de um **Session Manager**/**DCV**, e submetem jobs ao Slurm.
- **Computação:** os **compute nodes** (Hpc ou C, com **EFA**) ficam em uma subnet privada, em um **cluster placement group**, e são criados conforme a fila.
- **Storage:** o **FSx for Lustre** fica ligado ao **S3**.
- **Orquestração:** tudo é criado pelo **ParallelCluster**, ou gerenciado pelo **PCS**.

![Diagrama HPC - Fortemente acoplado com ParallelCluster/PCS, EFA e FSx for Lustre](./img/aws-hpc/aws-hpc-01.png)

**HPC fracamente acoplado com AWS Batch:**

1. A chegada dos dados no S3, ou um agendamento, dispara um pipeline no **Step Functions** ou no **EventBridge**.
2. O pipeline envia um **array job** ao **Batch**.
3. O Batch distribui os jobs filhos em um compute environment **Spot** com **vários tipos de instância** e **várias AZs**.
4. Cada filho lê seu pedaço no S3 e grava o resultado de volta no S3.

![Diagrama HPC - Fracamente acoplado com AWS Batch e Spot](./img/aws-hpc/aws-hpc-02.png)

[↑ Índice](#índice)

### HPC híbrido e migração

Muitas empresas já têm um cluster HPC on-premises e não vão abandoná-lo. O cenário típico de prova é o **cloud bursting**: usar a AWS **só quando a fila local fica grande**, por exemplo antes de um prazo de projeto.

Levar jobs para a nuvem é a parte fácil. Os desafios de um ambiente híbrido estão em volta dele:

- **Dados:** os jobs precisam dos arquivos de entrada, que estão on-premises. É preciso decidir entre **copiar antes** (DataSync, Snowball) ou **acessar sob demanda** (File Cache), e trazer os resultados de volta.
- **Licenças:** softwares comerciais costumam consultar um **servidor de licenças** (ex: FlexLM) on-premises. Os nós na nuvem precisam alcançá-lo pela rede privada, ou o servidor precisa de uma cópia na AWS com uma ENI fixa (ver [Rede](#rede)).
- **Identidade:** os usuários e grupos Linux (UIDs/GIDs) precisam ser **os mesmos** nos dois lados, ou as permissões de arquivo quebram. A solução comum é integrar os nós da nuvem ao mesmo **Active Directory/LDAP**, via **AWS Directory Service** ou AD Connector.
- **Rede:** um link estável e com banda suficiente para dados e controle, ou seja, **Direct Connect**.

- **Rede:** **Direct Connect** (ou VPN) entre o data center e a VPC.
- **Movimentação de dados:**
  - **AWS DataSync:** copia dados do storage on-premises (NFS, SMB, HDFS) para S3, EFS ou FSx de forma agendada e incremental, com verificação de integridade.
  - **Snowball Edge:** para volumes muito grandes (centenas de TB ou PB) quando a rede não comporta a transferência no prazo.
  - **Amazon File Cache:** um cache de alto desempenho, baseado em Lustre, na AWS. Ele apresenta aos nós da nuvem os dados de um **NFS on-premises** (ou do S3) **sem copiar tudo antes**: os arquivos são carregados sob demanda e as alterações podem ser escritas de volta na origem.
- **Agendamento:** o Slurm on-premises pode ser configurado para enviar jobs a uma partição na AWS, criada com **ParallelCluster** ou **PCS**.
- **Visualização remota:** o **Amazon DCV** (antigo NICE DCV) transmite a tela de uma instância com GPU para o computador do engenheiro. Ele vê e manipula modelos 3D e resultados de simulação **sem baixar** os arquivos, que podem ter centenas de GB. Não há custo de licença do DCV em instâncias EC2.

![Diagrama HPC - Cloud bursting a partir de um cluster on-premises](./img/aws-hpc/aws-hpc-03.png)

[↑ Índice](#índice)

### Otimização de custo

- **Escalar até zero:** ParallelCluster, PCS e Batch removem os nós quando a fila esvazia. Só o head node fica ligado, e no PCS nem ele é seu.
- **Spot** para cargas fracamente acopladas e para jobs com **checkpoint**, com vários tipos de instância (ex: `c6i`, `c7i`, `c7a`, `c7g`) para ter mais pools disponíveis.
- **Graviton** (`hpc7g`, `c7g`, `c8g`): melhor preço por núcleo, se o software compilar para ARM64.
- **FSx for Lustre scratch** apagado no fim do projeto, com os dados permanentes no **S3**. Aplicar **lifecycle** do S3 para mover resultados antigos para classes mais baratas.
- **Savings Plans** para a base de uso constante e **Capacity Blocks** para picos planejados de GPU.

[↑ Índice](#índice)

### Dicas para a prova

- "Simulação com **MPI**, comunicação **intensa entre nós**, **latência** mínima" → **cluster placement group** + **EFA** + instâncias **Hpc**/C, em **uma única AZ e subnet**.
- "Instância de 100 Gbps, mas uma cópia entre duas instâncias **não passa de ~5 Gbps**" → limite **por fluxo** do ENA: usar **várias conexões em paralelo**, **cluster placement group** (~10 Gbps por fluxo) ou **ENA Express** (mesma AZ).
- "Reduzir latência e aumentar a banda de um fluxo TCP **sem alterar a aplicação**" → **ENA Express**. "MPI em escala" → **EFA**.
- "EFA configurado, mas os nós não se comunicam" → **security group** sem a regra de **todo o tráfego** para ele mesmo, ou nós em **subnets diferentes**.
- "Milhares de tarefas **independentes** com o **menor custo**" → **AWS Batch** (array jobs) com **Spot**, vários tipos de instância e AZs.
- "Tarefas independentes **curtas** (< 15 min) em grande volume" → **Step Functions Distributed Map** + **Lambda**.
- "Sistema de arquivos **paralelo** de alto desempenho para HPC, com os dados no **S3**" → **FSx for Lustre** com **data repository association**.
- "Armazenamento temporário mais barato para um job curto" → FSx for Lustre **scratch**. "Jobs longos, dados que precisam sobreviver a falhas" → FSx for Lustre **persistent**.
- "Equipe que usa **Slurm** on-premises quer levar o cluster para a AWS" → **ParallelCluster** (controle total) ou **PCS** (gerenciado, **menor esforço operacional**).
- "Pipeline de jobs em containers, com dependências e sem gerenciar agendador" → **AWS Batch** + **Step Functions**.
- "Usar a nuvem só quando o cluster **on-premises** fica sobrecarregado" → **cloud bursting** com **Direct Connect** + ParallelCluster/PCS, e **DataSync** ou **File Cache** para os dados.
- "Acessar os dados on-premises a partir da nuvem **sem copiar tudo antes**" → **Amazon File Cache**.
- "Engenheiros precisam visualizar resultados 3D **sem baixar** os arquivos" → **Amazon DCV** em instâncias com GPU.
- "Job longo e fortemente acoplado precisa sobreviver a falhas" → **checkpoints** periódicos no FSx ou no S3.
- "Garantir que **todos** os nós do cluster consigam subir juntos" → **On-Demand Capacity Reservation** (ou **Capacity Blocks** para GPU).

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

### High Performance Computing

| Pergunta do cenário | Solução |
|---|---|
| Simulação MPI fortemente acoplada, latência mínima | [Cluster placement group + EFA](#rede-ena-e-elastic-fabric-adapter-efa) + Hpc/C em uma AZ |
| Milhares de tarefas independentes, menor custo | [AWS Batch](#orquestração-parallelcluster-pcs-e-batch) (array jobs) + Spot |
| Cluster Slurm com controle total | [AWS ParallelCluster](#orquestração-parallelcluster-pcs-e-batch) |
| Slurm com menor esforço operacional | [AWS PCS](#orquestração-parallelcluster-pcs-e-batch) |
| Sistema de arquivos paralelo ligado ao S3 | [FSx for Lustre](#armazenamento-para-hpc) |
| Dados on-premises acessados da nuvem sem cópia prévia | [Amazon File Cache](#hpc-híbrido-e-migração) |
| Usar a nuvem quando o cluster local fica sobrecarregado | [Cloud bursting](#hpc-híbrido-e-migração) |
| Visualização remota de resultados 3D | [Amazon DCV](#hpc-híbrido-e-migração) |

[↑ Índice](#índice)
