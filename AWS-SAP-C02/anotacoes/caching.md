---
titulo: Caching
certificacao: AWS Certified Solutions Architect - Professional (SAP-C02)
servicos:
  - Amazon CloudFront
criado-em: 2026-11-19
atualizado-em: 2026-11-19
revisoes: []           # ex.: [2026-08-30, 2026-09-13]
tags: [sap-c02, caching, cdn, edge]
---

# Caching

> **Escopo:** _(em uma frase, o que este arquivo cobre)_
>
> **Por que cai na prova:** _(o ângulo que a SAP-C02 costuma explorar)_

## Índice

- **Cache na borda (CDN):** [Amazon CloudFront](#amazon-cloudfront)
- **Revisão:** [Qual serviço usar](#revisão-qual-serviço-usar)

## Amazon CloudFront

### Contexto: por que cache na borda

Uma aplicação hospedada em **uma região** (ex: `us-east-1`) atende bem quem está perto dela. Um usuário em São Paulo ou Tóquio, porém, paga a **latência** da distância: cada requisição atravessa continentes, e cada conexão TLS precisa de várias idas e voltas. Além disso, se milhões de usuários pedem **o mesmo arquivo** (uma imagem, um vídeo, o JavaScript do site), a origem processa e transfere esse arquivo milhões de vezes, o que gera **custo** e **carga** desnecessários.

**Cache** é guardar uma cópia de um dado **perto de quem o usa**, para não buscá-lo de novo na origem. Uma **CDN** (*Content Delivery Network*) aplica essa ideia em escala global: uma rede de servidores espalhados pelo mundo guarda cópias do conteúdo e responde aos usuários **do ponto mais próximo**. Os ganhos são três:

- **latência menor** para o usuário;
- **menos carga e custo** na origem;
- **proteção**: a CDN fica na frente da aplicação e absorve ataques e picos de tráfego.

O **Amazon CloudFront** é a CDN da AWS. Na prova, ele aparece em quase todo cenário de **conteúdo global**, **site estático**, **streaming**, **proteção da origem** e **DDoS**.

[↑ Índice](#índice)

### O que é o CloudFront

- **Distribuição:** a configuração do CloudFront, com um nome DNS (`d111111abcdef8.cloudfront.net`) ou um **domínio próprio** (CNAME/alias no Route 53, com certificado do **ACM em `us-east-1`**). Ela define as **origens**, os **cache behaviors** e as regras de segurança.
- **Conteúdo estático e dinâmico:**
  - **estático** (imagens, CSS, JS, vídeos) é guardado em cache pelo **TTL**;
  - **dinâmico** (APIs, páginas personalizadas) também ganha com o CloudFront mesmo **sem cache**, porque a conexão TLS termina perto do usuário e o tráfego até a origem segue pela **rede privada da AWS**, com conexões reaproveitadas.
- **Leitura e escrita:** além de `GET`, aceita `PUT`, `POST`, `PATCH` e `DELETE`. Por exemplo, para uploads ao S3 ou chamadas de API pela borda.
- **Protocolos:** HTTP/1.1, **HTTP/2**, **HTTP/3** (QUIC), **WebSocket** e **gRPC**. A **viewer protocol policy** pode **redirecionar HTTP para HTTPS** ou exigir HTTPS.
- **Segurança integrada:** **Shield Standard** automático, **AWS WAF**, **geo restriction**, conteúdo privado com **signed URLs/cookies** e **field-level encryption**.
- **Escopo global:** o CloudFront é um serviço **global**, gerenciado a partir de **`us-east-1`**: certificados ACM, web ACLs do WAF, métricas no CloudWatch e funções Lambda@Edge ficam nessa região.

[↑ Índice](#índice)

### Arquitetura: edge locations, regional edge caches e Origin Shield

- **Edge locations (PoPs):** centenas de pontos de presença no mundo. Recebem a requisição do usuário (o DNS do CloudFront responde com a edge mais próxima), terminam o TLS e respondem do cache quando há **cache hit**.
- **Regional edge caches:** uma camada intermediária, com cache **maior**, entre as edge locations e a origem. Guardam objetos **menos populares**, que já saíram do cache das edges, e evitam que a origem seja consultada de novo.
- **Origin Shield** (opcional, cobrado à parte): uma camada **única** de cache, em uma região escolhida perto da origem, por onde passam **todas** as requisições que não encontraram o objeto no cache. Ele aumenta o **cache hit ratio** e **reduz a carga na origem**. É útil quando a origem é cara ou frágil (ex: um servidor on-premises ou de empacotamento de vídeo) ou quando há **várias camadas de CDN**.

![Diagrama CloudFront - Arquitetura](./img/amazon-cloudfront/amazon-cloudfront-01.png)

> **Obs:** em uma **cache miss**, só a primeira requisição de um objeto vai até a origem. Requisições simultâneas pelo mesmo objeto são agrupadas (*request collapsing*), o que protege a origem em picos.

[↑ Índice](#índice)

### Origens

- **Amazon S3:**
  - **bucket** (endpoint REST), privado, com **Origin Access Control (OAC)**: o caso mais comum, para distribuir arquivos e sites estáticos;
  - **endpoint de website** do S3: tratado como **custom origin**, porque o OAC não funciona com ele. É usado quando o site depende de **redirects** e **documentos de índice** do website hosting;
  - também é possível **enviar arquivos** ao S3 pelo CloudFront, usando a borda como entrada.
- **VPC origins:** **ALB, NLB ou instâncias EC2 em subnets privadas**, sem exposição à internet. O CloudFront alcança a origem pela rede da AWS.
- **Custom origins (HTTP/HTTPS):** qualquer servidor acessível pela internet: ALB ou EC2 públicos, **API Gateway**, **Lambda function URLs**, servidores on-premises ou de outra nuvem.
- **Mídia:** **MediaPackage** e **MediaStore**, para vídeo sob demanda e ao vivo.
- **Várias origens na mesma distribuição:** cada **cache behavior** aponta para uma origem (ex: `/static/*` → S3, `/api/*` → ALB).

> **Obs:** o CloudFront **não é** uma solução de replicação. Comparado com a **S3 Cross-Region Replication**, o CloudFront guarda **cópias temporárias** (pelo TTL) em centenas de locais, ideal para conteúdo **estático lido no mundo todo**. A **CRR** mantém **cópias completas e atualizadas** em poucas regiões escolhidas, ideal para dados **dinâmicos** com baixa latência em regiões específicas, ou para DR. Ver [Replicação (CRR e SRR)](storage.md#replicação-crr-e-srr).

[↑ Índice](#índice)

### Proteger a origem

O objetivo é que os usuários **só** acessem o conteúdo **pelo CloudFront**, sem contornar o cache, o WAF e as restrições:

- **S3 → Origin Access Control (OAC):** o bucket fica **privado**. O CloudFront assina as requisições com **SigV4**, e a **bucket policy** só aceita o principal `cloudfront.amazonaws.com` com a condição `AWS:SourceArn` igual ao ARN da distribuição. Com **SSE-KMS**, a key policy também precisa liberar o CloudFront. O **OAI** (*Origin Access Identity*) é o mecanismo **legado**. Detalhes em [CloudFront com OAC](security.md#cloudfront-com-origin-access-control-oac).
- **ALB, NLB e EC2 → VPC origins:** a origem fica em **subnet privada**, sem IP público. É a forma mais forte de garantir que **só o CloudFront** a alcance.
- **Origem pública (ALB ou custom):**
  - o CloudFront envia um **header customizado secreto** (ex: `X-Origin-Verify`), e uma regra no listener do ALB ou no **WAF regional** só aceita requisições com esse header. O segredo deve ser **rotacionado** (Secrets Manager + Lambda);
  - o **security group** da origem libera apenas a **managed prefix list** do CloudFront (`com.amazonaws.global.cloudfront.origin-facing`), em vez de manter a lista de IPs à mão.
- **HTTPS até a origem:** a **origin protocol policy** pode exigir HTTPS entre o CloudFront e a origem, com certificado válido na origem.

![Diagrama CloudFront - Proteger a origem](./img/amazon-cloudfront/amazon-cloudfront-02.png)

[↑ Índice](#índice)

### Cache behaviors, cache key e TTL

- **Cache behaviors:** regras por **caminho** (*path pattern*, como `/images/*` ou `*.jpg`), avaliadas **em ordem**, com um **default (`*`)** no fim. Cada behavior define:
  - a **origem**;
  - os **métodos HTTP** permitidos;
  - a **viewer protocol policy** (HTTP/HTTPS);
  - as **policies** de cache, de requisição e de resposta;
  - as **edge functions**;
  - a restrição a **signed URLs/cookies**.
- **Cache key:** o que diferencia um objeto de outro no cache. Por padrão, só o **domínio e o caminho** da URL. Incluir **headers, cookies ou query strings** na cache key faz o CloudFront guardar **versões diferentes** do objeto. Quanto **mais** itens, **menor** o cache hit ratio. Inclua só o que **realmente muda a resposta** (ex: `Accept-Language` para páginas traduzidas).
- **Policies:**
  - **cache policy:** define a **cache key** e os **TTLs** (mínimo, padrão e máximo). Há policies gerenciadas, como `CachingOptimized` e `CachingDisabled` (para APIs);
  - **origin request policy:** define o que é **enviado à origem** **sem** entrar na cache key (ex: `User-Agent`, cookies de sessão, headers de geolocalização do CloudFront);
  - **response headers policy:** adiciona headers na resposta, como **CORS** e headers de segurança (**HSTS**, CSP, X-Frame-Options).
- **TTL:** por padrão **24 horas**. A origem pode controlar o tempo com os headers `Cache-Control` e `Expires`, dentro dos limites mínimo e máximo da cache policy.
- **Atualizar conteúdo antes do TTL:**
  - **invalidation:** remove objetos do cache por caminho (`/index.html`, `/images/*`). As primeiras **1.000 paths por mês** são gratuitas, e o restante é cobrado;
  - **versionamento de arquivos** (`app.v2.js`, `logo-3f9a.png`): a **prática recomendada**. A nova versão tem outra URL, então não depende de invalidação, permite rollback e evita servir versões misturadas.

![Diagrama CloudFront - Cache behaviors e policies](./img/amazon-cloudfront/amazon-cloudfront-03.png)

> **Obs:** um **cache hit ratio baixo** quase sempre vem de uma **cache key grande demais**, por exemplo encaminhando **todos** os headers, cookies ou query strings. A correção é enxugar a cache key e mover para a **origin request policy** o que a origem precisa receber, mas que não muda a resposta. Uma CloudFront Function também pode **normalizar** a requisição (ex: ordenar query strings, padronizar maiúsculas).

[↑ Índice](#índice)

### Alta disponibilidade: origin failover

- **Origin group:** uma origem **primária** e uma **secundária**. Se a primária responder com um dos **códigos de erro configurados** (ex: 500, 502, 503, 504, 403, 404) ou não responder a tempo, o CloudFront **repete a mesma requisição** na secundária.
- O failover é **por requisição**, sem esperar propagação de DNS, e vale só para **`GET`, `HEAD` e `OPTIONS`**.
- As origens podem estar em **regiões diferentes**. O padrão clássico é **S3 em duas regiões com CRR**, ou **ALBs em duas regiões**, para **alta disponibilidade regional** de conteúdo estático.
- Para **escritas** e failover de toda a aplicação entre regiões, o CloudFront não basta: use **Route 53** (failover, latência) ou **Global Accelerator**.

![Diagrama CloudFront - Origin failover entre regiões](./img/amazon-cloudfront/amazon-cloudfront-04.png)

> **Obs:** páginas de erro personalizadas (**custom error pages**) devolvem um objeto próprio (ex: `/erros/503.html` no S3) quando a origem responde **4xx ou 5xx**. O **Error Caching Minimum TTL** define por quanto tempo o CloudFront guarda o erro em cache antes de tentar a origem de novo (padrão de 10 segundos).

[↑ Índice](#índice)

### Restringir o acesso ao conteúdo

- **Signed URLs e signed cookies:** liberam conteúdo **privado** só para quem a aplicação autorizar (ex: assinantes de um serviço de vídeo, cursos pagos):
  1. o usuário se autentica na **aplicação**;
  2. a aplicação gera uma **URL** ou um **cookie assinado** com a **chave privada**, definindo **expiração** e, opcionalmente, **início**, **IP de origem** e **caminho** (com curinga);
  3. o CloudFront valida a assinatura com a **chave pública**, cadastrada em um **trusted key group**.
- **Signed URL vs. signed cookies:**
  - **signed URL:** um **arquivo** específico (download de um instalador, um vídeo). Funciona com clientes que não aceitam cookies;
  - **signed cookies:** **muitos arquivos** sem mudar as URLs (toda a área de assinantes, segmentos de vídeo HLS/DASH).
- **Políticas:**
  - **canned:** só a expiração;
  - **custom:** data de início, IP e caminhos com curinga.
- **Key groups vs. key pairs:** use **trusted key groups**, gerenciados por IAM. Os **CloudFront key pairs** são **legados** e exigem o usuário **root**.

| | **CloudFront signed URL** | **S3 pre-signed URL** |
|---|---|---|
| Acesso | Pelo **CloudFront** (com cache), a qualquer origem | **Direto no S3**, sem CloudFront |
| Assinatura | Par de chaves do **key group** | **Credenciais IAM** de quem gerou a URL |
| Restrições | Expiração, início, **IP**, caminho com curinga | Só expiração (até 7 dias, ou o fim da sessão da role) |
| Quando usar | Conteúdo **distribuído pelo CloudFront** | Upload ou download pontual **direto no bucket** |

![Diagrama CloudFront - Signed URLs e signed cookies](./img/amazon-cloudfront/amazon-cloudfront-05.png)

- **Geo restriction:** **allow list** ou **block list** de **países**, por distribuição (ex: **direitos autorais** que só permitem exibir um filme em certos países). O país é identificado pelo IP. Para regras mais ricas (país **e** caminho, ou país **e** taxa), use o **geo match do WAF**. Ver [AWS WAF](security.md#aws-web-application-firewall-waf).
- **Field-level encryption:** o CloudFront **criptografa campos específicos** de um `POST` (ex: número de cartão, CPF) com uma **chave pública**, **na borda**. Só o componente da aplicação que tem a **chave privada** consegue ler o campo, mesmo que outras camadas (logs, proxies, microsserviços) vejam a requisição.
- **Segurança na borda:**
  - **AWS WAF** associado à distribuição (web ACL criada em `us-east-1`), que bloqueia os ataques **na borda**, antes da região;
  - **Shield Standard** automático, e **Shield Advanced** para proteção e suporte avançados. Ver [AWS Shield](security.md#aws-shield);
  - **TLS:** certificado do **ACM em `us-east-1`**. Ver [ACM](security.md#aws-certificate-manager-acm). Uma **security policy** define a versão mínima de TLS, e o **SNI** é o padrão (sem custo).

[↑ Índice](#índice)

### Customização na borda: CloudFront Functions e Lambda@Edge

Às vezes é preciso **rodar código** na requisição ou na resposta, perto do usuário: reescrever URLs, validar tokens, adicionar headers, escolher a origem. O CloudFront oferece dois tipos de **edge function**, que **não guardam cache**: só alteram requisições e respostas.

- **Quatro pontos de execução:**
  - **viewer request:** quando o CloudFront **recebe** a requisição do usuário, antes de olhar o cache;
  - **origin request:** antes de **enviar à origem**, só em **cache miss**;
  - **origin response:** quando **recebe a resposta** da origem, antes de guardar em cache;
  - **viewer response:** antes de **devolver** a resposta ao usuário.
- **CloudFront Functions:**
  - **JavaScript leve**, executado nas **edge locations**, só nos eventos **viewer**;
  - tempo **abaixo de 1 ms**, 2 MB de memória, código de até 10 KB, **milhões de requisições por segundo**, custo muito menor;
  - pode ler dados de configuração no **CloudFront KeyValueStore** (ex: tabela de redirects ou flags de A/B).
- **Lambda@Edge:**
  - **Node.js ou Python**, executado nos **regional edge caches**, nos **quatro eventos**;
  - mais tempo (segundos), mais memória (até **10 GB** nos eventos de origem), **acesso à rede**, ao **body** da requisição e ao **SDK da AWS**;
  - a função é criada em **`us-east-1`** e replicada pelo CloudFront.
- As duas podem ser **combinadas** (ex: CloudFront Function no viewer request e Lambda@Edge no origin request), mas **não no mesmo evento**.

| Caso de uso | Use |
|---|---|
| Normalizar a **cache key**, manipular **headers**, **redirects** e **rewrites** de URL | **CloudFront Functions** |
| Validar um **token JWT** sem chamar serviços externos | **CloudFront Functions** |
| Teste **A/B** ou entregar conteúdo por **dispositivo** (`User-Agent`) | **CloudFront Functions** |
| Autenticar consultando o **Cognito** ou um provedor **OIDC**, ou qualquer chamada de rede | **Lambda@Edge** |
| **Escolher a origem** dinamicamente (ex: bucket da região mais próxima do usuário) | **Lambda@Edge** (origin request) |
| Ler ou alterar o **body**, gerar respostas com bibliotecas, processar imagens | **Lambda@Edge** |

![Diagrama CloudFront - CloudFront Functions e Lambda@Edge](./img/amazon-cloudfront/amazon-cloudfront-06.png)

> **Obs:** para cenários de **aplicação global**, o Lambda@Edge pode responder APIs dinâmicas consultando, por exemplo, uma **DynamoDB Global Table** na região mais próxima. Assim, a lógica roda perto do usuário, e só os dados ficam replicados.

[↑ Índice](#índice)

### Monitoramento e implantação

- **Métricas no CloudWatch** (em `us-east-1`): requisições, bytes, **taxas de erro 4xx/5xx** e, com métricas adicionais habilitadas, **cache hit rate** e latência da origem.
- **Standard logs** (access logs): cada requisição registrada no **S3**, no **CloudWatch Logs** ou no **Data Firehose**, com alguns minutos de atraso. São usados para análise com **Athena**.
- **Real-time logs:** enviados em **segundos** para o **Kinesis Data Streams**, com amostragem configurável. São usados para monitoramento e detecção em tempo real.
- **Continuous deployment:** uma **staging distribution** recebe uma **porcentagem do tráfego** ou as requisições com um **header** específico, para testar mudanças de configuração antes de **promovê-las** à distribuição de produção.
- **IPs anycast estáticos:** uma lista fixa de IPs para a distribuição, usada quando clientes ou operadoras precisam liberar **IPs fixos** (ex: *zero-rating* de operadoras móveis).

[↑ Índice](#índice)

### Custo do CloudFront

- **Pay-as-you-go:**
  - **transferência de dados para a internet** (varia por região do mundo; mais barata nos EUA e na Europa);
  - **requisições** HTTP/HTTPS;
  - recursos opcionais: Origin Shield, invalidações além da cota gratuita, real-time logs, edge functions, field-level encryption.
- **Transferência da origem AWS para o CloudFront é gratuita** (S3, ALB, EC2). Servir pelo CloudFront costuma ser **mais barato** que servir direto do S3 ou do EC2 para a internet.
- **Price classes:** limitam as edge locations usadas para **reduzir custo**:
  - **All:** todas, melhor desempenho;
  - **200:** exclui as regiões mais caras;
  - **100:** só as mais baratas (América do Norte e Europa).

  Usuários de regiões fora da classe são atendidos pela edge mais próxima **dentro** da classe, com mais latência.
- **Planos de preço fixo (flat-rate):** desde o fim de 2025, a AWS também oferece planos mensais por distribuição (Free, Pro, Business, Premium), **sem cobrança por excedente**. Eles incluem CloudFront, **WAF**, proteção DDoS, DNS no Route 53 e créditos de S3, com recursos que variam por plano.
- **Reduzir custo:**
  - aumentar o **cache hit ratio** (cache key enxuta, TTLs maiores, **Origin Shield**);
  - comprimir conteúdo (**gzip/Brotli** automático);
  - escolher a **price class** certa;
  - usar o **CloudFront Security Savings Bundle**, que dá desconto em troca de compromisso de uso e inclui créditos de WAF.

[↑ Índice](#índice)

### CloudFront vs. outras opções

| Necessidade | Solução |
|---|---|
| Conteúdo **HTTP(S)**, estático ou dinâmico, para usuários **globais**, com **cache** | **CloudFront** |
| Tráfego **TCP/UDP** (jogos, VoIP, IoT), **IPs estáticos** e failover regional rápido, **sem cache** | **Global Accelerator**. Ver [Global Accelerator, CloudFront ou Route 53?](compute-and-load-balancing.md#global-accelerator-cloudfront-ou-route-53) |
| Cópias **completas e atualizadas** dos objetos em outras regiões | **S3 Cross-Region Replication** |
| **Uploads** rápidos de usuários distantes **direto para um bucket** | **S3 Transfer Acceleration** (usa as edge locations, mas sem cache) |
| Direcionar usuários para a **região** certa (latência, geolocalização, failover) | **Route 53** |
| Latência de **um dígito de milissegundo** para usuários em uma cidade ou rede 5G | **Local Zones / Wavelength** |
| Site estático com HTTPS e domínio próprio | **S3 + CloudFront + OAC** (+ Route 53 e ACM). Ver [Site estático](storage.md#site-estático-e-requester-pays) |

[↑ Índice](#índice)

### Dicas para a prova

- "Reduzir **latência** para usuários **globais** e **carga na origem**" → **CloudFront**. Para conteúdo **dinâmico**, o CloudFront também ajuda (TLS na borda + rede da AWS), mesmo com `CachingDisabled`.
- "Bucket acessível **somente pelo CloudFront**" → **OAC** + bucket policy com `AWS:SourceArn`. Com SSE-KMS → liberar o CloudFront na **key policy**.
- "ALB/EC2 **sem exposição à internet**, só acessível pelo CloudFront" → **VPC origins**. "ALB público aceitar **só tráfego do CloudFront**" → **header secreto** + regra no ALB/WAF e **prefix list** do CloudFront no security group.
- "**Cache hit ratio** baixo" / "origem sobrecarregada" → enxugar a **cache key** (só os headers, cookies e query strings necessários), aumentar TTLs, **Origin Shield**.
- "Publicar uma nova versão sem esperar o TTL" → **versionar os nomes dos arquivos** (preferível) ou **invalidation**.
- "Alta disponibilidade do conteúdo estático entre **regiões**" → **origin group** (primária + secundária) com **S3 CRR**.
- "Conteúdo pago ou privado só para usuários autorizados" → **signed URLs** (um arquivo) ou **signed cookies** (muitos arquivos), com **trusted key groups**.
- "Restringir por **país** por direitos autorais" → **geo restriction**. "País **e** outras condições" → **WAF geo match**.
- "Proteger dados sensíveis de um formulário **desde a borda** até um serviço específico" → **field-level encryption**.
- "Certificado para domínio próprio no CloudFront" → **ACM em `us-east-1`**. "Web ACL para o CloudFront" → escopo global, **`us-east-1`**.
- "Reescrever URLs, headers, redirects, validar JWT, A/B **com latência mínima e alto volume**" → **CloudFront Functions**. "Chamar outros serviços, acessar o body, usar SDK ou escolher a origem" → **Lambda@Edge**.
- "**Reduzir custo** do CloudFront sem atender o mundo todo" → **price class 100/200**.
- "Testar uma mudança de configuração com parte do tráfego" → **continuous deployment** (staging distribution).
- "Monitorar requisições **em tempo real**" → **real-time logs** para Kinesis Data Streams. "Analisar acessos depois" → **standard logs** no S3 + Athena.
- "Aplicação **TCP/UDP** ou que precisa de **IPs estáticos** e failover em segundos" → **Global Accelerator**, não CloudFront.
- "Mostrar uma página amigável quando a origem falha" → **custom error pages** + **Error Caching Minimum TTL**.

[↑ Índice](#índice)

## Revisão: qual serviço usar

Resumo das comparações que mais aparecem nas questões. Os detalhes estão nas seções de cada serviço.

[↑ Índice](#índice)

### Cache na borda (CloudFront)

| Pergunta do cenário | Solução |
|---|---|
| Conteúdo global com baixa latência e menos carga na origem | [CloudFront](#o-que-é-o-cloudfront) |
| Origem cara ou frágil, muitas requisições chegando nela | [Origin Shield](#arquitetura-edge-locations-regional-edge-caches-e-origin-shield) |
| Só o CloudFront acessa o S3 / o ALB | [OAC / VPC origins / header secreto](#proteger-a-origem) |
| Cache hit ratio baixo / publicar nova versão | [Cache key enxuta / versionamento ou invalidation](#cache-behaviors-cache-key-e-ttl) |
| Conteúdo estático disponível mesmo com a região fora | [Origin group + S3 CRR](#alta-disponibilidade-origin-failover) |
| Conteúdo pago / bloquear países / proteger campos de formulário | [Signed URLs/cookies / geo restriction / field-level encryption](#restringir-o-acesso-ao-conteúdo) |
| Código leve na borda vs. código com rede e SDK | [CloudFront Functions vs. Lambda@Edge](#customização-na-borda-cloudfront-functions-e-lambdaedge) |
| Logs em tempo real / testar mudança com parte do tráfego | [Real-time logs / continuous deployment](#monitoramento-e-implantação) |
| Reduzir custo | [Price classes, cache hit ratio, flat-rate](#custo-do-cloudfront) |
| CloudFront, Global Accelerator, CRR ou Transfer Acceleration? | [Comparação](#cloudfront-vs-outras-opções) |

[↑ Índice](#índice)
