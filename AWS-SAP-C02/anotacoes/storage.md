---
titulo: Storage
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - Amazon EBS
  - Amazon Data Lifecycle Manager
  - AWS Backup
  - Amazon EFS
  - Amazon FSx
  - AWS DataSync
  - AWS Data Exchange
  - Amazon S3
criado-em: 2026-11-16
atualizado-em: 2026-11-19
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, storage]
---

# Storage

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## Índice

- **Block storage:** [Amazon EBS](#amazon-elastic-block-store-ebs)
- **File storage:** [Amazon EFS](#amazon-elastic-file-system-efs) · [Amazon FSx](#amazon-fsx)
- **Object storage:** [Amazon S3](#amazon-simple-storage-service-s3)
- **Transferência e compartilhamento de dados:** [AWS DataSync](#aws-datasync) · [AWS Data Exchange](#aws-data-exchange)
- **Revisão:** [Qual serviço usar](#revisão-qual-serviço-usar)

## Amazon Elastic Block Store (EBS)

### Contexto: block, file e object storage

Na AWS, há três formas de guardar dados, e cada uma resolve um problema diferente:

- **Block storage (armazenamento em bloco):** o disco é dividido em **blocos** de tamanho fixo, e o **sistema operacional** da máquina decide o que gravar em cada bloco, formatando-o com um sistema de arquivos (ext4, XFS, NTFS). É o equivalente a um **disco rígido ou SSD** ligado a um computador. Só **uma máquina** usa o disco de cada vez, salvo exceções. Na AWS: **EBS** e o instance store.
- **File storage (armazenamento de arquivos):** um **sistema de arquivos compartilhado**, com pastas e arquivos, acessado pela rede por **várias máquinas ao mesmo tempo** (protocolos NFS ou SMB). É o equivalente a uma "pasta de rede" corporativa. Na AWS: **EFS** e a família **FSx**.
- **Object storage (armazenamento de objetos):** cada dado é um **objeto** inteiro (um arquivo + metadados), guardado e lido por **APIs HTTP** (`PUT`, `GET`), sem hierarquia real de pastas nem edição parcial. É praticamente ilimitado e barato. Na AWS: **S3**.

| | **Block (EBS)** | **File (EFS / FSx)** | **Object (S3)** |
|---|---|---|---|
| Como é acessado | Como um **disco local** da instância | Pela rede, como uma **pasta compartilhada** (NFS/SMB) | Por **API HTTP** |
| Quem acessa | **Uma instância** (Multi-Attach é exceção) | **Muitas** instâncias, containers e servidores | Qualquer aplicação, de qualquer lugar |
| Escopo | **Uma AZ** | Região (EFS) ou AZs configuradas | Região |
| Uso típico | Sistema operacional, bancos de dados, aplicações que precisam de disco de baixa latência | Conteúdo compartilhado, home dirs, aplicações legadas que usam arquivos | Data lakes, backups, mídia, sites estáticos, arquivamento |

![Diagrama EBS - Block, file e object storage](./img/amazon-ebs/amazon-ebs-01.png)

[↑ Índice](#índice)

### O que é o EBS

O **Amazon Elastic Block Store** oferece **volumes de bloco** para instâncias EC2: discos virtuais que a instância enxerga como se fossem locais, mas que ficam **na rede** da AZ, separados do servidor físico. Os pontos que definem o serviço são:

- **Persistente:** o volume **existe independentemente da instância**. Ele sobrevive a stop/start e pode ser **desanexado** de uma instância e **anexado** a outra, como um HD externo. Ver a diferença para o instance store em [Armazenamento](compute-and-load-balancing.md#armazenamento).
- **Preso a uma AZ:** um volume só pode ser anexado a instâncias **da mesma AZ**. Para levá-lo a outra AZ ou região, o caminho é o **snapshot**.
- **Replicado dentro da AZ:** cada volume é replicado automaticamente em vários servidores **da mesma AZ**, o que protege contra falhas de hardware, mas **não** contra a falha da AZ inteira.
- **Durabilidade:** de 99,8% a 99,9% ao ano na maioria dos tipos, e **99,999%** no **io2**.
- **Cobrança pelo provisionado:** paga-se o **tamanho alocado** (GB/mês), usado ou não, mais IOPS e throughput extras nos tipos que permitem provisioná-los. Volumes **não anexados** continuam cobrando.
- **Desempenho depende também da instância:** cada tipo de instância tem um **limite de banda para EBS** (*EBS-optimized*). Um volume rápido em uma instância pequena fica limitado pela instância.

**Duas medidas de desempenho** aparecem em todas as questões:

- **IOPS:** quantas **operações** de leitura/escrita por segundo. Importa para acessos **pequenos e aleatórios**, como bancos de dados transacionais.
- **Throughput:** quantos **MB/s** são transferidos. Importa para leituras e escritas **grandes e sequenciais**, como big data, logs e processamento de vídeo.

[↑ Índice](#índice)

### Tipos de volume

| Tipo | Mídia | Desempenho | Características | Uso típico |
|---|---|---|---|---|
| **gp3** (General Purpose SSD) | SSD | Base de **3.000 IOPS** e **125 MB/s** em qualquer tamanho; IOPS e throughput **provisionados à parte**, sem aumentar o tamanho | **Padrão recomendado**; mais barato que o gp2 | A maioria das cargas: boot, aplicações, bancos de pequeno e médio porte |
| **gp2** (General Purpose SSD) | SSD | IOPS **proporcionais ao tamanho** (3 IOPS/GB), com *burst* por créditos em volumes pequenos | Geração anterior: para ganhar IOPS, era preciso aumentar o disco | Legado. **Migrar para gp3** reduz custo |
| **io2 Block Express** (Provisioned IOPS SSD) | SSD | Até **256.000 IOPS** e 4.000 MB/s por volume, latência abaixo de 1 ms | Durabilidade **99,999%**; **Multi-Attach** | Bancos de dados **críticos** e de alto desempenho (Oracle, SQL Server, SAP HANA) |
| **io1** (Provisioned IOPS SSD) | SSD | Até 64.000 IOPS | Geração anterior do io2; Multi-Attach | Legado |
| **st1** (Throughput Optimized HDD) | HDD | Até **500 MB/s**, baseado em throughput | **Não pode ser volume de boot** | Big data, data warehouses, processamento de logs: leitura **sequencial** grande e frequente |
| **sc1** (Cold HDD) | HDD | Até 250 MB/s | **O mais barato**; não pode ser boot | Dados **pouco acessados** que precisam ficar em disco |

**Como escolher:**
- precisa de **muitas IOPS** com **latência mínima** e alta durabilidade → **io2**;
- uso geral → **gp3**;
- grandes volumes **sequenciais** com muito throughput e baixo custo → **st1**;
- dados frios em disco → **sc1**.

![Diagrama EBS - Tipos de volume](./img/amazon-ebs/amazon-ebs-02.png)

> **Obs:** Os limites máximos dos tipos (principalmente do gp3 e do io2) têm sido **ampliados** nos últimos anos. Os valores acima servem para comparar os tipos, mas vale conferir os números atuais na documentação.

**Ir além de um volume:** se um único volume não basta, é possível combinar vários em **RAID 0** (*striping*) no sistema operacional, somando IOPS e throughput. Isso respeita o limite de banda EBS da instância. **RAID 1** (espelhamento) não é recomendado, porque o EBS já é replicado e o espelho dobra o tráfego de rede.

[↑ Índice](#índice)

### Elastic Volumes e Multi-Attach

- **Elastic Volumes:** é possível **aumentar o tamanho**, **mudar o tipo** (ex: gp2 → gp3) e **ajustar IOPS/throughput** de um volume **em uso**, sem desanexá-lo nem parar a instância.
  - Depois de aumentar o tamanho, é preciso **expandir o sistema de arquivos** no sistema operacional.
  - **Não é possível diminuir** o tamanho. Para isso, é preciso criar um volume menor e copiar os dados.
  - Há um intervalo mínimo entre duas modificações no mesmo volume, de algumas horas.
- **Multi-Attach** (io1/io2): o **mesmo volume** é anexado a **várias instâncias Nitro** (até 16) **na mesma AZ**, todas com leitura e escrita. A aplicação precisa de um **sistema de arquivos com suporte a cluster** (*cluster-aware*), que coordene as escritas. Sistemas comuns como ext4 e XFS **corrompem** os dados se duas máquinas escreverem ao mesmo tempo. É usado em clusters de alta disponibilidade de aplicações e bancos que gerenciam o acesso concorrente. Para arquivos compartilhados em geral, o serviço certo é o **EFS** ou o **FSx**.

[↑ Índice](#índice)

### Snapshots

Um **snapshot** é uma **cópia de um volume em um momento** (*point-in-time*), guardada no **S3** de forma gerenciada (você não vê o bucket). É a base de backup, recuperação e migração no EBS:

- **Incrementais:** o primeiro snapshot copia todos os blocos usados. Os seguintes copiam **só os blocos que mudaram**. Mesmo assim, cada snapshot pode ser restaurado sozinho, porque os blocos compartilhados são mantidos enquanto algum snapshot depender deles.
- **Regionais:** um snapshot pode ser restaurado em um volume em **qualquer AZ da região**. É assim que se "move" um volume entre AZs.
- **Cópia entre regiões e contas:** `copy-snapshot` para **DR** em outra região ou para **outra conta** (ex: uma conta de backup isolada).
- **Compartilhamento:** com contas específicas, ou público. Snapshots **criptografados** só podem ser compartilhados se usarem uma **chave KMS gerenciada pelo cliente** (*customer managed key*), com acesso concedido à conta de destino.
- **Consistência:** o snapshot captura o que **já foi gravado** no volume. Para bancos de dados, use snapshots **consistentes com a aplicação**: pause as escritas, use o *flush* do banco ou os **snapshots multi-volume** (*crash-consistent* entre vários volumes da mesma instância), ou deixe o **AWS Backup** coordenar com o VSS no Windows.

**Restauração e desempenho:**

- Um volume criado a partir de um snapshot fica disponível **na hora**, mas os blocos são carregados do S3 **sob demanda** (*lazy loading*). O **primeiro acesso** a cada bloco é **lento**.
- Para evitar isso:
  - **Fast Snapshot Restore (FSR):** habilitado por snapshot e por AZ, faz os volumes criados já virem com **desempenho total** desde o primeiro acesso. É **cobrado por hora**. É usado em DR com RTO baixo e em escalas rápidas a partir de AMIs.
  - **Inicialização manual:** ler todos os blocos do volume (ex: com `fio` ou `dd`) antes de colocá-lo em produção.

**Gestão de custo e retenção:**

- **Snapshot Archive:** move snapshots raramente acessados para um nível de arquivamento **até 75% mais barato**, com **retenção mínima de 90 dias** e **restauração de 24 a 72 horas**. É usado para retenção de longo prazo por compliance.
- **Recycle Bin:** snapshots (e AMIs) apagados vão para uma **lixeira** por um período definido em **regras de retenção**, e podem ser recuperados. Protege contra exclusão acidental ou maliciosa.
- **Data Lifecycle Manager (DLM):** **políticas** que criam, retêm e apagam snapshots automaticamente por tags, com cópia entre regiões e contas. É a automação **nativa e gratuita** do EBS.
- **AWS Backup:** backup **centralizado** de vários serviços (EBS, EC2, RDS, EFS, DynamoDB...), com planos, cópia entre regiões e **entre contas** via Organizations, e **Vault Lock**, que torna os backups **imutáveis** (WORM). É a resposta quando o cenário fala em **governança e compliance multi-conta**.

![Diagrama EBS - Snapshots: incrementais, regionais e ciclo de vida](./img/amazon-ebs/amazon-ebs-03.png)

[↑ Índice](#índice)

### Criptografia

- **Criptografia com KMS** (AES-256): protege os **dados em repouso** no volume, os **snapshots** e os volumes criados a partir deles, e os dados **em trânsito** entre a instância e o volume. O impacto no desempenho é mínimo.
- **Encryption by default:** uma configuração **por região e por conta** que faz **todo volume novo** ser criptografado automaticamente. É a resposta para "garantir que nenhum volume seja criado sem criptografia". Ela pode ser reforçada com SCPs e AWS Config.
- **Não é possível criptografar um volume existente diretamente.** O caminho é:
  1. criar um **snapshot** do volume não criptografado;
  2. **copiar** o snapshot com a opção de criptografia, escolhendo a chave KMS;
  3. criar um **novo volume** a partir da cópia criptografada;
  4. trocar o volume da instância.

  O mesmo caminho serve para **trocar a chave KMS** de um volume.
- **Compartilhar entre contas:** use uma **customer managed key**, conceda acesso à conta de destino na **key policy**, e a conta de destino normalmente **copia** o snapshot e o recriptografa com **a própria chave**. Chaves **gerenciadas pela AWS** (`aws/ebs`) **não** podem ser compartilhadas.

![Diagrama EBS - Criptografar um volume existente e compartilhar entre contas](./img/amazon-ebs/amazon-ebs-04.png)

[↑ Índice](#índice)

### Alta disponibilidade e recuperação

Como o EBS fica **em uma AZ**, a disponibilidade além da AZ é responsabilidade da arquitetura:

- **Falha da AZ:** o volume fica indisponível. Para continuar, é preciso **restaurar snapshots em outra AZ** (RTO de minutos, RPO = intervalo entre snapshots), ou ter **replicação no nível da aplicação** (ex: réplica do banco em outra AZ).
- **DR entre regiões:** **cópia de snapshots** para a região de DR (com DLM ou AWS Backup). Para RPO e RTO de segundos ou minutos em servidores inteiros, o **AWS Elastic Disaster Recovery** replica os volumes continuamente.
- **Bancos de dados gerenciados** (RDS, Aurora) cuidam da replicação entre AZs e regiões. Por isso, em muitas questões, "banco em EC2 com EBS" perde para um banco gerenciado quando o requisito é alta disponibilidade com o menor esforço operacional.
- **Dados compartilhados entre AZs:** EBS não serve, porque é preso a uma AZ. A resposta é **EFS** ou **FSx**.

[↑ Índice](#índice)

### Monitoramento e custo

- **CloudWatch:** `VolumeReadOps`/`VolumeWriteOps` (IOPS), `VolumeReadBytes`/`VolumeWriteBytes` (throughput), **`VolumeQueueLength`**, em que fila alta indica que o volume está no limite, e **`BurstBalance`** (gp2, st1, sc1), cujo esgotamento derruba o desempenho.
- **Status checks do volume:** indicam problemas de I/O.
- **Otimização de custo:**
  - migrar **gp2 → gp3**: mais barato e com IOPS independentes do tamanho;
  - apagar **volumes não anexados** e **snapshots antigos** (DLM, Recycle Bin com retenção curta);
  - usar o **Snapshot Archive** para retenção longa;
  - fazer o **rightsizing** de tipo, tamanho e IOPS com o **Compute Optimizer** e o **Trusted Advisor**.

[↑ Índice](#índice)

### Dicas para a prova

- "Banco de dados crítico com **IOPS muito altas** e latência abaixo de 1 ms" → **io2 Block Express**.
- "Uso geral com **menor custo**", ou "precisa de mais IOPS **sem aumentar** o disco" → **gp3**. "Reduzir custo dos volumes gp2" → migrar para **gp3** (Elastic Volumes, sem parada).
- "Processamento sequencial de grandes volumes (big data, logs) com **baixo custo**" → **st1**. "Dados frios em disco ao menor custo" → **sc1**. Nenhum dos dois serve como **boot**.
- "Mesmo volume em várias instâncias na mesma AZ, com aplicação que gerencia o acesso concorrente" → **Multi-Attach** (io1/io2). "Arquivos compartilhados entre instâncias/AZs" → **EFS/FSx**, não EBS.
- "Mover um volume para outra AZ ou região" → **snapshot** → restaurar na outra AZ / **copiar** para a outra região.
- "Volumes restaurados de snapshot estão **lentos** no início" → **Fast Snapshot Restore** (ou inicializar os blocos).
- "Automatizar snapshots com retenção e cópia entre regiões" → **Data Lifecycle Manager**. "Backup **centralizado** de vários serviços e contas, com backups **imutáveis**" → **AWS Backup** (+ Vault Lock).
- "Retenção de snapshots por anos ao menor custo" → **Snapshot Archive**. "Proteger contra exclusão acidental de snapshots" → **Recycle Bin**.
- "Garantir que todo volume novo seja criptografado" → **EBS encryption by default** (por região).
- "Criptografar um volume existente" ou "trocar a chave KMS" → snapshot → **copy-snapshot com criptografia** → novo volume.
- "Compartilhar snapshot criptografado com outra conta" → **customer managed KMS key** + key policy liberando a conta (não a chave `aws/ebs`).
- "Aumentar o volume sem parar a aplicação" → **Elastic Volumes** + expandir o sistema de arquivos. "Diminuir o volume" → criar um volume menor e copiar os dados.
- "Um volume não entrega o desempenho necessário" → **RAID 0** de vários volumes (respeitando a banda EBS da instância), ou uma instância com **mais banda EBS**.
- "Volume io2 rápido em instância pequena não atinge as IOPS provisionadas" → limite de **banda EBS da instância**.
- "Disco de altíssimo IOPS para dados temporários, sem custo extra" → **instance store**, não EBS.

[↑ Índice](#índice)

## Amazon Elastic File System (EFS)

### Contexto: por que um file system compartilhado

O EBS resolve o "disco de **uma** máquina". Muitas aplicações, porém, precisam que **várias máquinas leiam e gravem os mesmos arquivos ao mesmo tempo**:

- um site com várias instâncias atrás de um load balancer, em que os uploads dos usuários precisam aparecer para todas;
- containers que compartilham configuração, modelos de machine learning ou conteúdo;
- diretórios home de usuários, pipelines de mídia, ferramentas de CI e aplicações legadas que esperam gravar em uma "pasta".

Antes da nuvem, isso era um **NAS** (*Network Attached Storage*): um servidor de arquivos na rede, que as máquinas "montam" como uma pasta pelo protocolo **NFS** (Linux) ou **SMB** (Windows). Manter um NAS significa dimensionar discos, cuidar da alta disponibilidade e expandir a capacidade.

O **EFS** é esse NAS como serviço gerenciado: um sistema de arquivos **NFS** que você monta em instâncias, containers e funções, que **cresce e diminui sozinho** e que, por padrão, guarda os dados em **várias AZs**. Ver a comparação entre block, file e object storage em [Contexto: block, file e object storage](#contexto-block-file-e-object-storage).

[↑ Índice](#índice)

### O que é o EFS

- **NFS v4.0/4.1, só Linux:** o EFS segue a semântica de arquivos POSIX (permissões, locks, hierarquia de diretórios). **Não** funciona com Windows nem SMB. Para isso, o serviço certo é o **FSx for Windows File Server**.
- **Elástico:** não há tamanho a provisionar. Você grava e o file system cresce até petabytes; apaga e ele diminui. Paga-se o **armazenamento usado** (GB/mês), diferente do EBS, que cobra o provisionado.
- **Milhares de clientes simultâneos:** instâncias EC2, tasks ECS/Fargate, pods EKS, funções Lambda e servidores on-premises podem montar o mesmo file system ao mesmo tempo.
- **Durabilidade de 11 noves** e consistência *close-to-open*: o que um cliente grava e fecha fica visível para os outros na próxima abertura do arquivo.
- **Custo:** o armazenamento do EFS Standard custa **bem mais** por GB que o EBS gp3 e muito mais que o S3. A economia vem das **classes IA e Archive** e de pagar só pelo que está gravado.

[↑ Índice](#índice)

### Arquitetura: mount targets e acesso

O file system é um recurso **regional**, fora da sua VPC. Para que os clientes cheguem a ele, você cria **mount targets**:

- um **mount target** é uma **interface de rede (ENI)** com um IP privado em uma subnet da sua VPC, protegida por **security group**;
- cria-se **um mount target por AZ** (no máximo um por AZ), normalmente em todas as AZs onde há clientes;
- o cliente monta pelo **nome DNS** do file system (`fs-123.efs.us-east-1.amazonaws.com`), que resolve para o mount target **da mesma AZ**. Isso evita latência e custo de tráfego entre AZs;
- o security group do mount target precisa liberar a **porta TCP 2049 (NFS)** a partir dos clientes.

**Quem pode montar:**

- **EC2:** com o **mount helper** (`amazon-efs-utils`), que facilita TLS e IAM: `sudo mount -t efs -o tls fs-123:/ /mnt/efs`. Para montar no boot, use o `/etc/fstab`.
- **ECS e Fargate:** o volume EFS é declarado na **task definition**. É a forma padrão de dar **armazenamento persistente e compartilhado** a tasks Fargate, que não têm disco persistente próprio.
- **EKS:** pelo **EFS CSI driver**, como volume persistente (`PersistentVolume`) com acesso **ReadWriteMany**, ou seja, vários pods em vários nós gravando ao mesmo tempo.
- **Lambda:** a função precisa estar **conectada à VPC** e usar um **access point**. É usado para arquivos grandes (modelos de ML, bibliotecas) ou para compartilhar estado entre execuções.
- **On-premises:** via **Direct Connect** ou **Site-to-Site VPN**, montando pelo **IP** do mount target, porque o nome DNS do EFS só resolve dentro da VPC, a menos que se use o Route 53 Resolver.
- **Outras VPCs ou contas:** via **VPC peering** ou **Transit Gateway**, também pelo IP do mount target.

![Diagrama EFS - Arquitetura com mount targets por AZ](./img/amazon-efs/amazon-efs-01.png)

[↑ Índice](#índice)

### Regional vs. One Zone

| | **Regional** (recomendado) | **One Zone** |
|---|---|---|
| Onde ficam os dados | Em **3 ou mais AZs** da região | Em **uma AZ** |
| Mount targets | Em várias AZs | Só na AZ do file system |
| Disponibilidade | 99,99% | 99,9%; **perde dados** se a AZ for destruída |
| Custo | Maior | Aproximadamente **metade** do preço |
| Uso típico | Produção, dados que precisam de alta disponibilidade | Dev/teste, dados que podem ser recriados, cópias secundárias |

Clientes de outras AZs ainda podem montar um file system One Zone, mas pagam **transferência entre AZs** e ficam sem acesso se a AZ cair. Em file systems One Zone, os **backups automáticos do AWS Backup** já vêm ativados.

[↑ Índice](#índice)

### Classes de armazenamento e lifecycle

O EFS tem três classes. Os dados novos sempre são gravados no **Standard**:

| Classe | Latência | Para dados acessados | Custo |
|---|---|---|---|
| **EFS Standard** | Abaixo de 1 ms (SSD) | Com frequência | Armazenamento mais caro; sem taxa de acesso |
| **EFS Infrequent Access (IA)** | Dezenas de ms | Poucas vezes **por trimestre** | Armazenamento bem mais barato + **taxa por GB lido** |
| **EFS Archive** | Dezenas de ms | Poucas vezes **por ano** | O mais barato + taxa de acesso; **mínimo de 90 dias** de armazenamento; exige o modo de throughput **Elastic** |

O **lifecycle management** move cada **arquivo** entre as classes de acordo com o **último acesso** a ele, sem mudar nada para a aplicação. O caminho do arquivo continua o mesmo, só muda onde ele está guardado:

- **TransitionToIA:** arquivos sem acesso há N dias (ex: 30) vão para o IA;
- **TransitionToArchive:** arquivos sem acesso há mais tempo (ex: 90 dias) vão para o Archive;
- **TransitionToPrimaryStorageClass = AFTER_1_ACCESS:** no **primeiro acesso**, o arquivo volta para o Standard. Sem essa opção, ele fica na classe fria e cada leitura paga a taxa de acesso.

Esse comportamento lembra o **S3 Intelligent-Tiering**: é a resposta para "reduzir o custo de um file system em que a maior parte dos arquivos é raramente acessada, sem mudar a aplicação". Arquivos no IA e no Archive são cobrados por no mínimo **128 KiB**, e o **AWS Backup** não paga taxa de acesso ao ler dados nessas classes.

![Diagrama EFS - Classes de armazenamento e lifecycle management](./img/amazon-efs/amazon-efs-02.png)

[↑ Índice](#índice)

### Modos de desempenho e throughput

**Performance mode** (escolhido na criação):

- **General Purpose** (padrão e **recomendado**): a menor latência por operação. É o único modo dos file systems One Zone. A métrica `PercentIOLimit` mostra se o file system está perto do limite de operações.
- **Max I/O:** modo da **geração anterior**, para cargas muito paralelas que toleram latência maior. Não funciona com One Zone nem com o throughput Elastic. Em questões antigas, aparecia como resposta para "milhares de instâncias em paralelo" (big data, mídia); hoje a AWS recomenda General Purpose com Elastic.

**Throughput mode** (pode ser trocado depois):

| Modo | Como funciona | Cobrança | Quando usar |
|---|---|---|---|
| **Elastic** (padrão e **recomendado**) | Escala o throughput **automaticamente** conforme a demanda, até dezenas de GiB/s de leitura | Por **GB lido e gravado** | Cargas **imprevisíveis ou com picos**; uso médio baixo comparado ao pico |
| **Provisioned** | Você define o throughput, **independente do tamanho** armazenado | Pelo throughput provisionado acima do incluído | Carga **conhecida e constante**, com throughput alto em relação ao volume de dados |
| **Bursting** | Throughput **proporcional ao tamanho** no Standard (50 MiB/s por TiB), com **créditos** para picos de até 100 MiB/s por TiB | Incluído no armazenamento | Cargas em que o throughput cresce junto com os dados |

O problema clássico do Bursting: um file system **pequeno** (poucos GB) tem throughput base baixo, os créditos acabam (`BurstCreditBalance` chega a zero) e a aplicação fica lenta. A solução é mudar para **Elastic** ou **Provisioned**, não gravar arquivos falsos para "aumentar o tamanho", como se fazia antigamente.

> **Obs:** depois de mudar para Provisioned ou alterar o valor provisionado, é preciso esperar **24 horas** para voltar para outro modo ou diminuir o valor.

[↑ Índice](#índice)

### Segurança e controle de acesso

Uma requisição ao EFS passa por várias camadas, e todas precisam permitir:

1. **Rede:** o **security group do mount target** libera a porta 2049 a partir dos clientes. O padrão é referenciar o SG dos clientes, e não faixas de IP.
2. **IAM:**
   - a **file system policy** é uma política **baseada em recurso** (como uma bucket policy do S3). Usos comuns: **exigir TLS** (`aws:SecureTransport`), negar acesso **root**, permitir só leitura ou liberar acesso a outra conta;
   - a **identity policy** do cliente (role da instância, task role, role da Lambda) concede ações como `elasticfilesystem:ClientMount` e `ClientWrite`;
   - para que o IAM seja avaliado, o cliente monta com `-o iam`. Sem uma file system policy, qualquer cliente que alcance o mount target pela rede tem acesso completo.
3. **Access points:** pontos de entrada **por aplicação**, que:
   - forçam uma **identidade POSIX** (UID/GID), ignorando o usuário que o cliente informar;
   - definem um **diretório raiz**, de modo que a aplicação só enxerga aquele diretório como `/`;
   - podem criar o diretório com dono e permissões definidos.

   São a forma de **várias aplicações ou tenants compartilharem um file system com isolamento**, e são obrigatórios para a Lambda.
4. **Permissões POSIX:** usuário, grupo e outros nos arquivos e diretórios, como em qualquer Linux.

**Criptografia:**

- **Em repouso:** com KMS, definida **na criação** do file system. **Não é possível ativar depois.** Para criptografar um file system existente, crie um novo file system criptografado e copie os dados com o **DataSync** (ou restaure um backup). Uma **SCP** ou uma regra do AWS Config podem exigir criptografia na criação (`elasticfilesystem:Encrypted`).
- **Em trânsito:** **TLS**, ativado na montagem com o mount helper (`-o tls`). A file system policy pode **negar** conexões sem TLS.

![Diagrama EFS - Camadas de controle de acesso e access points](./img/amazon-efs/amazon-efs-03.png)

[↑ Índice](#índice)

### Replicação e backup do EFS

- **EFS Replication:** replica o file system para **outra região** (ou a mesma), inclusive para **outra conta**.
  - RPO de **15 minutos** para a maioria dos file systems; RTO de minutos;
  - a réplica é **somente leitura** enquanto a replicação está ativa;
  - **failover:** apagar a configuração de replicação, o que torna a réplica gravável, e apontar a aplicação para ela;
  - **failback:** criar a replicação **no sentido inverso** para levar de volta o que mudou durante o failover, depois recriar a original;
  - é a resposta para **DR do EFS entre regiões** com o menor esforço.
- **AWS Backup:** backups **point-in-time**, incrementais, ativados **por padrão** em file systems novos criados pelo console. Permite:
  - **cópia entre regiões e contas** e **Vault Lock** (imutabilidade);
  - restaurar o file system inteiro ou **arquivos específicos**.

Replicação e backup se complementam. A replicação também copia **exclusões e corrupções** em minutos. Para voltar a uma versão anterior dos dados, o caminho é o **backup**.

![Diagrama EFS - Replicação entre regiões e AWS Backup](./img/amazon-efs/amazon-efs-04.png)

[↑ Índice](#índice)

### Migração e integração

- **AWS DataSync:** copia dados de um NAS on-premises (NFS/SMB), de outro EFS ou do S3 **para o EFS**, com criptografia em trânsito, verificação de integridade, agendamento e controle de banda. É a resposta para **migrar** um file server para o EFS ou **copiar dados entre file systems** (ex: para mudar a criptografia ou a região sem usar a replicação).
- **AWS Transfer Family:** expõe o EFS (ou o S3) por **SFTP, FTPS e FTP**, para parceiros que enviam arquivos por esses protocolos.
- **Aplicações legadas que gravam em diretório:** o EFS permite levá-las para várias instâncias ou containers sem mudar o código.

[↑ Índice](#índice)

### Monitoramento e custo do EFS

- **CloudWatch:**
  - `BurstCreditBalance` (modo Bursting): créditos perto de zero indicam que o throughput vai cair;
  - `PercentIOLimit` (General Purpose): perto de 100% indica o limite de operações;
  - `MeteredIOBytes` e `PermittedThroughput`: uso e limite de throughput;
  - `StorageBytes` (por classe), `ClientConnections` e `TimeSinceLastSync` (atraso da replicação).
- **Otimização de custo:**
  - ativar o **lifecycle management** com IA e Archive;
  - usar **One Zone** para dados que podem ser recriados;
  - escolher **Elastic** para cargas com picos e **Provisioned** para cargas altas e constantes;
  - se os dados são acessados como **objetos inteiros** por API, e não como arquivos montados, o **S3** é muito mais barato.

[↑ Índice](#índice)

### Dicas para a prova

- "Várias instâncias EC2 em **várias AZs** precisam **ler e gravar os mesmos arquivos**" (Linux) → **EFS**. Se for **Windows/SMB/Active Directory** → **FSx for Windows File Server**.
- "Armazenamento **persistente e compartilhado** para tasks **Fargate**" ou "volume **ReadWriteMany** no EKS" → **EFS**.
- "Função Lambda precisa de arquivos grandes ou de um sistema de arquivos compartilhado" → **EFS** com a Lambda **na VPC** e um **access point**.
- "Reduzir custo de um file system em que a maioria dos arquivos é pouco acessada, **sem mudar a aplicação**" → **lifecycle management** para IA/Archive, com volta ao Standard no primeiro acesso.
- "Dados que podem ser recriados, ao menor custo" → **EFS One Zone** (+ lifecycle).
- "File system pequeno com **throughput insuficiente**" ou "créditos de burst esgotados" → mudar para **Elastic** (picos) ou **Provisioned** (carga constante).
- "Montar o EFS a partir do data center" → **Direct Connect/VPN** + **IP do mount target**. "Migrar o NAS on-premises para o EFS" → **DataSync**.
- "Isolar várias aplicações ou tenants no mesmo file system" → **access points** (UID/GID e diretório raiz forçados).
- "Exigir criptografia em trânsito" → **file system policy** negando conexões sem `aws:SecureTransport` + montagem com `-o tls`.
- "Criptografar em repouso um file system existente" → **criar um novo criptografado e copiar com DataSync**. Não dá para ativar depois.
- "DR do EFS em outra região com RPO de minutos" → **EFS Replication**. "Recuperar arquivos apagados ou corrompidos" / "backups imutáveis entre contas" → **AWS Backup** (+ Vault Lock).
- "Clientes não conseguem montar (timeout)" → security group do mount target sem a **porta 2049**, ou falta de mount target na AZ/subnet.
- "Compartilhar arquivos entre servidores, mas o acesso é por **API** e o volume é enorme" → **S3**, não EFS.

[↑ Índice](#índice)

## Amazon FSx

### Contexto: file systems de mercado na nuvem

O EFS é um file system **criado pela AWS**: simples, elástico, só NFS e só Linux. Muitas empresas, porém, dependem de file systems **específicos**, com recursos que as aplicações e as equipes já usam:

- o **Windows File Server**, com compartilhamentos SMB, permissões do Active Directory e "versões anteriores" no Explorer;
- o **NetApp ONTAP**, storage corporativo com snapshots, replicação SnapMirror e acesso por NFS, SMB e iSCSI ao mesmo tempo;
- o **Lustre**, file system paralelo usado em supercomputadores e clusters de HPC;
- o **ZFS**, comum em servidores Linux e appliances NFS.

Migrar essas cargas para o EFS exigiria mudar aplicações, permissões ou processos. O **Amazon FSx** oferece **esses mesmos file systems como serviço gerenciado**. A AWS cuida de hardware, patches, alta disponibilidade e backups, e você mantém os recursos e o comportamento que conhece. Na prova, o desafio é reconhecer, pelas palavras do cenário, **qual dos quatro** atende o caso, ou quando o **EFS** basta.

[↑ Índice](#índice)

### Visão geral da família FSx

| | **Windows File Server** | **Lustre** | **NetApp ONTAP** | **OpenZFS** |
|---|---|---|---|---|
| Protocolos | **SMB** | Lustre (cliente Linux) | **NFS, SMB, iSCSI, NVMe/TCP** | **NFS** (v3, v4.x) |
| Clientes | Windows (e Linux/macOS via SMB) | Linux | Linux, Windows, macOS | Linux, macOS (e Windows com cliente NFS) |
| Identidade | **Active Directory**, ACLs NTFS | POSIX | AD (SMB) e POSIX (NFS) | POSIX |
| Disponibilidade | Single-AZ ou **Multi-AZ** | **Uma AZ** (Scratch ou Persistent) | Single-AZ ou **Multi-AZ** | Single-AZ ou **Multi-AZ** |
| Destaque | DFS, shadow copies, dedup, quotas | **Centenas de GB/s**, integração com **S3** | **Multiprotocolo**, SnapMirror, tiering, FlexClone | Latência muito baixa, **clones instantâneos** |
| Gatilho na prova | "Windows", "SMB", "Active Directory" | "HPC", "ML", "processar dados do S3" | "NetApp", "Linux **e** Windows", "iSCSI" | "ZFS", "migrar NFS on-premises", "clones" |

![Diagrama FSx - Família FSx](./img/amazon-fsx/amazon-fsx-01.png)

[↑ Índice](#índice)

### FSx for Windows File Server

Um **file server Windows** gerenciado, acessado por **SMB**, com o sistema de arquivos **NTFS**:

- **Active Directory:** o file system entra em um domínio, que pode ser o **AWS Managed Microsoft AD** ou um **AD autogerenciado** (on-premises via Direct Connect/VPN, ou em EC2). As permissões são as **ACLs do Windows**, por usuários e grupos do domínio.
- **Implantação:**
  - **Single-AZ:** um file server em uma AZ, com os dados replicados dentro dela;
  - **Multi-AZ:** um file server **preferido** em uma AZ e um **standby** em outra, com **replicação síncrona**. Na falha, o **failover é automático**: o nome DNS passa a apontar para o standby e os clientes reconectam. É a resposta para **alta disponibilidade** de file shares Windows.
- **Armazenamento e desempenho:** **SSD** (latência baixa, bancos e aplicações) ou **HDD** (home dirs e compartilhamentos gerais, mais barato). Você provisiona a **capacidade** (só pode aumentar) e o **throughput**, que pode ser ajustado depois.
- **Recursos de Windows Server:**
  - **DFS Namespaces:** juntam vários file systems (ou shares on-premises) em **um caminho único** (`\\corp.local\dados`). É assim que se escala além de um file system ou se migra sem mudar o caminho para os usuários;
  - **Shadow Copies:** o usuário restaura versões anteriores dos arquivos pelo Explorer, sem chamar a equipe de TI;
  - **deduplicação de dados**, **quotas por usuário**, **auditoria de acesso** (para CloudWatch Logs ou Kinesis Data Firehose) e **SMB Multichannel**;
  - **continuously available shares** para **SQL Server** em cluster.
- **Usos típicos:** home dirs e pastas de departamento, aplicações .NET e de linha de negócio, SQL Server, SharePoint, conteúdo para **Amazon WorkSpaces**.

![Diagrama FSx - FSx for Windows File Server Multi-AZ](./img/amazon-fsx/amazon-fsx-02.png)

> **Obs:** o **Amazon FSx File Gateway** (do Storage Gateway), que fazia cache local de um FSx for Windows em escritórios remotos, não aceita mais novos clientes. Em questões antigas, ele aparece como resposta para "acesso de baixa latência ao FSx a partir das filiais".

[↑ Índice](#índice)

### FSx for Lustre

**Lustre** é um file system **paralelo**: os arquivos são divididos entre vários servidores de armazenamento, e milhares de clientes leem e gravam pedaços diferentes **ao mesmo tempo**. Por isso, ele entrega **centenas de GB/s**, milhões de IOPS e latência abaixo de 1 ms. Ver o contexto de HPC em [High Performance Computing (HPC)](compute-and-load-balancing.md#high-performance-computing-hpc).

- **Clientes Linux** com o cliente Lustre: clusters de EC2 (inclusive com **EFA** e **GPUDirect Storage** no Persistent 2), **SageMaker**, **AWS Batch**, **ParallelCluster** e EKS.
- **Integração com o S3**, a característica que mais cai na prova: com uma **Data Repository Association (DRA)**, um diretório do Lustre fica ligado a um bucket ou prefixo do S3.
  - **import:** os **metadados** dos objetos aparecem na hora como arquivos; os **dados** são carregados do S3 no **primeiro acesso** (*lazy loading*);
  - **export:** arquivos novos ou alterados voltam para o S3, de forma **automática** ou por **data repository task**.

  O padrão é: o **S3 é a fonte da verdade** (barato e durável), e o Lustre é a **camada de alto desempenho** usada durante o processamento.
- **Tipos de implantação:**
  - **Scratch:** armazenamento **temporário**, **sem replicação**. Se um servidor falha, os arquivos dele ficam indisponíveis. É o mais barato, com burst alto. Serve para **processamento curto** em que os dados estão seguros no S3.
  - **Persistent (1 e 2):** os dados são **replicados na AZ** e os servidores com falha são **substituídos** automaticamente. Serve para cargas **longas** ou que não podem perder dados. O Persistent 2 é a geração atual, com mais throughput por TiB.
- **Classes de armazenamento:**
  - **SSD:** latência sub-ms em todos os dados;
  - **HDD:** mais barato, com cache SSD opcional;
  - **Intelligent-Tiering:** **elástico**, paga-se o que está armazenado; move os dados frios para camadas mais baratas sem taxa de recuperação.
- **Uma AZ:** os clientes devem estar na mesma AZ para melhor desempenho. A durabilidade de longo prazo vem do S3 e dos backups.

![Diagrama FSx - FSx for Lustre com S3](./img/amazon-fsx/amazon-fsx-03.png)

[↑ Índice](#índice)

### FSx for NetApp ONTAP

O sistema operacional de storage da **NetApp** como serviço gerenciado. É o "canivete suíço" da família:

- **Multiprotocolo:** **NFS**, **SMB**, **iSCSI** e **NVMe/TCP** no mesmo file system. Os **mesmos dados** podem ser acessados por **Linux e Windows**, e há volumes de **bloco** (LUNs) para bancos de dados e VMware.
- **Organização:** um file system contém **SVMs** (*storage virtual machines*, servidores de arquivos lógicos, cada um com seus endpoints e seu domínio AD), e cada SVM contém **volumes**.
- **Single-AZ ou Multi-AZ**, com failover automático. Implantações **scale-out**, com vários pares de servidores, atendem cargas maiores.
- **Armazenamento em duas camadas:**
  - **SSD tier:** dados ativos, capacidade provisionada;
  - **capacity pool:** dados frios, **elástico e barato**. O **tiering** entre as camadas é **automático**, por política de volume.

  Junto com **compressão, deduplicação e compactação**, isso reduz bastante o custo por GB efetivo.
- **Recursos NetApp:**
  - **Snapshots:** instantâneos e eficientes em espaço;
  - **FlexClone:** cópias **instantâneas** e graváveis de volumes, sem duplicar os dados, para **dev/test**;
  - **SnapMirror:** replicação para outro FSx ONTAP (**DR entre regiões**) ou de/para um **NetApp on-premises** (**migração** e DR híbrido);
  - **SnapLock:** WORM;
  - **FlexCache:** cache remoto;
  - integração com antivírus.
- **Usos típicos:**
  - migração de **NetApp on-premises** sem mudar os processos;
  - dados compartilhados por **Linux e Windows**;
  - **datastore** para **VMware Cloud on AWS**;
  - bancos de dados via iSCSI;
  - ambientes de dev/test com clones.

![Diagrama FSx - FSx for NetApp ONTAP](./img/amazon-fsx/amazon-fsx-04.png)

[↑ Índice](#índice)

### FSx for OpenZFS

O file system **ZFS** gerenciado, acessado por **NFS** (v3 e v4.x):

- **Latência muito baixa** (abaixo de 1 ms) e alto número de IOPS, com cache em memória e em disco nos servidores.
- **Snapshots** e **clones instantâneos**: criar uma cópia gravável de um volume leva segundos, sem copiar os dados. Serve para ambientes de **dev/test**, CI e análises sobre uma cópia da produção.
- **Compressão** (LZ4, Zstandard), que reduz o espaço e muitas vezes melhora o desempenho.
- Single-AZ ou **Multi-AZ**.
- **Usos típicos:** migração de servidores **ZFS** ou de file servers **NFS on-premises** (Linux) que precisam de desempenho previsível e baixa latência, sem a complexidade do ONTAP.

**OpenZFS ou EFS?** Os dois são NFS para Linux.

| Precisa de... | Use |
|---|---|
| Capacidade **elástica**, sem gerenciar, com clientes em várias AZs | **EFS** |
| **Latência mínima**, **clones**, compatibilidade com ZFS, ou desempenho de IOPS acima do que o EFS entrega com bom custo | **OpenZFS** |

[↑ Índice](#índice)

### Recursos comuns do FSx

- **Rede:** o file system cria **ENIs na sua VPC**, protegidas por **security groups**. O acesso on-premises e de outras VPCs é feito pela rede privada (**Direct Connect**, **VPN**, **VPC peering**, **Transit Gateway**).
- **Criptografia:**
  - **em repouso** com **KMS**, sempre ativa;
  - **em trânsito** de acordo com o protocolo: criptografia SMB 3 no Windows, Kerberos ou IPsec no ONTAP, e criptografia automática em instâncias Nitro compatíveis.
- **Backups:**
  - **automáticos diários**, incrementais, guardados no S3 de forma gerenciada, com retenção configurável;
  - backups **manuais**, que ficam até serem apagados;
  - **AWS Backup** para políticas centralizadas, **cópia entre regiões e contas** e Vault Lock.

  O FSx for Lustre **Scratch** não tem backup, porque os dados devem estar no S3.
- **Capacidade provisionada:** diferente do EFS, a maioria das opções exige definir **capacidade de armazenamento** e **throughput**. A capacidade pode aumentar, mas **não diminuir**. As exceções elásticas são o capacity pool do ONTAP e a classe Intelligent-Tiering do Lustre e do OpenZFS.
- **Monitoramento:** métricas no **CloudWatch** (throughput, IOPS, capacidade livre, uso de CPU dos servidores de arquivos) e API no **CloudTrail**.

[↑ Índice](#índice)

### Migração para o FSx

- **AWS DataSync:** copia de file servers **SMB ou NFS** on-premises (ou de outros serviços de storage) para **qualquer FSx**, com agendamento, verificação e controle de banda. É a resposta padrão para migração online de arquivos.
- **FSx for Windows:** além do DataSync, o **Robocopy** preserva as ACLs. Os **DFS Namespaces** permitem migrar share por share **sem mudar o caminho** que os usuários acessam.
- **FSx for NetApp ONTAP:** **SnapMirror** a partir do NetApp on-premises, que replica os volumes com os snapshots e as permissões.
- **FSx for Lustre:** os dados geralmente **já estão no S3**, e a **DRA** os importa sob demanda.
- **Grandes volumes com pouca banda:** **AWS Snowball** para levar os dados até a AWS, depois DataSync para a sincronização final.

[↑ Índice](#índice)

### Como escolher entre EFS e FSx

Uma sequência de perguntas resolve a maioria dos cenários:

1. Clientes **Windows**, **SMB** e **Active Directory**? → **FSx for Windows File Server**.
2. **Linux e Windows** nos **mesmos dados**, vindo de **NetApp**, ou precisa de **iSCSI**? → **FSx for NetApp ONTAP**.
3. **HPC ou ML**, com **throughput extremo** e dados no **S3**? → **FSx for Lustre**.
4. Linux NFS vindo de **ZFS**, **latência mínima** ou **clones** instantâneos? → **FSx for OpenZFS**.
5. Linux NFS **elástico**, sem gerenciar capacidade, para EC2, containers e Lambda? → **Amazon EFS**.

![Diagrama FSx - Qual file storage usar](./img/amazon-fsx/amazon-fsx-05.png)

[↑ Índice](#índice)

### Dicas para a prova

- "File share **Windows** com integração ao **Active Directory**" → **FSx for Windows File Server**. "Com **alta disponibilidade**" → **Multi-AZ**.
- "Usuários precisam **restaurar versões anteriores** dos próprios arquivos" → **Shadow Copies** no FSx for Windows.
- "Juntar vários file shares em **um caminho único**" ou "migrar sem mudar o caminho" → **DFS Namespaces**.
- "Processar dados do **S3** com **file system de alto desempenho** para HPC/ML" → **FSx for Lustre** com **DRA** (lazy loading + export).
- "Processamento **temporário** e de **menor custo**, dados já salvos no S3" → Lustre **Scratch**. "Carga longa, não pode perder dados" → Lustre **Persistent**.
- "Mesmos dados via **NFS e SMB**" ou "aplicações **Linux e Windows** compartilhando arquivos" → **FSx for NetApp ONTAP**.
- "Migrar **NetApp on-premises** mantendo snapshots e replicação" → **FSx for NetApp ONTAP** + **SnapMirror**.
- "Storage de **bloco iSCSI** gerenciado e compartilhado" ou "datastore para **VMware Cloud on AWS**" → **FSx for NetApp ONTAP**.
- "Reduzir custo de file storage com muitos dados frios, sem mudar a aplicação" → ONTAP com **capacity pool** (tiering), EFS com **lifecycle**, ou Lustre/OpenZFS com **Intelligent-Tiering**.
- "Migrar servidor **ZFS** ou NFS on-premises com baixa latência" ou "**clones** instantâneos para dev/test" → **FSx for OpenZFS** (ou FlexClone no ONTAP).
- "Migrar arquivos de um NAS on-premises para o FSx/EFS" → **DataSync**.
- "Linux NFS, elástico, serverless, várias AZs" → **EFS**, não FSx.
- "Backup do FSx em outra região/conta" → **AWS Backup** (cópia entre regiões e contas). "DR do ONTAP com RPO baixo" → **SnapMirror** para outra região.

[↑ Índice](#índice)

## Amazon Simple Storage Service (S3)

### Contexto: armazenamento de objetos

O EBS e o EFS imitam o que um sistema operacional já conhece: um **disco** e uma **pasta de rede**. O S3 parte de outra ideia. Em vez de montar algo na máquina, a aplicação **envia e busca arquivos inteiros por HTTP**, como faria com qualquer API:

- `PUT` grava um arquivo (o **objeto**) com um nome (a **chave**);
- `GET` lê o objeto pelo nome;
- não há "abrir o arquivo e mudar a linha 30": para alterar, grava-se o objeto inteiro de novo.

Abrir mão do sistema de arquivos traz vantagens enormes:

- **capacidade praticamente ilimitada**, sem provisionar nada;
- **11 noves de durabilidade**, com os dados espalhados por várias AZs;
- acesso de **qualquer lugar** (aplicações, browsers, outros serviços da AWS);
- **custo muito menor** por GB, com classes que chegam a centavos por TB/mês para arquivamento.

Por isso o S3 é a base de **data lakes**, **backups**, **sites estáticos**, **mídia**, **logs** e da integração entre serviços da AWS. Na prova, ele aparece em quase todos os domínios: custo, DR, migração, segurança e analytics. Ver a comparação com block e file storage em [Contexto: block, file e object storage](#contexto-block-file-e-object-storage).

[↑ Índice](#índice)

### Conceitos do S3: buckets, objetos e chaves

- **Bucket:** o contêiner dos objetos. Pertence a **uma região**, mas o nome é **único no mundo inteiro**. O limite padrão é de 10 mil buckets por conta, e pode ser aumentado.
- **Objeto:** os dados (até **5 TB**) + **metadados** (`Content-Type`, metadados definidos pelo usuário `x-amz-meta-*`) + até 10 **tags** + **version ID** (com versionamento).
- **Chave:** o nome completo do objeto, como `fotos/2026/praia.jpg`. **Não existem pastas**: `fotos/2026/` é só um **prefixo** da chave. O console exibe os prefixos como pastas para facilitar a navegação.
- **Consistência forte:** depois de um `PUT` ou `DELETE` bem-sucedido, o próximo `GET` ou `LIST` já reflete a mudança, para todos os clientes.
- **Tamanho dos uploads:** um `PUT` único aceita até **5 GB**. Acima disso, o **multipart upload** é obrigatório, e ele é recomendado a partir de 100 MB.
- **Tipos de bucket:**
  - **general purpose:** o bucket "normal", usado em quase todos os cenários;
  - **directory bucket:** usado pelo S3 Express One Zone, em uma única AZ;
  - **table bucket** (S3 Tables) e **vector bucket** (S3 Vectors): para tabelas Apache Iceberg e vetores de IA.

![Diagrama S3 - Buckets, objetos e chaves](./img/amazon-s3/amazon-s3-01.png)

[↑ Índice](#índice)

### Classes de armazenamento do S3

Todas as classes têm **durabilidade de 11 noves**. O que muda é a **latência de leitura**, a **disponibilidade**, os **mínimos cobrados** e a relação entre o custo de **guardar** e o custo de **ler**:

| Classe | Primeiro byte | AZs | Mínimos cobrados | Taxa de recuperação | Uso típico |
|---|---|---|---|---|---|
| **S3 Express One Zone** | ms de **um dígito** | **1** | — | Não | ML, analytics e processamento intensivo, com o compute na mesma AZ |
| **S3 Standard** | ms | ≥3 | — | Não | Dados acessados com frequência: aplicações, sites, data lakes "quentes" |
| **S3 Intelligent-Tiering** | ms (camadas opcionais: horas) | ≥3 | Taxa de monitoramento por objeto | Não | **Padrão de acesso desconhecido ou que muda** |
| **S3 Standard-IA** | ms | ≥3 | **30 dias**, **128 KB** | Sim | Backups e cópias de DR acessados raramente, mas lidos na hora |
| **S3 One Zone-IA** | ms | **1** | 30 dias, 128 KB | Sim | Cópias secundárias e dados que podem ser recriados |
| **S3 Glacier Instant Retrieval** | **ms** | ≥3 | **90 dias**, 128 KB | Sim, mais cara | Arquivo acessado ~1 vez por trimestre que precisa de leitura imediata (imagens médicas, acervos de mídia) |
| **S3 Glacier Flexible Retrieval** | **minutos a horas** | ≥3 | **90 dias** | Sim | Backup e arquivo acessados 1-2 vezes por ano |
| **S3 Glacier Deep Archive** | **horas** (até 12 h; Bulk até 48 h) | ≥3 | **180 dias** | Sim | Retenção de 7-10 anos por compliance; substitui fitas |

**Como ler as classes Glacier:**

- **Glacier Instant Retrieval** é lido como qualquer objeto, direto pela API.
- **Glacier Flexible Retrieval** e **Deep Archive** exigem um **restore** antes da leitura. O restore cria uma **cópia temporária** do objeto, que fica disponível por N dias. As opções de velocidade são:
  - Flexible: **Expedited** (1-5 min), **Standard** (3-5 h) e **Bulk** (5-12 h, gratuito);
  - Deep Archive: **Standard** (até 12 h) e **Bulk** (até 48 h).
- O evento `s3:ObjectRestore:Completed` avisa quando o restore termina.

**Como escolher:**
- acesso imprevisível → **Intelligent-Tiering**;
- dado que **pode ser recriado** → classes **One Zone**;
- precisa ler **na hora**, mas quase nunca → **Standard-IA** ou **Glacier Instant Retrieval**;
- aceita esperar **horas** → **Glacier Flexible** ou **Deep Archive**.

![Diagrama S3 - Classes de armazenamento](./img/amazon-s3/amazon-s3-02.png)

> **Obs:** a cobrança mínima é uma pegadinha comum. Um objeto apagado ou movido do Standard-IA com 10 dias **paga os 30 dias**. Objetos menores que 128 KB pagam como 128 KB no IA. Para milhões de objetos pequenos, as classes IA podem sair **mais caras** que o Standard.

[↑ Índice](#índice)

### Lifecycle rules e Intelligent-Tiering

**Lifecycle rules** automatizam a mudança de classe e a exclusão dos objetos com o tempo. Cada regra se aplica a um **prefixo**, a **tags** ou a uma faixa de **tamanho** de objeto:

- **Transition actions:** mudam a classe depois de N dias da criação (ex: Standard → Standard-IA aos 30 dias → Glacier Flexible aos 90 → Deep Archive aos 365).
- **Expiration actions:** apagam o objeto depois de N dias (ex: logs com 7 anos).
- **Versões não atuais:** com versionamento, regras próprias para as versões antigas (ex: mover para Glacier aos 30 dias e apagar aos 365, ou manter só as 3 mais recentes) e para remover **delete markers** que ficaram sem versões.
- **Multipart uploads incompletos:** uma regra para **abortá-los** (ex: após 7 dias). As partes já enviadas são cobradas e não aparecem na listagem comum.
- **Restrições:** os objetos precisam ficar **30 dias** no Standard antes de ir para Standard-IA ou One Zone-IA, e as transições seguem um sentido: das classes mais "quentes" para as mais "frias".

**S3 Intelligent-Tiering** faz o mesmo trabalho **sozinho**, objeto por objeto, de acordo com o **último acesso**:

- **Frequent Access** (padrão) → **Infrequent Access** após **30 dias** sem acesso → **Archive Instant Access** após **90 dias**;
- camadas **opcionais**, ativadas por configuração: **Archive Access** (90+ dias) e **Deep Archive Access** (180+ dias), que exigem restore;
- ao ser acessado, o objeto volta para **Frequent Access**;
- **não há taxa de recuperação**; paga-se uma pequena **taxa de monitoramento por objeto**. Objetos menores que **128 KB** não são monitorados e ficam sempre em Frequent.

**Lifecycle ou Intelligent-Tiering?**
- padrão de acesso **previsível**, como logs que ninguém lê depois de 30 dias → **lifecycle rules**, que é mais barato;
- padrão **desconhecido ou variável** → **Intelligent-Tiering**.

Para descobrir o padrão, use o **Storage Class Analysis**, que observa os acessos e recomenda quando mover objetos do Standard para o Standard-IA, ou o **S3 Storage Lens**.

![Diagrama S3 - Lifecycle rules e Intelligent-Tiering](./img/amazon-s3/amazon-s3-03.png)

[↑ Índice](#índice)

### Versionamento e proteção de dados

- **Versionamento:** cada `PUT` cria uma **nova versão** do objeto, e cada `DELETE` cria um **delete marker** em vez de apagar. A versão anterior pode ser recuperada. Uma vez ativado, o versionamento só pode ser **suspenso**, não desativado. É **pré-requisito da replicação** e do **Object Lock**.
- **MFA Delete**, **Object Lock** (WORM, modos governance e compliance) e **Legal hold:** ver [S3 Object Lock (WORM)](security.md#s3-object-lock-worm) e [Amazon S3 Security](security.md#amazon-s3-security).
- **AWS Backup para S3:** backups contínuos (restauração a um ponto no tempo nos últimos 35 dias) ou periódicos, com cópia entre contas e **Vault Lock**. É útil quando a governança de backup é centralizada para vários serviços.

[↑ Índice](#índice)

### Replicação (CRR e SRR)

A replicação copia objetos **de forma assíncrona** de um bucket para outro:

- **CRR (Cross-Region Replication):** para outra região. Atende **DR**, **compliance** (cópia em outra geografia) e **latência** para usuários de outra região.
- **SRR (Same-Region Replication):** na mesma região. Atende **agregação de logs** de vários buckets, cópia de **produção para teste** e cópia para **outra conta** sem mudar de região.

**Requisitos e opções:**

- **Versionamento ativo** na origem e no destino, e uma **IAM role** que o S3 assume para replicar.
- O destino pode estar em **outra conta**. Nesse caso, é preciso uma bucket policy no destino; a opção **owner override** torna a conta de destino dona das réplicas.
- Pode **trocar a classe** no destino (ex: replicar direto para Glacier Deep Archive na região de DR).
- Objetos criptografados com **SSE-KMS** são replicados se a regra indicar a chave KMS do destino.
- **S3 Replication Time Control (RTC):** **99,99%** dos objetos replicados em até **15 minutos**, com SLA e métricas. É a resposta para "RPO de replicação previsível".

**Regras que mais caem:**

- Só objetos **novos** (criados depois da regra) são replicados. Para os **existentes** ou os que falharam, use o **S3 Batch Replication**.
- **Não há encadeamento:** com A → B → C, os objetos criados em A não chegam a C.
- **Delete markers** podem ser replicados (opcional). A exclusão de uma **versão específica** **nunca** é replicada, o que protege o destino contra exclusão maliciosa na origem.
- **Replicação bidirecional:** duas regras (A → B e B → A) + **replica modification sync**. Costuma ser usada com **Multi-Region Access Points** para arquiteturas ativo/ativo. Ver [Multi-Region Access Points](security.md#multi-region-access-points).

![Diagrama S3 - Replicação CRR e SRR](./img/amazon-s3/amazon-s3-04.png)

[↑ Índice](#índice)

### Desempenho do S3

- **Escala por prefixo:** cada prefixo suporta pelo menos **3.500** `PUT/COPY/POST/DELETE` e **5.500** `GET/HEAD` por segundo, e não há limite de prefixos por bucket. Para mais requisições, distribua as chaves entre **vários prefixos**. O S3 escala particionando os prefixos automaticamente, mas um aumento brusco pode gerar erros **503 Slow Down** temporários; a aplicação deve fazer **retry com backoff**.
- **Multipart upload:** divide o arquivo em partes (até 10.000) enviadas **em paralelo**. Uma parte que falha é reenviada sozinha. É obrigatório acima de 5 GB e recomendado acima de 100 MB.
- **S3 Transfer Acceleration:** o cliente envia para a **edge location** mais próxima, e os dados seguem pela **rede da AWS** até o bucket. Usa um endpoint próprio (`bucket.s3-accelerate.amazonaws.com`) e só cobra quando de fato acelera. É a resposta para **uploads de usuários distantes** da região do bucket.
- **Byte-range fetches:** o cabeçalho `Range` baixa **partes** do objeto em paralelo (mais throughput) ou só um trecho (ex: o cabeçalho de um arquivo).
- **Leituras com muito acesso repetido:** **CloudFront** na frente do bucket, que armazena em cache nas edge locations e reduz requisições ao S3.
- **Latência mínima:** **S3 Express One Zone** (directory buckets), com latência de milissegundos de um dígito e centenas de milhares de requisições por segundo, colocando o compute na **mesma AZ**.
- **Consultas dentro do objeto:** para filtrar dados sem baixar o objeto inteiro, use o **Athena**. O S3 Select foi fechado para novos clientes.

![Diagrama S3 - Multipart, Transfer Acceleration, byte-range e prefixos](./img/amazon-s3/amazon-s3-05.png)

[↑ Índice](#índice)

### Eventos do S3

O S3 pode avisar outros serviços quando algo acontece no bucket: objeto criado, removido, restaurado do Glacier, replicação que falhou, entre outros.

- **Event Notifications** (nativo):
  - destinos: **SNS**, **SQS** (fila standard) e **Lambda**;
  - filtro só por **prefixo e sufixo** da chave (ex: `uploads/` e `.jpg`);
  - o destino precisa de uma **resource policy** que permita o S3 publicar.
- **Amazon EventBridge** (ativado no bucket):
  - **todos** os eventos do S3 vão para o EventBridge;
  - as regras filtram por **metadados, tamanho, chave** e outros campos;
  - **vários destinos** por evento (Step Functions, Kinesis, API destinations, outra conta ou região);
  - **archive e replay** dos eventos.
- **Qual escolher:**
  - para um gatilho simples (ex: gerar a miniatura de uma imagem com Lambda), **Event Notifications** bastam;
  - para filtros avançados, vários consumidores ou orquestração, use o **EventBridge**.
- **Entrega:** os eventos chegam normalmente em segundos, mas podem ser **duplicados** ou, raramente, perdidos. O processamento deve ser **idempotente**.

![Diagrama S3 - Event Notifications e EventBridge](./img/amazon-s3/amazon-s3-06.png)

[↑ Índice](#índice)

### Operações em larga escala

- **S3 Inventory:** um relatório **diário ou semanal** (CSV, ORC ou Parquet) com a lista dos objetos e seus metadados: classe, tamanho, criptografia, status de replicação e de Object Lock. É mais barato e rápido que chamar `LIST` em bilhões de objetos, e pode ser consultado com o **Athena**.
- **S3 Batch Operations:** executa **uma operação em bilhões de objetos** a partir de uma lista (o Inventory ou um CSV), com retries, acompanhamento e **relatório de conclusão**. Operações possíveis:
  - **copiar** objetos (ex: para **criptografar objetos existentes** ou mudar a classe);
  - **invocar uma Lambda** para cada objeto;
  - substituir **tags**, metadados ou ACLs;
  - **restaurar** do Glacier;
  - aplicar **Object Lock**;
  - **Batch Replication**.
- **S3 Storage Lens:** um painel de **uso e atividade** de todos os buckets da **organização**, por conta, região, bucket e prefixo. Traz recomendações de custo e proteção de dados (ex: buckets sem versionamento, multipart uploads incompletos). As métricas gratuitas são básicas; as **avançadas** são pagas.

[↑ Índice](#índice)

### Site estático e Requester Pays

- **Static website hosting:**
  - o bucket serve HTML, CSS, JS e imagens por um **website endpoint** (`bucket.s3-website-<região>.amazonaws.com`), com página de índice, página de erro e regras de redirecionamento;
  - o objeto precisa ser **público**, e o endpoint só fala **HTTP**;
  - para **HTTPS**, domínio próprio com certificado, cache e um bucket **privado**, coloque o **CloudFront com Origin Access Control (OAC)** na frente. Ver [CloudFront com Origin Access Control (OAC)](security.md#cloudfront-com-origin-access-control-oac).
- **Requester Pays:** quem **baixa** os dados paga as **requisições e a transferência**, e não o dono do bucket. É usado para compartilhar **grandes datasets** com outras contas. O solicitante precisa estar autenticado na AWS, sem acesso anônimo.

[↑ Índice](#índice)

### Segurança do S3

A segurança do S3 está em [Amazon S3 Security](security.md#amazon-s3-security) e em [Amazon S3 Access Points](security.md#amazon-s3-access-points). Os pontos que mais se cruzam com storage são:

- **Bucket policies** e **IAM policies**; **Block Public Access** ativado por padrão; **ACLs desabilitadas** por padrão (Object Ownership = bucket owner enforced).
- **Criptografia:** **SSE-S3** é aplicada por padrão a todo objeto novo. As alternativas são SSE-KMS, DSSE-KMS e SSE-C. O **S3 Bucket Key** reduz as chamadas e o custo do KMS. Para criptografar **objetos já existentes**, use o **Batch Operations** (copy).
- **Acesso privado a partir da VPC:** **gateway endpoint**, gratuito e só para a mesma região, ou **interface endpoint**, pago e acessível de on-premises e de outras regiões.
- **Pre-signed URLs:** acesso temporário a um objeto sem credenciais AWS (ex: download ou upload direto pelo browser).

[↑ Índice](#índice)

### Custo do S3

O custo do S3 tem cinco componentes:

1. **Armazenamento:** GB/mês, de acordo com a classe.
2. **Requisições:** `PUT`, `COPY`, `POST` e `LIST` custam mais que `GET`. Nas classes IA e Glacier, as requisições são mais caras.
3. **Recuperação:** por GB lido nas classes IA e Glacier.
4. **Transferência:** **entrada** gratuita; **saída** para a internet é paga; para o **CloudFront** é gratuita; entre **regiões** é paga, o que inclui a replicação CRR; dentro da mesma região, para EC2, é gratuita.
5. **Gestão e recursos:** Inventory, Storage Lens avançado, monitoramento do Intelligent-Tiering, RTC e Transfer Acceleration.

**Formas de reduzir:**
- **lifecycle rules** ou **Intelligent-Tiering**;
- abortar **multipart uploads incompletos**;
- apagar **versões antigas**;
- **gateway endpoint** em vez de NAT Gateway para o tráfego da VPC ao S3;
- **CloudFront** para reduzir a saída de dados;
- **Requester Pays** para datasets compartilhados.

[↑ Índice](#índice)

### Dicas para a prova

- "Padrão de acesso **desconhecido ou variável**, reduzir custo sem esforço operacional" → **S3 Intelligent-Tiering**.
- "Logs acessados nos primeiros 30 dias e guardados por 7 anos" → **lifecycle**: Standard → Standard-IA → Glacier Flexible/Deep Archive → **expiração**.
- "Arquivo raramente acessado que precisa ser lido **em milissegundos**" → **Glacier Instant Retrieval** (ou Standard-IA, se o acesso for mais frequente).
- "Retenção de 7-10 anos ao **menor custo**, leitura em até 48 h" → **Glacier Deep Archive**.
- "Dados que podem ser **recriados**, acesso raro, menor custo" → **One Zone-IA**.
- "Milhões de objetos **pequenos**" → cuidado: IA e Glacier cobram mínimo de **128 KB** (e metadados extras no Glacier). O Standard pode sair mais barato.
- "Custo de armazenamento crescendo sem explicação" → **multipart uploads incompletos** e **versões antigas**: lifecycle para abortar e expirar. Diagnóstico com o **Storage Lens**.
- "Cópia em outra região para DR/compliance" → **CRR**. "Com RPO previsível de 15 minutos" → **CRR + RTC**. "Objetos que já existiam no bucket" → **S3 Batch Replication**.
- "Proteger as réplicas contra exclusão maliciosa na conta de origem" → replicação para **outra conta** (+ **Object Lock** no destino). Exclusões de versões não são replicadas.
- "Ativo/ativo com buckets em várias regiões e um endpoint único" → **Multi-Region Access Points** + **replicação bidirecional**.
- "Uploads lentos de usuários em outro continente" → **Transfer Acceleration**. "Arquivos grandes" → **multipart upload**. "Download mais rápido de arquivos grandes" → **byte-range fetches**.
- "Erros **503 Slow Down** com muitas requisições" → distribuir chaves em **mais prefixos** + retry com backoff.
- "Latência de **milissegundos de um dígito** para treino de ML ou analytics intensivo" → **S3 Express One Zone**.
- "Processar cada arquivo enviado (miniatura, antivírus, ETL)" → **Event Notifications** → Lambda/SQS. "Vários consumidores, filtros por metadados" → **EventBridge**.
- "Executar uma operação em **bilhões de objetos** (copiar, criptografar existentes, mudar tags, restaurar)" → **S3 Batch Operations** com lista do **S3 Inventory**.
- "Visão de uso e custo do S3 em **toda a organização**" → **S3 Storage Lens**.
- "Site estático com **HTTPS** e domínio próprio" → **CloudFront + OAC** na frente do bucket (o website endpoint é só HTTP).
- "Compartilhar um dataset enorme sem pagar a transferência" → **Requester Pays**.
- "Instâncias em subnet privada acessando o S3 **sem NAT**, sem custo" → **gateway endpoint**.

[↑ Índice](#índice)

## AWS DataSync

### Contexto: mover dados para a AWS

Escolher onde guardar os dados (EBS, EFS, FSx, S3) é só metade do problema. A outra metade é **levar os dados até lá**. Em uma migração, as empresas costumam ter **terabytes ou petabytes** em NAS, file servers Windows, clusters Hadoop ou em outra nuvem. Copiar isso com `rsync`, `robocopy` ou `aws s3 cp` funciona em pequena escala, mas, em grande escala, exige:

- servidores e scripts para **paralelizar** a cópia e usar toda a banda disponível;
- **reiniciar** do ponto onde parou depois de uma falha de rede;
- **verificar** que os dados chegaram íntegros;
- preservar **permissões, donos e datas** dos arquivos;
- **sincronizar** as mudanças que acontecem enquanto a migração não termina;
- **limitar a banda** para não derrubar a rede da empresa em horário comercial.

O **AWS DataSync** é o serviço gerenciado que faz tudo isso. Na prova, ele é a resposta padrão para **transferência online** de arquivos e objetos para (ou de, ou entre) os storages da AWS. O desafio é diferenciá-lo de serviços parecidos, como **Snowball**, **Storage Gateway**, **Transfer Family** e **S3 Replication**.

[↑ Índice](#índice)

### O que é o DataSync

Um serviço de **transferência de dados online**, que copia arquivos e objetos entre um storage de **origem** e um de **destino**:

- **Velocidade:** usa protocolo próprio, com paralelismo e compressão. Uma única task consegue usar **todo um link de 10 Gbps**.
- **Segurança:** os dados trafegam com **TLS** e são **verificados** (checksums) durante e no fim da transferência.
- **Incremental:** cada execução copia **só o que mudou** desde a anterior. Isso permite uma carga inicial seguida de sincronizações até o dia do **cutover**.
- **Metadados:** preserva permissões **POSIX** (UID, GID, modo) e datas em NFS/EFS, e **ACLs NTFS** (donos, DACLs, SACLs) de SMB para o FSx for Windows. No S3, os metadados de arquivo viram **metadados do objeto**.
- **Gerenciado:** agendamento, filtros, limite de banda, logs e métricas, sem servidores de cópia para manter.

**Usos típicos:**

- **migrar** file servers e NAS para S3, EFS ou FSx;
- **arquivar** dados frios on-premises direto em classes **Glacier** do S3;
- **replicar** dados para a AWS como cópia de proteção ou DR;
- mover dados entre **serviços da AWS** (ex: EFS → FSx, S3 → EFS) ou **entre contas e regiões**;
- trazer dados de **outras nuvens** (Azure Blob, Azure Files, Google Cloud Storage);
- levar dados para processamento na nuvem (analytics, ML) e trazer os resultados de volta.

[↑ Índice](#índice)

### Componentes: agente, locations e tasks

- **Agente:** uma máquina virtual que **lê ou grava** no storage que a AWS não alcança sozinha, como NAS, file servers, HDFS e object storage on-premises. Ele roda em **VMware ESXi, Hyper-V, KVM ou Nutanix AHV**, ou como **AMI em EC2**. O agente só faz conexões **de saída** (porta 443) para o serviço DataSync, então não é preciso abrir portas de entrada no firewall. Para mais throughput, use **vários agentes** ou várias tasks.
- **Location:** a definição de **onde** estão os dados: tipo (NFS, SMB, S3, EFS etc.), caminho e credenciais. Toda task tem uma location de **origem** e uma de **destino**.
- **Task:** liga origem e destino e define **o que** copiar e **como** (filtros, verificação, metadados, banda, agendamento). Cada vez que a task roda, cria uma **task execution**, que passa pelas fases *preparing* (lista e compara), *transferring* e *verifying*.

![Diagrama DataSync - Arquitetura híbrida](./img/aws-datasync/aws-datasync-01.png)

**Quando o agente é necessário:**

| Cenário | Agente? |
|---|---|
| NFS, SMB, HDFS ou object storage **on-premises** | **Sim**, perto da origem (VM no data center) |
| Entre serviços AWS (**S3, EFS, FSx**) na **mesma conta** | **Não** |
| **S3 → S3** em outra **conta** ou **região** | **Não** (role do DataSync + bucket policy) |
| **EFS/FSx** em **outra conta** | **Sim**, agente em **EC2**, que monta o file system |
| **Outra nuvem** (Azure Blob, GCS) → S3 | **Não** no modo Enhanced; no Basic, agente em EC2 ou na outra nuvem |

> **Obs:** o agente fica **próximo da origem**, porque o tráfego entre o agente e o storage usa protocolos de LAN (NFS/SMB), enquanto o trecho agente → AWS usa o protocolo otimizado do DataSync. Um agente em EC2 lendo um NAS on-premises pela VPN seria lento.

[↑ Índice](#índice)

### Modos de task: Basic e Enhanced

| | **Basic** | **Enhanced** |
|---|---|---|
| Como processa | Fases em **sequência** (prepara, transfere, verifica) | Fases **em paralelo** |
| Quantidade de arquivos | Sujeita a **cotas** por execução | **Praticamente ilimitada** |
| Verificação | Por padrão, verifica **todos** os dados | Verifica **só os transferidos** |
| Logs | Não estruturados | **Estruturados (JSON)** e mais métricas |
| Locations | Todas | Todas, menos S3 on Outposts |
| Custo | Por GB transferido | Por GB transferido **+ taxa por execução** |

- O modo é escolhido **na criação** da task e **não pode ser alterado** depois.
- NFS, SMB e HDFS on-premises exigem um **agente compatível com Enhanced** para usar esse modo.
- Na prova, a regra prática: **datasets enormes** (milhões ou bilhões de arquivos) ou **outras nuvens sem agente** → **Enhanced**.

[↑ Índice](#índice)

### Opções da task

![Diagrama DataSync - Location, task e execução](./img/aws-datasync/aws-datasync-02.png)

- **O que copiar:** **filtros** de inclusão e exclusão por caminho (ex: só `/projetos`, sem `*.tmp`) ou um **manifest** com a lista exata de arquivos.
- **Como copiar:**
  - **só dados alterados** (padrão) ou **tudo**;
  - **manter ou apagar** no destino arquivos que não existem mais na origem (para um espelho exato);
  - **sobrescrever ou não** arquivos do destino;
  - quais **metadados** preservar (dono, permissões, datas, tags de objeto).
- **Verificação de integridade:** checksums durante a transferência e, opcionalmente, uma comparação no fim, dos arquivos transferidos ou de todo o dataset.
- **Limite de banda:** em MB/s, ajustável **durante a execução**. Assim, a migração não satura o link da empresa em horário comercial.
- **Agendamento:** execuções **de hora em hora, diárias ou semanais** (ou sob demanda, por API, Lambda ou EventBridge Scheduler). Não é replicação contínua em tempo real.
- **Classe de armazenamento no S3:** o DataSync grava direto em qualquer classe, inclusive **Glacier Flexible Retrieval** e **Deep Archive**.

> **Obs:** o DataSync **não é replicação em tempo real**. Para mudanças contínuas e com RPO baixo, use **S3 Replication**, **EFS Replication** ou **SnapMirror** (FSx for ONTAP). O DataSync sincroniza em **execuções agendadas**.

[↑ Índice](#índice)

### Rede e segurança do DataSync

- **Endpoint do serviço:** o agente pode se conectar a um endpoint:
  - **público** (pela internet, com TLS);
  - **FIPS** (para requisitos regulatórios);
  - **VPC endpoint (PrivateLink)**: o tráfego sai do data center por **Direct Connect ou VPN** e entra na VPC por uma interface privada, **sem passar pela internet**. É a resposta quando o cenário exige que os dados **não trafeguem pela internet pública**.
- **O tipo de endpoint é escolhido na ativação do agente** e não muda depois.
- **Acesso aos destinos AWS:**
  - **S3:** por uma **role IAM** que o DataSync assume;
  - **EFS e FSx:** o DataSync cria **ENIs na sua VPC** e monta o file system, então os **security groups** precisam liberar NFS (2049) ou SMB (445). Para o EFS, é possível usar **access points** e **autorização IAM**.
- **Credenciais da origem** (usuário SMB, chaves de object storage, Azure) podem ficar no **Secrets Manager**.
- **Criptografia:** TLS em trânsito. Em repouso, vale a criptografia do destino (SSE-S3, SSE-KMS, KMS no EFS/FSx).

[↑ Índice](#índice)

### Transferências entre serviços e contas da AWS

O DataSync também move dados **dentro da AWS**, sem agente na maioria dos casos:

- **Mesma conta:** entre S3, EFS e FSx, em qualquer combinação, inclusive **entre regiões**. Exemplos:
  - **EFS → FSx for OpenZFS** para ganhar desempenho;
  - **EFS → novo EFS criptografado**;
  - **S3 → EFS** para uma aplicação que precisa de arquivos.
- **S3 → S3 em outra conta:**
  1. a task fica na conta **de origem**, com uma **role do DataSync**;
  2. a **bucket policy** do bucket de destino **libera essa role**;
  3. o bucket de destino fica com **ACLs desativadas** (Object Ownership = bucket owner enforced), para que a conta de destino seja dona dos objetos;
  4. se os objetos usam **SSE-KMS**, a key policy também precisa liberar a role.
- **EFS/FSx em outra conta:** um **agente em EC2** na conta que roda a task monta o file system da outra conta, por VPC peering ou Transit Gateway, como se fosse um NAS on-premises.

![Diagrama DataSync - Transferências entre contas, serviços e nuvens](./img/aws-datasync/aws-datasync-03.png)

> **Obs:** para **copiar uma vez** um bucket inteiro para outra conta, o DataSync é uma boa opção. Para **manter** os buckets sincronizados continuamente, a resposta é **S3 Replication** (CRR/SRR). Para objetos **já existentes** com a replicação configurada, a resposta é **S3 Batch Replication**.

[↑ Índice](#índice)

### Monitoramento e custo do DataSync

- **CloudWatch:** métricas de bytes e arquivos transferidos, e logs da execução no **CloudWatch Logs** (inclusive de cada arquivo, no nível de log `TRANSFER`).
- **EventBridge:** eventos de **mudança de estado** da task e da execução. Por exemplo, para disparar o processamento quando a cópia terminar.
- **Task reports:** um relatório detalhado de **cada arquivo** (transferido, ignorado, verificado, com erro), gravado no **S3**. É a resposta para **auditoria** e prova de que a migração foi completa.
- **Custo:**
  - uma taxa **por GB transferido** (fixa, maior no modo Enhanced);
  - no **Enhanced**, uma **taxa por execução** da task;
  - o agente **não é cobrado**;
  - à parte: requisições do S3 (LIST, GET, PUT), armazenamento do destino, transferência entre regiões ou para fora da AWS, PrivateLink e CloudWatch.

> **Obs:** em uma migração para o **S3** com **milhões de arquivos pequenos**, o custo das **requisições** (PUT) pode superar o custo por GB. Uma alternativa é agrupar os arquivos antes, quando a aplicação permitir.

[↑ Índice](#índice)

### DataSync vs. outras opções de transferência

| Serviço | Quando usar | Palavras do cenário |
|---|---|---|
| **AWS DataSync** | Cópia **online**, única ou **agendada**, de arquivos e objetos para, de ou entre S3, EFS e FSx | "migrar NAS", "sincronizar", "verificar integridade", "preservar permissões", "limitar banda" |
| **AWS Snowball Edge** | Transferência **offline** em dispositivo físico, quando a rede é lenta ou inexistente | "pouca banda", "levaria semanas", "petabytes", "local remoto sem conexão" |
| **AWS Storage Gateway** | Acesso **híbrido contínuo**: aplicações on-premises continuam usando NFS/SMB/iSCSI com **cache local**, e os dados ficam no S3 | "manter acesso on-premises", "cache local", "backup em fita virtual" |
| **AWS Transfer Family** | **Parceiros** e sistemas legados enviam arquivos por **SFTP, FTPS, FTP ou AS2** para o S3 ou o EFS | "parceiros", "SFTP", "sem mudar o processo de envio" |
| **S3 Replication** | Cópia **contínua** e assíncrona de objetos **S3 → S3** | "toda nova versão", "outra região", "RPO de 15 min" (RTC) |
| **S3 Transfer Acceleration** | Uploads de usuários **distantes** pela rede de borda da AWS | "upload global lento" |
| **EFS Replication / SnapMirror** | Réplica contínua de **file systems** para DR | "DR do EFS", "RPO baixo", "ONTAP" |

![Diagrama DataSync - Como levar os dados para a AWS](./img/aws-datasync/aws-datasync-04.png)

**Calculando se a rede basta:** 100 TB em um link de **1 Gbps** com uso total levam cerca de **9 a 10 dias** (100 × 8.000 Gb ÷ 1 Gbps ≈ 800.000 s). Se o prazo é menor, ou se o link não pode ser usado inteiro, a resposta tende a ser **Snowball**. Um padrão comum combina os dois: **Snowball** leva a **carga inicial**, e o **DataSync** sincroniza as **mudanças incrementais** até o cutover.

> **Obs:** o **AWS DataSync Discovery**, que analisava o storage on-premises para recomendar o destino na AWS, foi descontinuado. Em questões antigas, ele aparece como resposta para "avaliar o storage on-premises antes da migração".

[↑ Índice](#índice)

### Dicas para a prova

- "Migrar NAS/file server **NFS ou SMB** on-premises para S3, EFS ou FSx, **online**" → **DataSync** com **agente** on-premises.
- "Preservar **permissões e ACLs** do Windows na migração para o FSx for Windows" → **DataSync** (SMB → FSx for Windows), ou Robocopy.
- "Os dados **não podem trafegar pela internet** pública" → agente com **VPC endpoint (PrivateLink)** sobre **Direct Connect/VPN**.
- "Migração não pode **saturar o link** em horário comercial" → **limite de banda** na task (ou agendamento fora do horário).
- "Garantir que todos os arquivos chegaram íntegros" / "auditar a migração" → **verificação de integridade** + **task reports** no S3.
- "Carga inicial e depois **sincronizar mudanças** até o cutover" → **DataSync agendado** (execuções incrementais). Com pouca banda → **Snowball** + DataSync.
- "Copiar **milhões/bilhões** de arquivos" ou "de **Azure Blob/GCS** para o S3 **sem agente**" → DataSync no modo **Enhanced**.
- "Mover dados entre **EFS, FSx e S3**" ou "copiar um file system para **outra região**" → **DataSync**, sem agente.
- "Copiar um bucket para **outra conta** uma vez" → DataSync (role + **bucket policy**). "Manter sincronizado continuamente" → **S3 Replication**.
- "Arquivar dados frios on-premises direto no **Glacier Deep Archive**" → **DataSync** com a classe de armazenamento de destino.
- "Aplicações on-premises **continuam** acessando os arquivos" → **Storage Gateway**, não DataSync.
- "Parceiros enviam arquivos por **SFTP**" → **Transfer Family**, não DataSync.
- "Dezenas de TB, **link lento**, prazo curto" → **Snowball Edge**.

[↑ Índice](#índice)

## AWS Data Exchange

### Contexto: dados de terceiros

Muitas análises dependem de dados que a empresa **não produz**: cotações financeiras, dados de clima, notícias, cadastros de empresas, dados demográficos, dados de saúde anonimizados, localização de estabelecimentos. Esses dados são **vendidos por provedores especializados** (Reuters, Dun & Bradstreet, Foursquare, entre outros). Tradicionalmente, comprar e receber esses dados era trabalhoso:

- negociar um contrato com cada provedor e pagar por fora da nuvem;
- receber arquivos por **FTP**, e-mail ou disco, com formatos e periodicidades diferentes;
- escrever scripts para **buscar as atualizações** e carregar no data lake;
- do lado do provedor: manter servidores de entrega, controlar **quem pode acessar** e **cortar o acesso** quando o contrato acaba.

O **AWS Data Exchange** resolve os dois lados. O assinante **encontra, assina e recebe** dados de terceiros direto na conta AWS, pagando pela fatura da AWS. O provedor **publica e licencia** os dados sem montar infraestrutura de entrega. Na prova, ele é a resposta para **"usar dados de terceiros"** ou **"vender/licenciar dados para outras empresas"** dentro da AWS.

[↑ Índice](#índice)

### O que é o Data Exchange

Um serviço para **trocar dados entre contas AWS de organizações diferentes**, com controle de **direito de acesso** (entitlement):

- **Catálogo:** milhares de produtos de dados listados no **AWS Marketplace**, na categoria *Data*. Muitos são **gratuitos**, inclusive os do programa **Open Data on AWS**.
- **Cobrança:** a assinatura é paga pela **fatura da AWS**, com prazos de **1 a 36 meses**. O uso dos serviços de análise (S3, Athena, Redshift) é cobrado à parte, normalmente.
- **Entrega contínua:** o provedor publica **novas versões** dos dados, e o assinante recebe as atualizações automaticamente.
- **Controle de acesso:** quando a assinatura termina, o acesso aos dados compartilhados é **revogado automaticamente**.
- **Integração:** os dados chegam ao **S3**, ao **Redshift** ou por **API**, prontos para Athena, Glue, EMR, QuickSight e SageMaker.

![Diagrama Data Exchange - Visão geral](./img/aws-data-exchange/aws-data-exchange-01.png)

[↑ Índice](#índice)

### Conceitos: produtos, data sets, revisions e data grants

- **Data set:** o conjunto de dados que o provedor organiza, como "cotações diárias da bolsa".
- **Revision:** uma **versão publicada** do data set. Cada atualização do provedor é uma nova revision. As revisions de arquivos são **imutáveis**, então o assinante sempre sabe que versão usou.
- **Asset:** o dado em si, dentro da revision: um arquivo, uma tabela compartilhada, um endpoint de API.
- **Produto:** o que é listado no Marketplace. Agrupa um ou mais data sets, com descrição, amostras e **ofertas**.
- **Ofertas:**
  - **pública**: preço e prazo visíveis para todos no catálogo;
  - **privada**: condições negociadas para um cliente específico;
  - **BYOS (Bring Your Own Subscription)**: o cliente já tem contrato com o provedor fora da AWS e passa a receber os dados pelo Data Exchange **sem pagar de novo**;
  - **gratuita**.
- **Verificação de assinatura:** o provedor pode exigir que o assinante **peça acesso** e informe dados da empresa e o uso pretendido. O provedor aprova ou recusa. É comum em dados regulados ou sensíveis.
- **Data grant:** compartilha um data set **direto com outra conta AWS**, **sem listar no Marketplace** e sem cobrança, com prazo de acesso definido. Qualquer conta pode criar data grants. É útil para compartilhar dados com parceiros, clientes ou outras unidades da empresa.

> **Obs:** para **vender** dados no Marketplace, o provedor precisa estar registrado como **vendedor do AWS Marketplace** e passar pela qualificação da AWS. Hoje, só empresas sediadas nos **EUA** ou na **União Europeia** podem. Os **data grants** não têm essa exigência.

[↑ Índice](#índice)

### Tipos de data set e formas de entrega

| Tipo | Como o assinante recebe | Cópia na conta do assinante? | Quem paga o uso |
|---|---|---|---|
| **Files** | **Export** das revisions para um bucket S3 próprio (manual, por job, ou **auto-export** a cada nova revision) | **Sim** | O assinante paga o storage da cópia |
| **Amazon S3 data access** | Lê os objetos **no bucket do provedor**, por um **S3 access point**. O bucket usa **Requester Pays** | **Não** (dados no lugar) | O assinante paga **requisições e transferência** |
| **Amazon Redshift** | **Datashare** do Redshift: consulta **somente leitura** das tabelas do provedor, **sem ETL** | **Não** (dados ao vivo) | O assinante paga o **compute** do próprio Redshift |
| **API** | Chama a API do provedor (hospedada no **API Gateway**) pelo SDK da AWS, com **autenticação IAM (SigV4)** | **Não** | Assinatura, e pode haver **preço por chamada** (metered) |
| **AWS Lake Formation** (preview) | Recebe permissões em tabelas do **data lake** do provedor, gerenciadas pelo Lake Formation | **Não** | O assinante paga as consultas |

![Diagrama Data Exchange - Tipos de data set](./img/aws-data-exchange/aws-data-exchange-02.png)

Como escolher, do ponto de vista do cenário:

- **Files:** o assinante quer **uma cópia** para processar livremente, guardar o histórico ou juntar com os próprios dados.
- **S3 data access:** datasets **muito grandes** ou que mudam muito, em que **copiar** seria caro ou lento. O provedor mantém **uma única cópia**, e cada assinante lê no lugar.
- **Redshift:** o assinante já usa Redshift e quer **consultar com SQL na hora**, com dados sempre atualizados, sem pipeline de carga.
- **API:** dados **sob demanda** ou em tempo real (ex: consultar um CNPJ, uma cotação, a previsão do tempo de um ponto).

[↑ Índice](#índice)

### Automação da ingestão

O Data Exchange publica **eventos no EventBridge** quando o provedor publica uma nova revision, por exemplo. Combinado com o **auto-export**, isso permite montar um pipeline sem servidores:

1. O provedor publica uma nova **revision**.
2. O **auto-export** copia os arquivos para o bucket S3 do assinante (criptografados com **SSE-S3 ou SSE-KMS**).
3. Uma regra do **EventBridge** dispara uma **Lambda** ou uma **Step Functions** para validar e transformar os dados, ou iniciar um **Glue crawler**.
4. Os dados ficam disponíveis no Glue Data Catalog para **Athena**, **QuickSight** e **SageMaker**.

![Diagrama Data Exchange - Automação da ingestão](./img/aws-data-exchange/aws-data-exchange-03.png)

> **Obs:** o bucket de destino do auto-export precisa de uma **bucket policy** que permita ao Data Exchange gravar nele. Se usar **SSE-KMS**, a key policy também precisa liberar o serviço.

[↑ Índice](#índice)

### Data Exchange vs. outras formas de compartilhar dados

| Necessidade | Solução |
|---|---|
| **Assinar** dados de terceiros, ou **vender/licenciar** dados com cobrança e controle de acesso | **AWS Data Exchange** (produto no Marketplace) |
| Compartilhar um data set com **outra conta**, com prazo de acesso, **sem cobrança** | Data Exchange **data grant**, ou mecanismos nativos abaixo |
| Dar acesso a um bucket para outra conta da **mesma organização** | **Bucket policy** / **S3 Access Points** / roles entre contas |
| Compartilhar tabelas do Redshift entre clusters ou contas da empresa | **Redshift data sharing** (sem Data Exchange) |
| Governança de acesso a um **data lake** entre contas (tabela, coluna, linha) | **Lake Formation** cross-account |
| Duas empresas **analisam dados combinados sem expor os dados brutos** uma para a outra | **AWS Clean Rooms** |
| Parceiros enviam arquivos para você por **SFTP** | **AWS Transfer Family** |
| Migrar ou sincronizar **seus próprios** dados para a AWS | **DataSync** |

> **Obs:** a diferença entre Data Exchange e Clean Rooms costuma cair na prova. No **Data Exchange**, o dado **é entregue** (ou lido) pelo assinante. No **Clean Rooms**, os dados **não saem** das contas, e só o **resultado** de consultas aprovadas é compartilhado.

[↑ Índice](#índice)

### Dicas para a prova

- "Enriquecer análises com **dados de terceiros**" (financeiros, clima, demográficos) **com o mínimo de esforço** → **AWS Data Exchange**.
- "Empresa quer **vender ou licenciar** seus dados para outras empresas na AWS, sem construir infraestrutura de entrega e cobrança" → **Data Exchange** (produto no Marketplace).
- "Receber **automaticamente** as atualizações do provedor no S3 e processar" → **auto-export** + **EventBridge** + Lambda/Glue.
- "Dataset **muito grande**, evitar copiar para cada assinante" → **Amazon S3 data access** (access point + **Requester Pays**).
- "Consultar dados do provedor com **SQL no Redshift**, sempre atualizados, sem ETL" → **Data Exchange for Amazon Redshift** (datashare).
- "Consumir **API** de terceiros com autenticação e cobrança da AWS" → **Data Exchange for APIs**.
- "Já tem **contrato com o provedor** fora da AWS e quer receber os dados no S3" → oferta **BYOS**.
- "Provedor precisa **aprovar** quem assina (dados sensíveis)" → **verificação de assinatura**.
- "Compartilhar um data set com uma conta de parceiro **por tempo limitado**, sem Marketplace" → **data grant**.
- "Acesso termina quando o contrato acaba" → entitlement do Data Exchange **revoga automaticamente** (exceto cópias já exportadas).
- "Analisar dados com um parceiro **sem que nenhum veja os dados brutos** do outro" → **AWS Clean Rooms**, não Data Exchange.

[↑ Índice](#índice)

## Revisão: qual serviço usar

Resumo das comparações que mais aparecem nas questões. Os detalhes estão nas seções de cada serviço.

### Block storage (EBS)

| Pergunta do cenário | Solução |
|---|---|
| Uso geral, menor custo, IOPS sem aumentar o disco | [gp3](#tipos-de-volume) |
| IOPS extremas e latência mínima | [io2 Block Express](#tipos-de-volume) |
| Sequencial grande e barato / dados frios | [st1 / sc1](#tipos-de-volume) |
| Mesmo volume em várias instâncias da AZ | [Multi-Attach](#elastic-volumes-e-multi-attach) |
| Mover volume entre AZs ou regiões | [Snapshot e copy-snapshot](#snapshots) |
| Volume restaurado lento no início | [Fast Snapshot Restore](#snapshots) |
| Snapshots automáticos vs. backup centralizado multi-conta | [DLM vs. AWS Backup](#snapshots) |
| Criptografar volume existente / todo volume novo | [Snapshot + cópia / encryption by default](#criptografia) |
| Snapshot criptografado para outra conta | [Customer managed key](#criptografia) |

[↑ Índice](#índice)

### File storage (EFS)

| Pergunta do cenário | Solução |
|---|---|
| Arquivos compartilhados entre instâncias e AZs (Linux/NFS) | [EFS](#o-que-é-o-efs) |
| Armazenamento persistente para Fargate / ReadWriteMany no EKS / arquivos para Lambda | [EFS montado em containers e funções](#arquitetura-mount-targets-e-acesso) |
| Dados recriáveis ao menor custo | [EFS One Zone](#regional-vs-one-zone) |
| Maioria dos arquivos raramente acessada | [Lifecycle para IA/Archive](#classes-de-armazenamento-e-lifecycle) |
| Throughput insuficiente / créditos esgotados | [Elastic ou Provisioned](#modos-de-desempenho-e-throughput) |
| Isolar aplicações no mesmo file system | [Access points](#segurança-e-controle-de-acesso) |
| Exigir TLS / criptografar file system existente | [File system policy / novo FS + DataSync](#segurança-e-controle-de-acesso) |
| DR entre regiões vs. recuperar versão anterior | [EFS Replication vs. AWS Backup](#replicação-e-backup-do-efs) |
| Migrar NAS on-premises para o EFS | [DataSync](#migração-e-integração) |

[↑ Índice](#índice)

### File storage (FSx)

| Pergunta do cenário | Solução |
|---|---|
| File share Windows, SMB, Active Directory, alta disponibilidade | [FSx for Windows Multi-AZ](#fsx-for-windows-file-server) |
| Restaurar versões anteriores / caminho único para vários shares | [Shadow Copies / DFS Namespaces](#fsx-for-windows-file-server) |
| HPC/ML com dados no S3 / temporário vs. longo | [FSx for Lustre + DRA / Scratch vs. Persistent](#fsx-for-lustre) |
| Linux e Windows nos mesmos dados / iSCSI / NetApp on-premises | [FSx for NetApp ONTAP](#fsx-for-netapp-ontap) |
| NFS de baixa latência, ZFS, clones instantâneos | [FSx for OpenZFS](#fsx-for-openzfs) |
| Migrar NAS on-premises para FSx | [DataSync / SnapMirror / Robocopy](#migração-para-o-fsx) |
| EFS ou FSx? | [Como escolher](#como-escolher-entre-efs-e-fsx) |

[↑ Índice](#índice)

### Object storage (S3)

| Pergunta do cenário | Solução |
|---|---|
| Acesso imprevisível / previsível ao longo do tempo | [Intelligent-Tiering / lifecycle rules](#lifecycle-rules-e-intelligent-tiering) |
| Raro, mas leitura imediata / retenção longa ao menor custo | [Glacier Instant Retrieval / Deep Archive](#classes-de-armazenamento-do-s3) |
| Dados recriáveis, acesso raro | [One Zone-IA](#classes-de-armazenamento-do-s3) |
| Recuperar versões apagadas ou sobrescritas | [Versionamento](#versionamento-e-proteção-de-dados) |
| Cópia em outra região / RPO de 15 min / objetos existentes | [CRR / RTC / Batch Replication](#replicação-crr-e-srr) |
| Upload de longe / arquivos grandes / 503 Slow Down | [Transfer Acceleration / multipart / prefixos](#desempenho-do-s3) |
| Latência de um dígito de ms | [S3 Express One Zone](#desempenho-do-s3) |
| Reagir a uploads (simples vs. vários consumidores) | [Event Notifications vs. EventBridge](#eventos-do-s3) |
| Operação em bilhões de objetos / visão da organização | [Batch Operations + Inventory / Storage Lens](#operações-em-larga-escala) |
| Site estático com HTTPS / dataset pago por quem baixa | [CloudFront + OAC / Requester Pays](#site-estático-e-requester-pays) |

[↑ Índice](#índice)

### Transferência de dados (DataSync)

| Pergunta do cenário | Solução |
|---|---|
| Migrar NAS NFS/SMB on-premises online | [DataSync + agente](#componentes-agente-locations-e-tasks) |
| Precisa de agente? (on-prem, entre contas, outra nuvem) | [Tabela do agente](#componentes-agente-locations-e-tasks) |
| Milhões de arquivos / outra nuvem sem agente | [Modo Enhanced](#modos-de-task-basic-e-enhanced) |
| Limitar banda / sincronizar até o cutover / arquivar no Glacier | [Opções da task](#opções-da-task) |
| Não trafegar pela internet | [VPC endpoint + Direct Connect/VPN](#rede-e-segurança-do-datasync) |
| Copiar bucket ou file system para outra conta | [Transferências entre contas](#transferências-entre-serviços-e-contas-da-aws) |
| Auditar a migração | [Task reports](#monitoramento-e-custo-do-datasync) |
| DataSync, Snowball, Storage Gateway, Transfer Family ou S3 Replication? | [Comparação](#datasync-vs-outras-opções-de-transferência) |

[↑ Índice](#índice)

### Compartilhamento de dados (Data Exchange)

| Pergunta do cenário | Solução |
|---|---|
| Usar dados de terceiros / vender ou licenciar dados | [Data Exchange](#o-que-é-o-data-exchange) |
| Contrato já existente com o provedor / aprovar assinantes / compartilhar sem Marketplace | [BYOS / verificação / data grant](#conceitos-produtos-data-sets-revisions-e-data-grants) |
| Cópia no S3 vs. ler no lugar vs. SQL no Redshift vs. API | [Tipos de data set](#tipos-de-data-set-e-formas-de-entrega) |
| Receber e processar atualizações automaticamente | [Auto-export + EventBridge](#automação-da-ingestão) |
| Data Exchange, Clean Rooms, Lake Formation ou bucket policy? | [Comparação](#data-exchange-vs-outras-formas-de-compartilhar-dados) |

[↑ Índice](#índice)
