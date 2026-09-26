# Roteiro — Pitch Executivo (15-20 min)

> Do desafio (Página 6/8): *"O Pitch Executivo (15 a 20 min): apresente a
> arquitetura, o PCN e as estratégias financeiras como se estivesse
> vendendo a viabilidade do projeto para a diretoria da ONG."*
>
> Isso é **separado** da "Demo Tech" (que vem depois, no mesmo vídeo de
> até 20min) — aqui o público é a diretoria, não engenheiros. Tom de
> negócio: risco, confiança do doador, custo, continuidade. Zero jargão
> técnico desnecessário — cada número técnico só entra se sustentar um
> argumento de negócio.

**Duração alvo: 17-18 min de fala** (deixa 2-3min de margem dentro do
limite de 20min do vídeo inteiro, já contando a Demo Tech depois). Ritmo
de fala confortável ≈ 130-150 palavras/min.

---

## 0:00 – 1:30 · Abertura (o gancho)

**Objetivo:** a diretoria precisa sentir o problema antes de ver a
solução — senão "infraestrutura" soa como custo, não como proteção da
missão.

> "Antes de falar de tecnologia, quero falar de confiança. A SolidaryTech
> existe pra conectar doadores a causas reais. Toda vez que alguém clica
> em 'doar' e o sistema falha, a gente não perde só uma transação — a
> gente perde a confiança daquele doador na próxima vez. E doador que
> perde confiança não volta.
>
> Hoje eu vou apresentar os três pilares que sustentam essa confiança:
> **arquitetura confiável**, **continuidade de negócio garantida**, e
> **disciplina financeira** — e vou mostrar, com números documentados e
> infraestrutura descrita em código, que essa base já está operando, agora,
> na nuvem."

**[Cue visual opcional: 1 slide com o nome do projeto + os 3 pilares —
Confiabilidade / Continuidade / Custo]**

---

## 1:30 – 5:00 · Arquitetura (visão de negócio, não de engenheiro)

**Objetivo:** mostrar que a arquitetura foi desenhada em torno do que
importa pro negócio — o fluxo de doação — não em torno de tecnologia por
tecnologia.

> "A gente reorganizou a plataforma em três serviços independentes, cada
> um responsável por uma parte da missão: **cadastro de ONGs**,
> **gestão de voluntários**, e o mais crítico de todos — o
> **processamento de doações**. Chamamos esse último de 'Hot Path', o
> Caminho Crítico: é o único ponto do sistema que, se parar, impacta
> receita e reputação diretamente. Por isso ele recebe um nível de
> cuidado diferente dos demais — e vou mostrar exatamente qual nível em
> alguns minutos.
>
> Cada doação, ao ser confirmada, dispara automaticamente um evento de
> notificação — de forma assíncrona, sem travar a experiência do doador —
> que fica registrado para acompanhamento. Isso roda inteiro na Azure, com
> escala automática: em uma campanha de fim de ano, se o volume de doações
> triplicar, o serviço de doações cresce sozinho de 2 até 10 réplicas, o
> cluster de 3 até 5 servidores, e depois volta ao tamanho mínimo
> configurado — sem intervenção manual."

