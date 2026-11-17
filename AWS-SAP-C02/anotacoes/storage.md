---
titulo: Storage
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - Amazon EBS
  - Amazon Data Lifecycle Manager
  - AWS Backup
  - Amazon EFS
  - AWS DataSync
criado-em: 2026-11-16
atualizado-em: 2026-11-17
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, storage]
---

# Storage

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## Índice

- **Block storage:** [Amazon EBS](#amazon-elastic-block-store-ebs)
- **File storage:** [Amazon EFS](#amazon-elastic-file-system-efs)
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