**[Cue visual: diagrama simples de arquitetura — 4 caixas (ONGs,
Doações, Voluntários, Notificações) + 1 seta "doador → doação → evento na
fila → notificação registrada"]**

---

## 5:00 – 9:00 · Confiabilidade — a promessa ao doador (SRE)

**Objetivo:** traduzir SLI/SLO/SLA em **compromisso comercial**, não em
métrica técnica. É aqui que "engenharia de confiabilidade" vira
argumento de venda.

> "Toda promessa de confiabilidade que fazemos pros nossos parceiros —
> ONGs e doadores — hoje é **formal e mensurável**, não uma sensação. Pro
> fluxo de doações, nosso compromisso público é: **99,5% de
> disponibilidade mensal** — isso significa, na prática, menos de 4 horas
> de indisponibilidade por mês, mesmo em cenário de falha.
>
> Internamente, nossa meta de engenharia é ainda mais rígida — 99,9% de
> sucesso nas transações, medido continuamente numa janela de 30 dias.
> Por que a diferença entre as duas? Porque é assim que times de
> confiabilidade sérios trabalham: a meta interna é sempre mais apertada
> que a promessa externa, pra que um problema vire um alerta pra nós
> **antes** de virar uma quebra de contrato pra vocês.
>
> E aqui está o número que eu mais me orgulho de mostrar: no cenário de
> referência que usamos na fase anterior, um incidente levava **10 horas**
> pra ser detectado e resolvido — 6 horas até alguém perceber, mais 4
> horas vasculhando logs manualmente. Na validação dessa automação, que
> replicamos aqui com o mesmo padrão, detecção, abertura de incidente e
> **mitigação automática, sem intervenção humana**, levaram **menos de 2
> minutos**. Isso é uma redução de **99,7%** no tempo de resposta a
> incidentes.
>
> E essa cadeia está configurada no ambiente da SolidaryTech: quando a
> taxa de erro ou a latência do serviço de doações passa do limite, o
> alerta vai automaticamente pro time de plantão e pro canal da equipe, e,
> no caso de erro, o próprio sistema reinicia o serviço antes que uma
> pessoa precise agir."

**[Cue visual: dashboard SRE ao vivo — Disponibilidade Real, Error
Budget, ou o print/gráfico "10h → <2min"]**

---

## 9:00 – 13:00 · Governança financeira (FinOps) — números reais

**Objetivo:** provar que a plataforma é **auditável e otimizada por
padrão**, não um "custo em aberto" — isso é o que tranquiliza uma
diretoria de ONG, que responde por doações de terceiros.

> "Toda ONG que recebe doação tem uma responsabilidade extra com quem
> doa: cada real gasto em operação é um real que não vai pra causa. Por
> isso desenhamos a governança de custo como parte da arquitetura, não
> como uma auditoria feita depois.
>
> Primeiro: todos os recursos que provisionamos por código e que aceitam
> etiquetas carregam as etiquetas obrigatórias — projeto, ambiente e centro
> de custo —, além de uma etiqueta de componente. Isso significa que
> conseguimos abrir o relatório de custo da Azure e separar o gasto por
> componente: o banco do serviço de doações, por exemplo, tem etiqueta
> própria de caminho crítico.
>
> Segundo, o número: com base em preços públicos da Azure consultados em
> setembro, nosso forecast mensal — cluster, bancos de dados, mensageria,
> backup — fica em aproximadamente **US$ 174 por mês**.
>
> Terceiro, e esse é o ponto que eu quero destacar pra diretoria: nós não
> só identificamos uma otimização de custo — ela **já está no código**,
> antes dessa apresentação. O banco NoSQL da plataforma (voluntários e
> notificações) foi configurado no modelo que cobra só pelo uso real, em
> vez de capacidade fixa. Pela nossa estimativa, isso significa **99,6%**
> a menos naquele componente e cerca de **21% a menos no custo total
> mensal** em comparação com a capacidade fixa. Não é uma recomendação de
> PDF: está na infraestrutura como código.
>
> E já temos a próxima otimização mapeada: um compromisso de capacidade
> reservada de 1 ano nos servidores, que reduziria mais **30 a 40%** do
> custo de computação — decidimos não aplicar ainda, de propósito,
> porque reserva de longo prazo só faz sentido depois que confirmamos
> carga de produção estável, não durante a fase de validação. Essa é a
> disciplina financeira que queremos que a diretoria veja: otimizar
> quando faz sentido, não por impulso."

**[Cue visual: tabela de forecast (`02-FINOPS-FORECAST.md`, seção
2) + o comparativo provisionado × serverless do Cosmos DB]**

---

## 13:00 – 16:30 · Continuidade de negócio e segurança (PCN / DR)

**Objetivo:** este é o argumento de **gestão de risco** — a diretoria
precisa sair da sala sabendo que existe um plano formal pra pior
cenário, com números concretos, não só um "a gente se vira".

> "Toda operação séria de doações precisa responder uma pergunta
> desconfortável: e se perdermos o ambiente inteiro — um erro humano, uma
> falha de provedor, um ataque? A resposta pra isso é o nosso Plano de
> Continuidade de Negócio, e ele tem dois números que definem exatamente
> o que prometemos:
>
> Pro fluxo de doações — de novo, nosso ponto mais crítico — o compromisso
> é: no máximo **5 minutos de dado perdido**, e no máximo **30 minutos
> pra voltar a operar**. Pros demais sistemas — cadastro de ONGs e
> voluntários — o compromisso é 15 minutos de dado e até 2 horas de
> recuperação, porque a criticidade é menor.
>
> Como sustentamos esses números? Com **backup automático e agendado**
> rodando de hora em hora especificamente no fluxo de doações, e
> diariamente no resto do ambiente. Isso não é um script que alguém
> promete rodar 'se lembrar' — é infraestrutura como código, versionada,
> auditável, rodando sozinha.
>
> E, também de propósito, decidimos **não** investir agora numa réplica
> completa em segunda região geográfica — isso dobraria nosso custo
> operacional pra reduzir um RTO que já está bem dentro do aceitável pro
> porte atual da operação. É a mesma disciplina financeira de antes
> aplicada à continuidade: proteger o que precisa, sem gastar o dobro por
> segurança que ainda não é necessária. Se o volume de doações crescer a
> ponto de justificar, essa é uma evolução natural, documentada e
> orçada.
>
> Por trás disso tudo também tem segurança embutida na esteira: todo
> código passa por varredura automática de vulnerabilidades antes de
> chegar em produção — nenhuma imagem com vulnerabilidade crítica que já
> tenha correção disponível sobe pro ambiente real."

**[Cue visual: tabela de RTO/RPO (`03-PCN-DISASTER-RECOVERY.md`,
seção 2)]**

---

## 16:30 – 18:00 · Fechamento — o pedido

**Objetivo:** fechar como um pitch de verdade — não "terminei", mas "aqui
está o que peço pra diretoria".

> "Resumindo o que essa arquitetura entrega pra SolidaryTech: uma
> promessa de disponibilidade que a gente consegue honrar e provar; um
> custo mensal previsível, rastreável e já otimizado ativamente; e um
> plano de continuidade com números concretos, não intenções.
>
> O que eu trago pra essa reunião não é uma proposta em aberto — é um
> ambiente **rodando agora**, descrito inteiro em código, com o backup e a
> restauração do serviço de doações já executados na prática.
>
> O que eu peço da diretoria é simples: aprovação pra manter esse padrão
> como requisito não-negociável daqui pra frente — toda nova
> funcionalidade que entrar na plataforma entra dentro dessas mesmas
> garantias de confiabilidade, custo e continuidade. Essa é a base que
> permite a SolidaryTech crescer sem crescer o risco junto.
>
> Agora eu vou passar pra parte técnica e mostrar tudo isso operando ao
> vivo."

**[Transição pra Demo Tech]**

---

## Cheat sheet — números pra ter na ponta da língua

| Tema | Número | Onde vem |
|---|---|---|
| SLA externo | 99,5% disponibilidade mensal | `01-SRE-SLI-SLO-SLA.md` §5 |
| SLO interno | 99,9% sucesso / 30 dias | `01-SRE-SLI-SLO-SLA.md` §4 |
| Error budget | 43,2 min/mês de folga | `01-SRE-SLI-SLO-SLA.md` §4 |
| MTTR (antes → depois) | 10h → <2min (-99,7%), medido na validação da Fase 4 | `01-SRE-SLI-SLO-SLA.md` §6 |
| Forecast mensal total | ~US$ 173,57 | `02-FINOPS-FORECAST.md` §2 |
| Cosmos serverless (no código) × provisionado | -99,6% no componente / -21% no total (~US$46,59/mês, estimativa) | `02-FINOPS-FORECAST.md` §3 |
| Economia recomendada (não aplicada) | -30 a 40% no compute (Reserved Instances) | `02-FINOPS-FORECAST.md` §4 |
| RTO/RPO — donation (Hot Path) | RPO ≤5min / RTO ≤30min | `03-PCN-DISASTER-RECOVERY.md` §2 |
| RTO/RPO — demais serviços | RPO ≤15min / RTO ≤2h | `03-PCN-DISASTER-RECOVERY.md` §2 |
| Backup | horário (donation) / diário (todos) | `03-PCN-DISASTER-RECOVERY.md` §3.2 |
| Tagging | 9 recursos Azure com tags obrigatórias (todos os do Terraform que aceitam `tags`) | `02-FINOPS-FORECAST.md` §1 |

## Dicas de entrega

- **Fale pra diretoria, não pra plateia técnica** — se uma frase só faz
  sentido pra quem lê YAML, corta ou traduz.
- Os **porquês de "não fizemos X agora"** (reserva de capacidade,
  multi-região) são, na real, os momentos mais fortes do pitch — mostram
  julgamento financeiro, não só capacidade técnica. Não pule eles.
- Se o tempo apertar, o bloco que **pode encolher** é a Arquitetura
  (1:30-5:00) — a diretoria confia mais em resultado do que em diagrama.
  Os blocos de SRE, FinOps e PCN são os que respondem exatamente ao que
  o enunciado pede — não corta esses.
