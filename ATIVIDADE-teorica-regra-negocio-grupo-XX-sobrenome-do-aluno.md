# Atividade Teórica: Regra de Negócio no BD versus na Aplicação

**Aluno(s):** João Paulo Dias; João Victor; Maykon; Saymon; Pedro Henrique

**Turma:** Banco de Dados 2026

**Grupo:** 9

**Data:** 19 de agosto de 2026

**Repositório Git:** <https://github.com/PedroBirudevs/bd2026-regra-negocio-grupo-9>

---

## Resumo Executivo

**O problema.** Todo sistema de informação precisa garantir que certas afirmações sobre o negócio sejam sempre verdadeiras: um pedido não pode consumir estoque que não existe, um cliente não pode ser cadastrado com CPF já usado por outro, um cliente não pode manter dois pedidos em aberto ao mesmo tempo. A questão prática é decidir **onde** essas garantias são implementadas — no banco de dados, no código da aplicação, ou nos dois lugares. A decisão não é apenas estética: ela determina o que acontece quando duas transações concorrem pelo mesmo produto, quando um script de importação escreve direto na base, ou quando o aplicativo mobile é atualizado meses depois do aplicativo web.

**A diferença entre as duas camadas.** O banco de dados é o **guardião final do estado persistido**. Tudo o que for gravado passa obrigatoriamente por ele, venha de onde vier: do aplicativo web, do app mobile, de uma API pública, de um relatório administrativo ou de um `psql` aberto por um administrador às duas da manhã. As restrições declaradas no esquema (`PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `CHECK`) são verificadas pelo próprio SGBD, dentro do controle transacional, e por isso são as únicas garantias que **nenhum caminho de escrita consegue contornar**. A aplicação, por outro lado, é onde o negócio é **interpretado**: é ela que sabe qual desconto vale nesta campanha, em que ordem as etapas do pedido acontecem, que mensagem mostrar ao usuário e como o fluxo muda quando o marketing altera a política comercial na semana seguinte. A aplicação é expressiva, testável e barata de mudar; o banco é restritivo, compartilhado e caro de mudar — e é exatamente por isso que cada um serve para um tipo diferente de regra.

**Conclusão principal.** Não existe vencedor absoluto, e as duas posições extremas ("tudo no banco" e "tudo na aplicação") falham por motivos simétricos. Este trabalho defende uma **arquitetura híbrida com critério explícito de alocação**: o banco de dados deve ser o dono das **invariantes de dados** — propriedades que precisam ser verdadeiras para *qualquer* linha gravada, independentemente de quem gravou (identidade, unicidade, integridade referencial, obrigatoriedade, domínio de valores e as invariantes sensíveis a concorrência, como estoque não negativo); a aplicação deve ser a dona das **regras de processo e de política** — fluxos, cálculos, decisões comerciais, elegibilidade, mensagens e tudo aquilo que muda por decisão de negócio e não por mudança na natureza do dado. Regras críticas ficam nas duas camadas, mas com papéis diferentes: na aplicação para dar boa experiência ao usuário e mensagens claras; no banco como **última linha de defesa**, na forma mais declarativa possível. A justificativa técnica dessa posição está desenvolvida nas seções 1.5 e 4, e o critério prático de decisão está resumido na seção 2.3.

---

## 1. Desenvolvimento Teórico

### 1.1 O que é regra de negócio?

#### Definição

A definição clássica vem do relatório do *GUIDE Business Rules Project*, publicado pelo Business Rules Group: uma regra de negócio é uma declaração que define ou restringe algum aspecto do negócio, com a intenção de afirmar a estrutura do negócio ou de controlar e influenciar o seu comportamento (BUSINESS RULES GROUP, 2000, p. 4). Em linguagem mais direta: **regra de negócio é uma decisão da organização sobre o que pode, o que não pode e como as coisas devem acontecer.** Ela existe antes do software e continuaria existindo se a empresa operasse em papel.

Duas consequências importantes decorrem dessa definição:

1. A regra pertence ao **negócio**, não à tecnologia. "Cliente precisa ter 18 anos" é uma decisão da empresa (apoiada, no Brasil, na maioridade civil do art. 5º do Código Civil), não uma característica do PostgreSQL nem do Java.
2. O software é apenas o lugar onde a regra é **implementada**. A mesma regra pode ser implementada em um `CHECK`, em um `if` da camada de serviço, em uma validação de formulário — ou, na pior das hipóteses, em nenhum lugar.

#### Para que serve

Regras de negócio servem para manter o sistema **coerente com a realidade que ele representa**. Um banco de dados de vendas cujo estoque fica negativo, ou que registra dois clientes distintos com o mesmo CPF, não está apenas "com um bug": ele está descrevendo um mundo que não existe. A partir daí, todo relatório, toda apuração fiscal e toda decisão gerencial baseada naqueles dados fica comprometida. Regras de negócio também servem para **tornar a decisão explícita e auditável**: uma regra escrita no esquema ou em uma classe de domínio pode ser lida, revisada e testada; uma regra que só existe na cabeça de um analista, não.

#### Exemplos em um sistema de vendas

| Regra | Natureza |
|---|---|
| Não é permitido vender quantidade maior do que o estoque disponível | Restrição sobre estado + concorrência |
| Cliente precisa ter no mínimo 18 anos para se cadastrar | Restrição de elegibilidade, com base temporal |
| Um cliente não pode ter dois pedidos em aberto simultaneamente | Restrição sobre o conjunto de linhas de um mesmo cliente |
| O CPF deve ser único por cliente | Identificação / unicidade |
| Pedido acima de R$ 500,00 tem frete grátis na região Norte | Política comercial |
| Pedido só pode ser faturado se tiver pelo menos um item | Restrição sobre o fluxo/estado do processo |
| Todo item de pedido precisa referenciar um produto existente | Integridade referencial |

#### Regra de negócio × integridade de dados × validação de entrada

Esses três conceitos são frequentemente tratados como sinônimos, e essa confusão é a origem da maior parte das discussões improdutivas sobre "onde colocar a regra". Eles são distintos:

| Conceito | O que é | Onde falha se estiver ausente | Exemplo |
|---|---|---|---|
| **Regra de negócio** | Decisão da organização sobre o que é permitido ou como o processo ocorre. É um conceito de domínio, independente de tecnologia. | O sistema permite operações que a empresa não autoriza. | "Cliente inadimplente não pode abrir novo pedido." |
| **Regra de integridade de dados** | Propriedade estrutural que o conjunto de dados precisa satisfazer para ser um modelo consistente da realidade. É formalizada no modelo relacional (integridade de entidade, integridade referencial, restrições de domínio). | Os **dados** ficam corrompidos: linhas órfãs, duplicatas, valores impossíveis. | "Todo `item_pedido.produto_id` deve existir em `produto`." |
| **Validação de entrada** | Verificação sintática e de formato do que chega pela interface, antes de qualquer processamento. | O usuário recebe erro tardio, feio ou nenhum; e dados malformados entram no fluxo. | "O campo CPF deve conter 11 dígitos numéricos." |

A distinção fica mais clara com um único atributo. Para o CPF de um cliente:

- **Validação de entrada:** o campo veio preenchido, tem 11 dígitos, não contém letras.
- **Regra de negócio:** os dígitos verificadores conferem segundo o algoritmo da Receita Federal, e o CPF não pertence a um cliente bloqueado.
- **Integridade de dados:** não existem duas linhas em `cliente` com o mesmo CPF.

São três coisas diferentes, com custos e consequências diferentes, e **não pertencem necessariamente à mesma camada**. Toda regra de integridade de dados nasce de alguma regra de negócio, mas nem toda regra de negócio é uma regra de integridade — e é justamente essa assimetria que torna errado afirmar que "toda regra de negócio deve estar no banco".

#### Regras que precisam de proteção no banco de dados

Existe uma classe de regras cuja violação **corrompe os dados de forma que nenhuma correção posterior recupera com segurança**. Essas devem estar declaradas no esquema:

- **Integridade referencial.** Um item de pedido que aponta para um produto inexistente não pode ser interpretado, nem corrigido automaticamente: a informação de qual produto era foi perdida. `FOREIGN KEY` impede a criação da linha órfã e define o que acontece na exclusão do lado referenciado (`ON DELETE RESTRICT`, `CASCADE`, `SET NULL`).
- **Unicidade.** Dois clientes com o mesmo CPF significam que a chave que o negócio usa para identificar pessoas deixou de identificar. Depois que a duplicata existe e ganhou pedidos, a fusão dos registros é um trabalho manual e arriscado. `UNIQUE` (ou índice único parcial, para unicidade condicional) resolve o problema no ponto de gravação.
- **Obrigatoriedade.** Um pedido sem cliente ou um item sem quantidade não é um registro incompleto: é um registro sem sentido. `NOT NULL` garante presença.
- **Domínios e restrições de valores.** Preço negativo, quantidade zero, estoque negativo, status fora do conjunto previsto. Tipos adequados (`numeric(12,2)` em vez de `float` para dinheiro), tipos enumerados e `CHECK` mantêm cada coluna dentro do universo de valores que faz sentido.
- **Identidade.** `PRIMARY KEY` garante que cada linha é distinguível e endereçável — condição para que qualquer outra referência funcione.

O critério comum a todas: são propriedades verificáveis olhando apenas para os dados, que precisam valer para **qualquer** gravação, e cuja violação produz um estado do qual não se volta.

#### Regras mais adequadas à camada de aplicação

- **Fluxos de negócio.** A ordem das etapas (validar cliente → reservar estoque → calcular total → registrar pagamento → emitir nota) é orquestração. Expressá-la em SQL procedural é possível, mas o resultado é mais difícil de ler, testar e evoluir do que o equivalente em uma linguagem de aplicação.
- **Cálculo de preços.** Preço final envolve tabela de preços, campanha vigente, cupom, faixa de frete, regime tributário. É lógica com muitas dependências externas e alta taxa de mudança.
- **Políticas comerciais.** "Frete grátis acima de R$ 500 na região Norte até 31/08" é uma decisão que dura semanas. Colocá-la no esquema significa fazer *deploy* de banco a cada campanha.
- **Regras complexas.** Regras que dependem de serviços externos (consulta a birô de crédito, antifraude, cálculo de imposto por API) simplesmente não pertencem ao banco: chamadas externas dentro de transações prendem locks e acoplam a disponibilidade do banco à de terceiros.
- **Regras que mudam com frequência.** Quanto maior a taxa de mudança, mais o custo de alteração pesa — e alterar código de aplicação é significativamente mais barato e mais reversível do que alterar um esquema compartilhado em produção.

#### Por que a divisão não é absoluta

A divisão acima é um ponto de partida, não uma fronteira rígida. Três situações exigem presença em mais de uma camada:

1. **Experiência do usuário versus garantia.** A regra "estoque suficiente" precisa ser verificada na aplicação para que o usuário veja "restam 2 unidades" antes de finalizar a compra; e precisa ser garantida no banco para que duas compras simultâneas não zerem um estoque de uma unidade. São a mesma regra em dois papéis: um informativo, outro de garantia.
2. **Regras híbridas.** "CPF válido" tem uma parte estrutural (11 dígitos, único) que cabe no banco e uma parte algorítmica (dígitos verificadores) que cabe na aplicação.
3. **Erosão ao longo do tempo.** Uma regra que hoje existe só na aplicação porque só há uma aplicação sobrevive à criação da segunda aplicação — mas não intacta. A proteção no banco é o que garante que a regra continue valendo quando o sistema crescer de formas que ninguém previu.

O que **não** é aceitável é o inverso: supor que uma validação na aplicação substitui a integridade do banco. Ela não substitui, porque não é executada quando a escrita não passa pela aplicação.

---

### 1.2 Regras no banco de dados

#### Restrições declarativas

Restrições (*constraints*) são regras declaradas no esquema e verificadas pelo SGBD a cada gravação. O PostgreSQL implementa restrições de verificação, de não-nulidade, de chave primária, de unicidade, de chave estrangeira e de exclusão (POSTGRESQL, 2026a).

**`NOT NULL`** — impede que a coluna receba o valor nulo. É a restrição mais simples e a mais barata: nenhuma estrutura auxiliar, nenhuma consulta adicional.

```sql
nome varchar(120) NOT NULL
```

**`CHECK`** — impede a gravação de linhas em que uma expressão booleana não seja satisfeita.

```sql
CONSTRAINT ck_produto_estoque CHECK (estoque >= 0)
```

Três limitações precisam ser compreendidas para não escrever `CHECK` incorreto:

1. A expressão só enxerga **a linha corrente**. O PostgreSQL não suporta `CHECK` que referencie dados de outras linhas ou de outras tabelas; a documentação alerta explicitamente que uma restrição que viole essa regra pode até parecer funcionar em testes simples, mas não garante que o banco não alcance um estado em que a condição é falsa, e pode inviabilizar um `dump`/`restore` (POSTGRESQL, 2026a). Para restrições entre linhas ou entre tabelas, o caminho correto é `UNIQUE`, `EXCLUDE`, `FOREIGN KEY` ou, quando nenhum desses serve, um *trigger*.
2. `CHECK` é verificado **apenas quando a linha é inserida ou atualizada**, nunca depois. Uma expressão que dependa do tempo (`CURRENT_DATE`) é aceita pelo PostgreSQL, mas expressa "isto era verdade no momento da gravação" — não uma invariante permanente. Se a expressão puder se tornar falsa com a passagem do tempo, uma restauração de *backup* pode falhar.
3. `CHECK` e `NOT NULL` **não podem ser adiadas** para o fim da transação: são sempre verificadas imediatamente ao inserir ou modificar a linha. Apenas `UNIQUE`, `PRIMARY KEY`, `REFERENCES` e `EXCLUDE` podem ser declaradas `DEFERRABLE` e controladas por `SET CONSTRAINTS` (POSTGRESQL, 2026b). Isso foi verificado na prática: `ALTER TABLE ... ADD CONSTRAINT ... CHECK (...) DEFERRABLE` retorna `ERROR: CHECK constraints cannot be marked DEFERRABLE`.

**`UNIQUE`** — garante que não existam duas linhas com o mesmo valor (ou mesma combinação de valores) na coluna restringida. É implementada por um índice único, o que significa que a restrição tem custo de manutenção em escrita e benefício colateral em leitura. Detalhe frequentemente ignorado: por padrão, valores nulos são considerados **distintos entre si**, de modo que uma coluna `UNIQUE` aceita várias linhas com `NULL` — comportamento confirmado em teste. A partir da versão 15 é possível inverter isso com `UNIQUE NULLS NOT DISTINCT`.

**`PRIMARY KEY`** — é a combinação de `UNIQUE` com `NOT NULL`, mais a marcação semântica de que aquela coluna (ou conjunto) identifica a linha. Serve de alvo padrão para chaves estrangeiras que referenciem a tabela.

**`FOREIGN KEY`** — exige que o valor referenciado exista na tabela referenciada. A coluna referenciada precisa ser chave primária, restrição de unicidade ou coluna de índice único não parcial — o que garante que a verificação do lado referenciado seja indexada. O lado **referenciante**, porém, **não recebe índice automático**, e a documentação recomenda criá-lo quando houver `DELETE`/`UPDATE` frequente do lado referenciado, porque essas operações precisam varrer a tabela filha (POSTGRESQL, 2026a). Esquecer esse índice é uma causa comum de lentidão atribuída injustamente "às FKs".

#### Triggers

*Trigger* é um procedimento associado a uma tabela ou visão, disparado automaticamente pelo SGBD quando ocorre `INSERT`, `UPDATE`, `DELETE` ou `TRUNCATE`, podendo executar antes (`BEFORE`), depois (`AFTER`) ou no lugar (`INSTEAD OF`) da operação, por linha ou por comando (POSTGRESQL, 2026c). No PostgreSQL, a função de *trigger* é escrita em uma linguagem procedural — tipicamente PL/pgSQL — declarada sem argumentos e com retorno do tipo `trigger` (POSTGRESQL, 2026d).

*Triggers* são a ferramenta certa quando a regra é uma invariante de dados **que as restrições declarativas não conseguem expressar** — tipicamente porque envolve outras linhas ou outras tabelas. São a ferramenta errada quando usadas para colocar fluxo de negócio dentro do banco: efeitos colaterais invisíveis para quem lê o `INSERT`, ordem de disparo difícil de raciocinar, e depuração custosa. A regra prática adotada neste trabalho é: **se a restrição pode ser declarativa, ela não deve ser um *trigger***.

#### Funções e procedimentos armazenados

O PostgreSQL distingue duas construções que costumam ser tratadas como uma só:

- **Função** (`CREATE FUNCTION`) — retorna um valor, é chamada dentro de uma expressão SQL e executa **dentro** da transação de quem a chamou; não pode confirmar nem desfazer transações.
- **Procedimento** (`CREATE PROCEDURE`, disponível desde a versão 11) — não retorna valor pela chamada, é invocado com `CALL` e, dependendo do contexto, pode controlar transações internamente.

Ambos são "código no banco" e concentram lógica no servidor. Vantagem: reduzem tráfego de rede e idas e vindas em operações que fazem muitas manipulações de dados. Desvantagem: são escritos em uma linguagem específica do SGBD, dificultam versionamento e teste automatizado se não houver disciplina de migrações, e deslocam para o banco responsabilidades que a equipe de aplicação normalmente mantém.

#### Transações e propriedades ACID

Uma **transação** é uma unidade lógica de trabalho: um conjunto de comandos que o SGBD trata como uma única operação indivisível, delimitada por `BEGIN` e encerrada por `COMMIT` ou `ROLLBACK`. Não é sinônimo de restrição nem de *trigger*: restrições dizem *o que é válido*; a transação diz *qual conjunto de mudanças vale tudo ou nada*.

O acrônimo **ACID** foi consolidado por Härder e Reuter (1983) e descreve quatro propriedades:

- **Atomicidade (A).** Ou todos os efeitos da transação são aplicados, ou nenhum. Se a transação abrir um pedido, baixar estoque e falhar antes do `COMMIT`, o pedido não fica gravado. Isso foi verificado em teste: dentro de uma transação, o `INSERT` do pedido seguido de `ROLLBACK` deixou zero linhas na tabela.
- **Consistência (C).** Uma transação leva o banco de um estado consistente a outro estado consistente. Aqui é preciso precisão conceitual: o SGBD **garante automaticamente** as restrições declaradas (chaves, `CHECK`, `NOT NULL`, integridade referencial); as regras de consistência **não declaradas** — como "o total do pedido deve corresponder à soma dos itens" — continuam sendo responsabilidade de quem escreveu a transação. Dizer que "o banco garante a consistência" sem essa ressalva é o erro conceitual mais comum sobre ACID.
- **Isolamento (I).** Transações concorrentes não enxergam estados intermediários umas das outras conforme o nível de isolamento escolhido. O PostgreSQL usa MVCC e oferece três níveis distintos na prática — *Read Committed* (padrão), *Repeatable Read* e *Serializable*; *Read Uncommitted* é aceito, mas se comporta como *Read Committed* (POSTGRESQL, 2026e). Em *Read Committed*, cada comando enxerga um instantâneo dos dados confirmados até o início daquele comando; em *Repeatable Read*, o instantâneo é do início da transação; em *Serializable*, o PostgreSQL usa *Serializable Snapshot Isolation* (PORTS; GRITTNER, 2012) e pode abortar transações com erro de serialização, exigindo que a aplicação esteja preparada para repeti-las.
- **Durabilidade (D).** Depois do `COMMIT`, os efeitos sobrevivem a falhas de energia ou queda do servidor, garantidos pelo *write-ahead log*.

O ponto arquitetural que decorre daí: **a transação é o único mecanismo que permite compor várias regras em uma garantia única**. Uma aplicação pode verificar estoque, verificar pedido em aberto e calcular total corretamente e, ainda assim, produzir estado inconsistente se essas etapas não estiverem dentro da mesma transação.

#### Duas vantagens da implementação no banco

**1. Consistência centralizada e independente do caminho de escrita.** A restrição declarada no esquema vale para qualquer origem: aplicação web, app mobile, API, rotina de importação, correção manual via `psql`. Não existe caminho que a contorne — nem por esquecimento, nem por pressa, nem por má-fé. Nenhuma outra camada oferece essa propriedade.

**2. Correção sob concorrência.** Regras que dependem do estado corrente e são disputadas por várias sessões — estoque é o exemplo canônico — só podem ser garantidas onde existe controle de concorrência. O banco tem MVCC, bloqueio de linha e níveis de isolamento; a aplicação, sozinha, não tem nada equivalente. Uma verificação "consultar estoque, decidir, atualizar" feita na aplicação é uma condição de corrida clássica: entre a consulta e a atualização, outra sessão pode ter consumido o mesmo estoque.

#### Duas desvantagens da implementação no banco

**1. Risco de concentrar lógica demais no servidor.** Quando fluxo de negócio migra para *triggers* e procedimentos, o comportamento do sistema deixa de ser legível a partir do código da aplicação. Um `INSERT` aparentemente trivial dispara uma cadeia de efeitos que só aparece inspecionando o catálogo. Isso dificulta a análise de impacto, transforma qualquer mudança em operação de risco e concentra carga em um recurso que escala verticalmente com custo alto, enquanto servidores de aplicação escalam horizontalmente.

**2. Portabilidade, manutenção e testes.** SQL declarativo é razoavelmente portável — `CHECK`, `UNIQUE` e `FOREIGN KEY` fazem parte do padrão ISO/IEC 9075. Já PL/pgSQL não é: migrar para outro SGBD significa reescrever. Além disso, testar lógica no banco exige uma instância real com dados preparados, ciclo naturalmente mais lento do que testes unitários em memória; e mudanças de esquema em produção precisam de *scripts* de migração versionados e, muitas vezes, de janela de manutenção, enquanto uma mudança de regra na aplicação é resolvida em um *deploy* comum e reversível.

---

### 1.3 Regras na aplicação

#### Validação de entrada

É a primeira barreira, na fronteira com o usuário ou com o cliente da API: campos obrigatórios, formato, tipo, tamanho, faixa. Seu objetivo é **rejeitar cedo e explicar bem**. Uma validação de entrada não deve ser confundida com garantia: ela roda no cliente ou na borda do servidor e existe para dar retorno rápido, não para proteger o dado.

#### Camada de serviço

É onde as operações do sistema são orquestradas. No catálogo de Fowler (2002), a *Service Layer* define a fronteira da aplicação e o conjunto de operações disponíveis, coordenando a resposta de cada operação — inclusive o controle transacional. É nela que "criar pedido" existe como conceito: validar, reservar estoque, calcular, persistir, tudo dentro de uma transação.

#### Regras de domínio

São as regras que descrevem o próprio negócio, modeladas em objetos que representam conceitos do domínio (Pedido, Cliente, Produto). Evans (2003) argumenta que colocar essas regras dentro dos objetos de domínio, em vez de espalhá-las por procedimentos, é o que mantém o modelo compreensível à medida que o sistema cresce. O contraponto é o *anemic domain model*, em que os objetos guardam apenas dados e toda a regra vaza para classes de serviço — situação que, na prática, produz duplicação e inconsistência.

#### Arquitetura em camadas e separação de responsabilidades

A organização usual separa apresentação, aplicação/serviço, domínio e infraestrutura (incluindo persistência). O princípio é que cada camada tenha uma razão de mudar: a apresentação muda quando muda a interface, o domínio quando mudam as regras, a persistência quando muda o armazenamento. A localização da regra de negócio interage diretamente com esse princípio — regra de negócio dentro do banco significa que uma mudança de negócio exige mudança na camada de infraestrutura, o que atravessa a separação. Esse é um argumento legítimo a favor da aplicação, mas ele **não vale para invariantes de dados**, porque estas não são "detalhe de infraestrutura": são propriedades do dado em si.

#### Duas vantagens da implementação na aplicação

**1. Manutenção, evolução e testabilidade.** Regras em código de aplicação são versionadas junto com o resto do sistema, revisadas em *pull request*, cobertas por testes unitários rápidos e sem banco, e liberadas em um *deploy* comum com reversão simples. Regras complexas podem ser decompostas em funções pequenas, nomeadas com o vocabulário do negócio, e lidas por qualquer pessoa da equipe — inclusive quem não domina SQL.

**2. Experiência do usuário.** A aplicação pode validar antes de enviar, mostrar quantas unidades restam, indicar exatamente qual campo está errado, sugerir correção e evitar que o usuário preencha um formulário inteiro para receber uma mensagem genérica no final. Uma violação de restrição do banco produz uma mensagem correta, mas inadequada para o usuário final: `duplicate key value violates unique constraint "uq_cliente_cpf"` é informação de diagnóstico, não de interface.

#### Duas desvantagens da implementação na aplicação

**1. Inconsistência quando existem vários clientes ou aplicações.** A regra implementada na aplicação vale exatamente para as escritas que passam por aquela aplicação. Basta uma segunda aplicação, um *job* de importação, um microsserviço novo ou um `UPDATE` manual em produção para que a regra deixe de valer — sem nenhum erro visível no momento, e com dados inconsistentes descobertos meses depois. Fowler (2004) trata precisamente dessa distinção entre banco de aplicação (acessado por um único sistema) e banco de integração (compartilhado por vários); a segunda situação, muito comum em sistemas corporativos, torna a proteção exclusivamente na aplicação insuficiente por construção.

**2. Ausência de controle de concorrência.** Como discutido em 1.2, verificar-e-depois-gravar na aplicação não é atômico. Duas requisições simultâneas podem ambas ler estoque igual a 1, ambas concluir que a venda é possível e ambas gravar. O resultado é estoque negativo ou venda sem lastro — e o problema aparece justamente sob carga, quando é mais caro.

Há ainda um risco relacionado, de natureza organizacional: **uma aplicação pode simplesmente esquecer a regra, ou implementá-la de forma sutilmente diferente**. Duas equipes que interpretam "pedido em aberto" de maneiras distintas (uma considerando `ABERTO` e `AGUARDANDO_PAGAMENTO`, outra apenas `ABERTO`) produzem, sem nenhuma falha de código, um banco que viola a regra de negócio original.

---

### 1.4 Comparativo BD × Aplicação

| Critério | Banco de Dados | Aplicação |
|---|---|---|
| **Consistência** | Verificação no ponto único de gravação, dentro do controle transacional. Restrições declaradas valem para toda escrita, de qualquer origem, inclusive comandos manuais. É a única camada capaz de oferecer garantia absoluta sobre o estado persistido. | Verificação no caminho percorrido por aquela aplicação. Consistente enquanto todas as escritas passarem por ela e ela implementar a regra corretamente — duas condições que dependem de disciplina, não de mecanismo. |
| **Segurança** | Aplica-se independentemente de quem se conecta; combinada com `GRANT`/`REVOKE` e *row-level security*, permite que a regra não dependa da boa-fé do cliente. Resistente a bypass da camada de aplicação. | Pode ser contornada por qualquer acesso que não passe pela aplicação; validação executada no cliente é, do ponto de vista de segurança, apenas uma sugestão. Por outro lado, é onde ficam autorização de negócio, auditoria funcional e integração com identidade. |
| **Performance** | Restrições declarativas têm custo baixo e previsível; `UNIQUE`/`PK` mantêm índice (custo em escrita, ganho em leitura); `FOREIGN KEY` pode custar caro sem índice no lado referenciante. Validar perto do dado evita idas e voltas de rede. Concentrar lógica pesada, porém, sobrecarrega um recurso que escala verticalmente. | Reduz carga do servidor de banco e escala horizontalmente; evita ida ao banco para rejeitar entradas obviamente inválidas. Em compensação, regras que exigem consultar o estado atual geram *round-trips* adicionais, e verificações otimistas exigem repetição sob concorrência. |
| **Manutenção** | Mudança de regra é mudança de esquema: exige migração versionada, coordenação com todas as aplicações que usam a base e, às vezes, janela de manutenção. Reversão é mais delicada. Em compensação, a regra fica em um lugar só. | Mudança é *commit* + *deploy*, com reversão simples e histórico de revisão. O risco é a duplicação: a mesma regra reescrita em várias aplicações precisa ser mantida em sincronia manualmente. |
| **Portabilidade** | Restrições declarativas seguem o padrão SQL e migram razoavelmente bem entre SGBDs. Código procedural (PL/pgSQL) e recursos específicos (tipos `ENUM`, índices parciais, `EXCLUDE`) não são portáveis sem reescrita. | Independente de fornecedor de banco; trocar o SGBD não exige reescrever regra de negócio. A dependência se desloca para a linguagem e o *framework* — geralmente uma dependência mais confortável de administrar. |
| **Testabilidade** | Exige instância real, dados de apoio e isolamento entre testes; ciclo mais lento. Em contrapartida, o teste de uma restrição declarativa é trivial e definitivo: tenta violar, espera erro. Testar *triggers* e procedimentos complexos é significativamente mais trabalhoso. | Testes unitários rápidos, sem infraestrutura, com casos de borda e simulação de dependências. Alta cobertura barata. Ressalva importante: testes de aplicação **não** provam ausência de anomalias de concorrência, que só aparecem em teste de integração com transações reais. |
| **Controle central da regra** | Máximo. Uma definição, um lugar, verificação obrigatória. O esquema funciona como contrato explícito e legível para todos os consumidores da base. | Central apenas se houver uma única aplicação, ou se todas consumirem uma mesma biblioteca/serviço de domínio. Em arquitetura distribuída, exige governança deliberada para não fragmentar. |
| **Múltiplas aplicações** | Cenário em que a vantagem do banco é maior: a regra vale igualmente para web, mobile, API, ETL e acesso administrativo, sem coordenação entre equipes. | Cenário mais frágil: cada nova aplicação precisa reimplementar a regra, e divergências não geram erro — geram dados inconsistentes silenciosos. Mitigável concentrando a escrita atrás de um único serviço, o que é uma decisão arquitetural com seus próprios custos. |
| **Regras que mudam com frequência** | Pouco adequado. Cada alteração vira migração, com impacto potencial em todas as aplicações e, em tabelas grandes, com custo de validação sobre os dados existentes. Restrições rígidas sobre políticas voláteis tendem a ser removidas na primeira urgência. | Cenário natural da aplicação: alteração barata, reversível e testável, com possibilidade de parametrizar a regra em dados (tabela de configuração) em vez de codificá-la na estrutura. |

---

### 1.5 Análise crítica: qual a melhor opção?

**Existe uma opção vencedora absoluta? Não.** E o motivo não é diplomacia: é que as duas camadas oferecem garantias de naturezas diferentes, e a pergunta "onde a regra deve viver" só admite resposta depois de outra pergunta — *o que acontece se esta regra específica for violada?* Se a resposta for "os dados ficam permanentemente corrompidos", a regra precisa de proteção no banco. Se for "o usuário tem uma experiência ruim" ou "a empresa aplica uma política errada por um período", o lugar natural é a aplicação.

A seguir, os quatro cenários pedidos.

#### Cenário A — Sistema acessado por múltiplas aplicações

Considere o mesmo banco de vendas acessado por: (i) aplicativo web, (ii) aplicativo mobile, (iii) API pública consumida por parceiros, (iv) sistema administrativo interno — e, na prática, também por (v) rotinas de importação e correções manuais.

Se a regra "um cliente não pode ter dois pedidos em aberto" existir apenas na camada de serviço do sistema web, ocorre o seguinte:

- o app mobile, desenvolvido por outra equipe seis meses depois, implementa a mesma regra — mas considerando também o status `AGUARDANDO_PAGAMENTO` como "aberto", porque o produto mudou nesse intervalo;
- a API do parceiro chama diretamente a camada de persistência, porque a regra "não fazia sentido" para o fluxo de integração;
- o sistema administrativo reabre pedidos cancelados sem passar pela validação, porque é uma operação excepcional;
- a rotina de importação noturna insere pedidos em lote com `COPY`.

Nenhuma dessas quatro decisões é irracional isoladamente. Juntas, produzem um banco em que a regra de negócio simplesmente não vale — e ninguém percebe até que a apuração mensal aponte clientes com múltiplos pedidos abertos e valores duplicados. Note que **o problema não é detectado por testes**, porque cada aplicação passa nos seus próprios testes.

É o cenário que Fowler (2004) descreve como *integration database*: quando o banco é compartilhado por vários sistemas, o esquema é o único contrato comum a todos. Por isso, aqui, regras de unicidade, de integridade referencial e de invariante de estado precisam estar declaradas no banco. No exemplo da seção 2.1, isso é feito com um **índice único parcial**, que expressa "no máximo um pedido `ABERTO` por cliente" de forma declarativa — sem *trigger*, sem código, e válido para todos os cinco caminhos de escrita.

Uma alternativa arquitetural legítima é impedir o acesso direto: publicar um único serviço de escrita e proibir que qualquer outro sistema fale com o banco. Essa opção transforma o banco de integração em banco de aplicação e devolve à aplicação o controle central. Ela funciona, mas tem custos reais (disponibilidade do serviço, latência, governança) e, sobretudo, **não elimina o acesso administrativo direto**, que existe em praticamente toda operação. Na prática, ela reduz a probabilidade de divergência, mas não substitui a garantia estrutural.

#### Cenário B — Dados sensíveis ou exigência legal/fiscal

Quando os dados têm consequência legal — documentos fiscais, identificação de pessoas, valores tributados, registros de consentimento — a exigência muda de natureza: não basta que o sistema *normalmente* se comporte bem; é preciso poder **demonstrar** que determinado estado inválido não pode ter ocorrido.

Duas razões técnicas sustentam a proteção no banco nesse cenário:

1. **Garantia independente do código.** Se a única proteção contra CPFs duplicados for uma verificação na camada de serviço, a afirmação "não existem clientes duplicados" depende de auditar todo o histórico de versões de todas as aplicações que já escreveram naquela base. Com uma restrição `UNIQUE`, a afirmação depende apenas do esquema — que é verificável em um comando. Em auditoria, essa diferença é decisiva.
2. **Integridade de trilha e imutabilidade.** Documentos fiscais emitidos não podem ser alterados retroativamente. Chaves estrangeiras com `ON DELETE RESTRICT` impedem que um cliente com pedidos faturados seja apagado; restrições e *triggers* de imutabilidade impedem que itens de um pedido já faturado sejam modificados (exemplo implementado em 2.1). São garantias que não dependem de nenhuma aplicação se comportar corretamente.

Vale registrar o limite disso: o banco garante *estrutura*, não *conformidade legal*. A LGPD (Lei nº 13.709/2018) exige minimização, finalidade e base legal para tratamento — obrigações que se implementam com modelagem, políticas de acesso, retenção e processo, não com `CHECK`. Confundir integridade referencial com conformidade regulatória seria um erro conceitual na direção oposta.

#### Cenário C — Regras que mudam com frequência

Suponha a regra "clientes da categoria Ouro têm frete grátis acima de R$ 300; nas demais categorias, acima de R$ 500; na região Norte, acréscimo de R$ 20 no frete padrão", revisada a cada campanha.

Implementar isso no banco significa: alterar `CHECK` ou função a cada revisão; migração em produção a cada campanha comercial; risco de indisponibilidade proporcional ao tamanho das tabelas; e coordenação com todas as aplicações a cada mudança. Pior: uma regra volátil declarada de forma rígida **invalida dados históricos** — pedidos gravados sob a política anterior podem violar a restrição nova, fazendo com que uma restauração de *backup* ou uma revalidação falhem.

Aqui, a aplicação é claramente o lugar certo. Melhor ainda: parte dessas regras deve virar **dado** em vez de código — uma tabela `politica_frete` com vigência, consultada pela camada de serviço. Assim o banco continua garantindo a estrutura (a política existe, tem vigência válida, valores não negativos) sem embutir a decisão comercial no esquema.

O cuidado necessário é não estender essa conclusão às invariantes: "estoque não pode ficar negativo" **não** é uma regra que muda com frequência, mesmo que a política de reserva de estoque mude. A distinção entre a invariante e a política que a acompanha é o que evita que o argumento da volatilidade seja usado para retirar do banco proteções que deveriam estar lá.

#### Cenário D — Protótipo ou equipe pequena

Em uma equipe pequena com uma única aplicação e prazo curto, a tentação é colocar tudo na aplicação: é mais rápido, todo mundo conhece a linguagem, e não há segunda aplicação para se preocupar.

O argumento tem um mérito real — velocidade de entrega importa, e sobre-engenharia mata protótipos. Mas o cálculo custo-benefício aqui é bastante assimétrico:

- Declarar `PRIMARY KEY`, `NOT NULL`, `FOREIGN KEY`, `UNIQUE` e alguns `CHECK` custa **minutos** na criação do esquema e não exige nenhum conhecimento avançado.
- Adicionar essas mesmas restrições depois, sobre uma base que já acumulou dados inconsistentes, custa **dias** de limpeza manual, com decisões irreversíveis sobre qual duplicata manter e o que fazer com linhas órfãs.

Ou seja: o que se economiza é pequeno e o que se arrisca é grande. Além disso, protótipos que dão certo viram sistemas — e viram sistemas com a base de dados que o protótipo produziu.

A recomendação para esse cenário, portanto, não é "faça tudo no banco", e sim: **declare desde o início as restrições estruturais baratas** (identidade, obrigatoriedade, unicidade, integridade referencial, domínios), mantenha fluxo e política na aplicação, e adie *triggers* e procedimentos armazenados até que haja uma necessidade concreta que justifique o custo de manutenção.

#### Posição fundamentada

Os quatro cenários apontam na mesma direção, e ela não é nenhum dos extremos. A posição defendida é:

> **A regra deve ser alocada segundo a natureza da garantia que ela exige, e não segundo preferência tecnológica. Invariantes de dados — propriedades que precisam valer para toda linha, independentemente de quem escreveu, e cuja violação corrompe permanentemente a base — pertencem ao banco, na forma declarativa mais simples possível. Regras de processo e de política pertencem à aplicação. Regras críticas ficam nas duas camadas, com papéis distintos: a aplicação previne e explica; o banco garante.**

Essa não é uma escolha de "meio-termo" por conveniência. É a consequência direta de dois fatos técnicos independentes: (i) somente o banco vê todas as escritas, e (ii) somente o banco tem controle de concorrência transacional. Onde nenhum dos dois fatos é relevante para a regra em questão, a aplicação vence com folga em manutenção, teste, expressividade e experiência do usuário.

---

## 2. Exemplos e Casos

### 2.1 Exemplo no PostgreSQL

> **Nota de verificação.** Todo o SQL desta seção foi executado com sucesso em uma instância real do **PostgreSQL 16.14**, incluindo os testes de violação e o teste de concorrência com duas sessões simultâneas. Os resultados apresentados são os retornados pelo servidor, não estimativas. A documentação de referência citada é a da versão corrente (PostgreSQL 18); os recursos utilizados existem desde a versão 15 (`UNIQUE NULLS NOT DISTINCT` não é usado no esquema final; índices parciais, `GENERATED AS IDENTITY` e `CREATE PROCEDURE` estão disponíveis desde as versões 7.2, 10 e 11, respectivamente).

#### Esquema

```sql
CREATE SCHEMA vendas;
SET search_path TO vendas, public;

CREATE TABLE cliente (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome            varchar(120) NOT NULL,
    cpf             char(11)     NOT NULL,
    data_nascimento date         NOT NULL,
    email           varchar(160),
    criado_em       timestamptz  NOT NULL DEFAULT now(),
    CONSTRAINT uq_cliente_cpf             UNIQUE (cpf),
    CONSTRAINT ck_cliente_cpf_formato     CHECK (cpf ~ '^[0-9]{11}$'),
    CONSTRAINT ck_cliente_nome_preenchido CHECK (length(btrim(nome)) > 0),
    CONSTRAINT ck_cliente_nascimento_plausivel
        CHECK (data_nascimento >= DATE '1900-01-01')
);

CREATE TABLE produto (
    id        bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku       varchar(30)   NOT NULL,
    descricao varchar(200)  NOT NULL,
    preco     numeric(12,2) NOT NULL,
    estoque   integer       NOT NULL DEFAULT 0,
    ativo     boolean       NOT NULL DEFAULT true,
    CONSTRAINT uq_produto_sku     UNIQUE (sku),
    CONSTRAINT ck_produto_preco   CHECK (preco > 0),
    CONSTRAINT ck_produto_estoque CHECK (estoque >= 0)
);

CREATE TYPE status_pedido AS ENUM ('ABERTO', 'FATURADO', 'CANCELADO');

CREATE TABLE pedido (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id bigint        NOT NULL,
    status     status_pedido NOT NULL DEFAULT 'ABERTO',
    criado_em  timestamptz   NOT NULL DEFAULT now(),
    fechado_em timestamptz,
    CONSTRAINT fk_pedido_cliente FOREIGN KEY (cliente_id)
        REFERENCES cliente (id) ON DELETE RESTRICT,
    CONSTRAINT ck_pedido_fechamento CHECK (
        (status =  'ABERTO' AND fechado_em IS NULL) OR
        (status <> 'ABERTO' AND fechado_em IS NOT NULL)
    )
);

-- "Um cliente não pode ter dois pedidos em aberto":
-- restrição sobre CONJUNTO de linhas, resolvida de forma DECLARATIVA
CREATE UNIQUE INDEX uq_pedido_aberto_por_cliente
    ON pedido (cliente_id)
    WHERE status = 'ABERTO';

CREATE TABLE item_pedido (
    pedido_id      bigint        NOT NULL,
    produto_id     bigint        NOT NULL,
    quantidade     integer       NOT NULL,
    preco_unitario numeric(12,2) NOT NULL,
    CONSTRAINT pk_item_pedido  PRIMARY KEY (pedido_id, produto_id),
    CONSTRAINT fk_item_pedido  FOREIGN KEY (pedido_id)  REFERENCES pedido  (id) ON DELETE CASCADE,
    CONSTRAINT fk_item_produto FOREIGN KEY (produto_id) REFERENCES produto (id) ON DELETE RESTRICT,
    CONSTRAINT ck_item_quantidade CHECK (quantidade > 0),
    CONSTRAINT ck_item_preco      CHECK (preco_unitario >= 0)
);
```

#### Explicação regra a regra

| Objeto | Tipo | O que garante | Por que no banco |
|---|---|---|---|
| `cliente.id`, `produto.id`, `pedido.id` | `PRIMARY KEY` (identidade) | Cada linha é única e endereçável | Pré-requisito de qualquer referência; chave substituta evita que uma mudança de CPF quebre referências |
| `uq_cliente_cpf` | `UNIQUE` | Não existem dois clientes com o mesmo CPF | Invariante de identificação; violação é irreversível na prática |
| `ck_cliente_cpf_formato` | `CHECK` | CPF tem exatamente 11 dígitos numéricos | Domínio do atributo; **não** valida dígitos verificadores (ver observação abaixo) |
| `ck_cliente_nome_preenchido` | `CHECK` | Nome não é string vazia ou só de espaços | `NOT NULL` sozinho aceita `''`; o `CHECK` fecha essa brecha |
| `ck_produto_preco` | `CHECK` | Preço estritamente positivo | Valor impossível no domínio; `numeric(12,2)` evita ainda os erros de arredondamento de ponto flutuante |
| `ck_produto_estoque` | `CHECK` | Estoque nunca fica negativo | Última linha de defesa da regra de venda sem estoque |
| `fk_pedido_cliente` | `FOREIGN KEY` | Todo pedido pertence a um cliente existente | Integridade referencial; `ON DELETE RESTRICT` impede apagar cliente com histórico |
| `fk_item_pedido` | `FOREIGN KEY` `ON DELETE CASCADE` | Itens não sobrevivem ao pedido | Item é parte dependente do pedido (composição) |
| `fk_item_produto` | `FOREIGN KEY` `ON DELETE RESTRICT` | Produto vendido não pode ser apagado | Preserva rastreabilidade do que foi vendido |
| `pk_item_pedido` | `PRIMARY KEY` composta | O mesmo produto não aparece duas vezes no mesmo pedido | Evita ambiguidade no cálculo do total |
| `ck_pedido_fechamento` | `CHECK` | `fechado_em` preenchido se e somente se o pedido não estiver aberto | Coerência entre duas colunas da mesma linha — caso típico de `CHECK` |
| `uq_pedido_aberto_por_cliente` | Índice único parcial | No máximo um pedido `ABERTO` por cliente | Regra entre linhas resolvida sem *trigger* |
| `status_pedido` | Tipo `ENUM` | Status só assume valores previstos | Domínio fechado; alternativa portável seria `varchar` + `CHECK (status IN (...))` |

**Observação sobre o CPF.** O banco garante *formato* e *unicidade*. A validação dos **dígitos verificadores** é um algoritmo, não uma propriedade estrutural, e pertence à aplicação: implementá-la em `CHECK` produziria uma expressão longa, difícil de auditar e cara de alterar caso a regra mude. Esse é um exemplo concreto de uma única "regra do CPF" corretamente dividida entre camadas.

**Observação sobre a idade mínima de 18 anos.** Foi deliberadamente **não** implementada como `CHECK (data_nascimento <= CURRENT_DATE - INTERVAL '18 years')`. O PostgreSQL aceita essa expressão — foi testado e a tabela é criada sem erro —, mas ela é conceitualmente inadequada: `CHECK` só é avaliado no momento da gravação, de modo que a restrição expressa "tinha 18 anos quando foi cadastrado", e não uma invariante permanente. Se amanhã a regra virar 21 anos, as linhas antigas continuam gravadas e a nova restrição só valerá para novos registros, criando uma garantia que parece existir mas é parcial. Além disso, `CURRENT_DATE` não é uma função imutável, e a documentação alerta contra restrições cujo valor pode mudar com o tempo, inclusive pelo risco em `dump`/`restore` (POSTGRESQL, 2026a). A alocação correta é: **regra de elegibilidade na aplicação** (que sabe a data da operação, a política vigente e a mensagem a exibir) + `CHECK` de plausibilidade no banco (`data_nascimento >= '1900-01-01'`, e opcionalmente `<= CURRENT_DATE` como sanidade), impedindo datas absurdas vindas de qualquer origem.

#### CPF único: demonstração

```sql
INSERT INTO cliente (nome, cpf, data_nascimento)
VALUES ('Ana Souza', '11111111111', DATE '1990-05-10');

INSERT INTO cliente (nome, cpf, data_nascimento)
VALUES ('Ana Clone', '11111111111', DATE '1991-01-01');
```

Resultado real do servidor:

```
ERROR:  duplicate key value violates unique constraint "uq_cliente_cpf"
DETAIL:  Key (cpf)=(11111111111) already exists.
```

O ponto importante: **não há janela de corrida**. Uma verificação equivalente na aplicação (`SELECT` para ver se o CPF existe, depois `INSERT`) permite que duas requisições simultâneas consultem, ambas não encontrem nada e ambas insiram. A restrição `UNIQUE` é verificada pelo índice no momento da gravação, dentro da transação, e uma das duas necessariamente falha.

#### Pedido em aberto: demonstração

```sql
INSERT INTO pedido (cliente_id) VALUES (1);  -- INSERT 0 1
INSERT INTO pedido (cliente_id) VALUES (1);  -- segundo pedido em aberto
```

```
ERROR:  duplicate key value violates unique constraint "uq_pedido_aberto_por_cliente"
DETAIL:  Key (cliente_id)=(1) already exists.
```

Ao fechar o primeiro pedido, o segundo passa a ser permitido — a linha faturada sai do índice parcial:

```sql
UPDATE pedido SET status = 'FATURADO', fechado_em = now() WHERE id = 1;
INSERT INTO pedido (cliente_id) VALUES (1) RETURNING id, status;
--  id | status
-- ----+--------
--   3 | ABERTO
```

Este é o exemplo mais didático do trabalho: uma regra de negócio sobre **conjunto de linhas**, que muita gente implementaria com *trigger* (e a maioria implementaria com uma consulta na aplicação, sujeita a corrida), resolvida com um índice único parcial — declarativo, atômico e à prova de concorrência.

#### Estoque e concorrência

A regra "não vender sem estoque" tem duas partes: a **decisão** (a aplicação precisa saber se pode vender e informar o usuário) e a **invariante** (o estoque não pode ficar negativo, nunca, sob nenhuma ordem de execução).

A implementação **incorreta**, e muito comum, é esta:

```sql
-- ANTIPADRÃO: sujeito a lost update
SELECT estoque FROM produto WHERE id = 2;   -- aplicação lê 1
-- ...aplicação decide que pode vender...
UPDATE produto SET estoque = estoque - 1 WHERE id = 2;
```

Entre o `SELECT` e o `UPDATE`, outra sessão pode ter consumido o mesmo estoque. A implementação **correta** faz a decisão e a atualização no mesmo comando atômico:

```sql
BEGIN;

    INSERT INTO pedido (cliente_id) VALUES (2) RETURNING id;

    UPDATE produto
       SET estoque = estoque - 2
     WHERE id = 2
       AND estoque >= 2;
    -- a aplicação verifica o número de linhas afetadas:
    -- 0 linhas  => estoque insuficiente => ROLLBACK e mensagem ao usuário
    -- 1 linha   => reserva efetivada

    INSERT INTO item_pedido (pedido_id, produto_id, quantidade, preco_unitario)
    VALUES (:pedido_id, 2, 2, 120.00);

COMMIT;
```

**Teste de concorrência real.** Com `produto.estoque = 3`, duas sessões executaram o mesmo `UPDATE` simultaneamente (a sessão A mantendo a transação aberta por três segundos antes do `COMMIT`):

| Sessão | Comando | Retorno |
|---|---|---|
| A | `UPDATE produto SET estoque = estoque - 2 WHERE id = 2 AND estoque >= 2;` | `UPDATE 1` |
| B | mesmo comando, iniciado 1 s depois | bloqueia até o `COMMIT` de A, e retorna `UPDATE 0` |

Estoque final: **1**. Nenhuma venda a descoberto, nenhum estoque negativo.

O mecanismo é o seguinte: no nível *Read Committed* (padrão), quando um `UPDATE` encontra uma linha já bloqueada por outra transação, ele espera; quando o bloqueio é liberado por `COMMIT`, o comando **reavalia a condição do `WHERE` sobre a versão atualizada da linha** antes de aplicar a alteração (POSTGRESQL, 2026e). Como o estoque passou a 1 e a condição exigia `>= 2`, a linha deixou de qualificar e o `UPDATE` afetou zero linhas. É exatamente essa reavaliação que torna o padrão seguro — e é ela que falta na versão com `SELECT` separado.

Duas observações importantes:

1. **`UPDATE 0` não é erro; é resposta.** A camada de serviço precisa verificar a contagem de linhas afetadas e tratar zero como "estoque insuficiente". Ignorar esse retorno é um bug comum que produz pedidos sem baixa de estoque.
2. **O `CHECK (estoque >= 0)` continua sendo necessário**, mesmo com o padrão correto acima. Ele é a garantia que sobrevive a um código novo escrito sem o `AND estoque >= :qtd`, a um `UPDATE` manual e a uma rotina de importação. Testado:

```sql
UPDATE produto SET estoque = estoque - 99 WHERE id = 2;
-- ERROR:  new row for relation "produto" violates check constraint "ck_produto_estoque"
-- DETAIL:  Failing row contains (2, MOU-002, Mouse sem fio, 120.00, -96, t).
```

Alternativas legítimas para casos mais complexos: `SELECT ... FOR UPDATE` antes de calcular (bloqueio pessimista explícito, útil quando a decisão envolve vários produtos e precisa de ordenação de bloqueios para evitar *deadlock*), ou o nível `SERIALIZABLE`, que detecta anomalias automaticamente ao custo de a aplicação ter de repetir transações abortadas por erro de serialização (PORTS; GRITTNER, 2012).

#### Atomicidade: demonstração

```sql
BEGIN;
    INSERT INTO pedido (cliente_id) VALUES (2);
    UPDATE produto SET estoque = estoque - 99 WHERE id = 1 AND estoque >= 99;  -- UPDATE 0
ROLLBACK;

SELECT count(*) AS pedidos_cliente_2 FROM pedido WHERE cliente_id = 2;
-- 0
```

O pedido chegou a ser inserido dentro da transação e desapareceu por completo com o `ROLLBACK`. É a propriedade que garante que uma falha no meio do processo não deixa pedidos órfãos nem estoque baixado sem venda correspondente — e é o motivo pelo qual **a camada de serviço precisa controlar a transação**, e não apenas emitir comandos isolados.

#### Um *trigger*, com justificativa

O único *trigger* deste esquema existe porque a regra que ele implementa **não é expressável declarativamente**: "itens de um pedido já faturado ou cancelado não podem ser inseridos, alterados nem excluídos". A restrição depende do valor de `pedido.status`, que está em **outra tabela** — território explicitamente fora do alcance de um `CHECK` (POSTGRESQL, 2026a).

```sql
CREATE OR REPLACE FUNCTION fn_bloqueia_item_pedido_fechado()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_status    status_pedido;
    v_pedido_id bigint := COALESCE(NEW.pedido_id, OLD.pedido_id);
BEGIN
    SELECT p.status INTO v_status
      FROM pedido p
     WHERE p.id = v_pedido_id
     FOR SHARE;   -- impede que o pedido seja faturado concorrentemente

    IF v_status <> 'ABERTO' THEN
        RAISE EXCEPTION
            'Pedido % está com status % e não aceita alteração de itens',
            v_pedido_id, v_status
            USING ERRCODE = 'integrity_constraint_violation';
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_item_pedido_imutavel
    BEFORE INSERT OR UPDATE OR DELETE ON item_pedido
    FOR EACH ROW
    EXECUTE FUNCTION fn_bloqueia_item_pedido_fechado();
```

Resultado do teste (pedido 1 já faturado):

```
ERROR:  Pedido 1 está com status FATURADO e não aceita alteração de itens
```

O `FOR SHARE` não é decorativo: sem ele, uma transação poderia inserir um item enquanto outra fatura o mesmo pedido, e ambas confirmariam. O bloqueio compartilhado sobre a linha do pedido impede essa combinação.

#### Procedimento armazenado, com justificativa

```sql
CREATE OR REPLACE PROCEDURE sp_fechar_pedido(p_pedido_id bigint)
LANGUAGE plpgsql
AS $$
DECLARE
    v_itens int;
BEGIN
    SELECT count(*) INTO v_itens FROM item_pedido WHERE pedido_id = p_pedido_id;
    IF v_itens = 0 THEN
        RAISE EXCEPTION 'Pedido % não possui itens', p_pedido_id;
    END IF;

    UPDATE pedido
       SET status = 'FATURADO', fechado_em = now()
     WHERE id = p_pedido_id AND status = 'ABERTO';

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Pedido % não está aberto', p_pedido_id;
    END IF;
END;
$$;

CALL sp_fechar_pedido(3);
--  id | status   | tem_fechamento
-- ----+----------+----------------
--   3 | FATURADO | t
```

**Justificativa honesta:** este procedimento é apresentado para ilustrar a diferença entre procedimento, função e *trigger*, mas **não** é a alocação recomendada por este trabalho. A regra "pedido só pode ser faturado se tiver itens" é uma regra de processo, e o lugar natural dela é a camada de serviço. O caso em que valeria a pena movê-la para o banco é aquele em que o faturamento pode ser disparado por vários sistemas diferentes e a organização precisa de uma garantia única — o mesmo raciocínio do Cenário A. A alternativa puramente declarativa (um *constraint trigger* `DEFERRABLE` verificado no fim da transação) existe, mas acrescenta complexidade que só se justifica quando a garantia é realmente crítica.

---

### 2.2 Exemplo de validação na aplicação

```text
FUNÇÃO criarPedido(clienteId, listaDeItens, contexto):

    // ---------- 1. Validação de entrada (borda) ----------
    SE listaDeItens está vazia:
        RETORNAR Erro("Adicione ao menos um produto ao pedido")

    PARA CADA item EM listaDeItens:
        SE item.quantidade não é inteiro OU item.quantidade <= 0:
            RETORNAR Erro("Quantidade inválida para o produto " + item.sku)

    // ---------- 2. Regras de domínio (aplicação) ----------
    cliente = repositorioCliente.buscarPorId(clienteId)
    SE cliente é nulo:
        RETORNAR Erro("Cliente não encontrado")
    SE cliente.estáBloqueado():
        RETORNAR Erro("Cliente bloqueado para novas compras")
    SE cliente.idadeEm(contexto.dataAtual) < 18:
        RETORNAR Erro("Cliente precisa ter 18 anos ou mais")

    SE repositorioPedido.existePedidoAberto(clienteId):
        RETORNAR Erro("Finalize seu pedido em aberto antes de criar outro")

    produtos = repositorioProduto.buscarPorIds(idsDe(listaDeItens))
    PARA CADA item EM listaDeItens:
        SE produtos[item.produtoId] é nulo OU não está ativo:
            RETORNAR Erro("Produto indisponível: " + item.sku)
        SE produtos[item.produtoId].estoque < item.quantidade:
            RETORNAR Erro("Restam apenas " + estoque + " unidades de " + item.sku)

    total = calcularTotal(listaDeItens, produtos, cliente, contexto.campanhaVigente)

    // ---------- 3. Persistência transacional ----------
    INICIAR TRANSAÇÃO:

        pedidoId = repositorioPedido.inserir(clienteId, status = 'ABERTO')
        // se o cliente já tiver pedido aberto criado por outra requisição
        // simultânea, o índice único parcial faz este INSERT falhar

        PARA CADA item EM listaDeItens:
            linhasAfetadas = repositorioProduto.baixarEstoque(item.produtoId, item.quantidade)
            // executa: UPDATE produto SET estoque = estoque - :qtd
            //          WHERE id = :id AND estoque >= :qtd
            SE linhasAfetadas = 0:
                DESFAZER TRANSAÇÃO
                RETORNAR Erro("Estoque de " + item.sku + " esgotou durante a compra")

            repositorioItem.inserir(pedidoId, item.produtoId, item.quantidade, precoUnitario)

        CONFIRMAR TRANSAÇÃO

    // ---------- 4. Tratamento de violações vindas do banco ----------
    EM CASO DE ErroDeRestrição(e):
        DESFAZER TRANSAÇÃO
        SE e.restrição = 'uq_pedido_aberto_por_cliente':
            RETORNAR Erro("Você já possui um pedido em aberto")
        SE e.restrição = 'ck_produto_estoque':
            RETORNAR Erro("Não foi possível reservar o estoque solicitado")
        REGISTRAR_LOG(e)
        RETORNAR Erro("Não foi possível concluir o pedido. Tente novamente.")

    RETORNAR Sucesso(pedidoId, total)
```

#### O que pertence a cada camada

| Trecho | Camada | Por quê |
|---|---|---|
| Lista vazia, quantidade inteira positiva | Aplicação (validação de entrada) | Formato e sintaxe; retorno imediato ao usuário, sem ida ao banco |
| Cliente existe, não está bloqueado, tem 18 anos | Aplicação (domínio) | Elegibilidade dependente de política e de data de referência; muda com o negócio |
| Cálculo do total, campanha, desconto | Aplicação (domínio) | Política comercial volátil; depende de contexto externo à linha |
| Mensagens ao usuário | Aplicação | Banco produz diagnóstico técnico, não texto de interface |
| Consulta prévia de estoque e de pedido em aberto | Aplicação (**informativa**) | Serve para avisar cedo; **não é garantia**, porque há janela de corrida entre a consulta e a gravação |
| Delimitação da transação | Aplicação orquestra, banco executa | Só a aplicação sabe quais operações formam a unidade lógica; só o banco pode garanti-la |
| `UPDATE ... WHERE estoque >= :qtd` + verificação de linhas afetadas | **Banco garante, aplicação reage** | Decisão e escrita no mesmo comando atômico |
| Um único pedido aberto por cliente | **Banco** (índice único parcial) | Duas requisições simultâneas passam pela verificação da aplicação; só o índice impede as duas gravações |
| CPF único, FKs, `estoque >= 0`, `preco > 0` | **Banco** | Invariantes de dados; valem para toda escrita, de qualquer origem |

O detalhe arquitetural mais importante do pseudocódigo está no bloco 4: **a aplicação trata as violações de restrição do banco como parte do fluxo normal**, e não como falha inesperada. Isso é o que permite verificar antes (boa experiência) sem depender dessa verificação (garantia), traduzindo o erro técnico em mensagem útil quando a corrida efetivamente acontece. Uma aplicação que só verifica antes está exposta; uma que só reage ao erro do banco funciona, mas oferece experiência ruim. O padrão correto usa as duas.

---

### 2.3 Caso prático: sistema de vendas

#### Regra 1 — CPF único por cliente

| Aspecto | Análise |
|---|---|
| **Regra** | Não podem existir dois clientes ativos com o mesmo CPF. |
| **Camada principal** | **Banco de dados** — `CONSTRAINT uq_cliente_cpf UNIQUE (cpf)`. |
| **Justificativa** | É uma invariante de identificação. A violação é silenciosa no momento em que ocorre e destrutiva depois: os dois registros acumulam pedidos, e a fusão posterior exige decisões manuais sobre qual endereço, qual histórico e qual limite de crédito prevalece. Além disso, a verificação "consultar antes de inserir" na aplicação tem janela de corrida: dois cadastros simultâneos passam ambos pela consulta. `UNIQUE` é verificado no índice, dentro da transação, sem janela. |
| **Proteção complementar** | Na aplicação: validação de formato e de dígitos verificadores (algoritmo, não pertence ao esquema); consulta prévia para exibir "este CPF já está cadastrado — deseja recuperar sua conta?" antes de o usuário preencher o formulário inteiro; e tradução do erro `uq_cliente_cpf` para mensagem amigável quando a corrida ocorrer. No banco: `CHECK` de formato, para que qualquer origem de escrita grave CPF com 11 dígitos. |

#### Regra 2 — Não vender sem estoque disponível

| Aspecto | Análise |
|---|---|
| **Regra** | A soma das unidades vendidas nunca pode exceder o estoque disponível; o estoque nunca fica negativo. |
| **Camada principal** | **Compartilhada, com papéis distintos.** A garantia fica no banco (`CHECK (estoque >= 0)` + baixa atômica com `UPDATE ... WHERE estoque >= :qtd` dentro de transação); a decisão e a comunicação ficam na aplicação. |
| **Justificativa** | É a regra em que o argumento da concorrência é decisivo. Nenhuma verificação feita na aplicação antes da gravação sobrevive a duas requisições simultâneas: ambas leem o mesmo valor, ambas concluem que podem vender. O teste da seção 2.1 mostra o comportamento correto — a segunda sessão recebeu `UPDATE 0`. Por outro lado, seria péssimo deixar a regra *apenas* no banco: o usuário só descobriria a indisponibilidade ao final da compra, e a mensagem seria uma violação de restrição. |
| **Proteção complementar** | Na aplicação: consulta de disponibilidade na vitrine e no carrinho, com mensagem clara ("restam 2 unidades"); verificação do número de linhas afetadas pelo `UPDATE` e `ROLLBACK` com mensagem específica; política de reserva temporária, se o negócio exigir. No banco: além do `CHECK`, ordenar as baixas por `produto_id` quando o pedido tiver vários itens, reduzindo risco de *deadlock* entre transações concorrentes. |

#### Regra 3 — Um cliente não pode ter dois pedidos em aberto

| Aspecto | Análise |
|---|---|
| **Regra** | Para cada cliente, no máximo um pedido com status `ABERTO`. |
| **Camada principal** | **Banco de dados** — índice único parcial `CREATE UNIQUE INDEX uq_pedido_aberto_por_cliente ON pedido (cliente_id) WHERE status = 'ABERTO'`. |
| **Justificativa** | É uma restrição sobre um **conjunto de linhas**, exatamente o tipo de regra que um `CHECK` não consegue expressar, e que na aplicação está sujeita a corrida (duas abas do navegador, dois dispositivos, um duplo clique). O índice parcial resolve o caso de forma declarativa, atômica e barata — sem *trigger* e sem código procedural. Vale notar que a regra tem um componente de política: *o que conta como "aberto"* é decisão do negócio, e mudanças nessa definição exigem recriar o índice. Como essa definição é estrutural e estável (faz parte da máquina de estados do pedido), o custo é aceitável; se o conjunto de status "abertos" mudasse a cada campanha, a conclusão seria outra. |
| **Proteção complementar** | Na aplicação: verificar antes de criar e redirecionar o usuário ao pedido existente, em vez de exibir erro; desabilitar o botão após o primeiro clique; e tratar a violação de `uq_pedido_aberto_por_cliente` como "você já possui um pedido em aberto", com link para ele. |

#### Critério de decisão consolidado

Das três análises emerge um teste prático de três perguntas, aplicável a qualquer regra nova:

1. **A violação corrompe dados de forma irreversível?** Se sim → banco.
2. **A regra pode ser violada por duas operações simultâneas que, isoladamente, são válidas?** Se sim → banco (restrição declarativa ou comando atômico dentro de transação).
3. **A regra muda por decisão comercial, depende de contexto externo ou existe principalmente para informar o usuário?** Se sim → aplicação.

Quando as respostas 1 ou 2 e a resposta 3 forem simultaneamente "sim", a regra vai para as duas camadas — com a aplicação prevenindo e o banco garantindo.

---

## 3. Referências

### Documentação oficial do PostgreSQL

*Consultada na documentação da versão 18 (versão corrente em agosto de 2026). Os exemplos foram executados em PostgreSQL 16.14.*

- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — 5.5. Constraints**. 2026a. Disponível em: <https://www.postgresql.org/docs/current/ddl-constraints.html>. *(Base das seções 1.2 e 2.1: limitações do `CHECK`, comportamento de `UNIQUE`, exigências e indexação de `FOREIGN KEY`.)*
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — SET CONSTRAINTS**. 2026b. Disponível em: <https://www.postgresql.org/docs/current/sql-set-constraints.html>. *(Fonte da afirmação de que `NOT NULL` e `CHECK` são sempre verificadas imediatamente e não podem ser adiadas.)*
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — Chapter 37. Triggers**. 2026c. Disponível em: <https://www.postgresql.org/docs/current/triggers.html>.
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — 41.10. Trigger Functions (PL/pgSQL)**. 2026d. Disponível em: <https://www.postgresql.org/docs/current/plpgsql-trigger.html>.
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — 13.2. Transaction Isolation**. 2026e. Disponível em: <https://www.postgresql.org/docs/current/transaction-iso.html>. *(Níveis de isolamento e reavaliação da condição do `UPDATE` após bloqueio em Read Committed.)*
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — 11.8. Partial Indexes**. 2026f. Disponível em: <https://www.postgresql.org/docs/current/indexes-partial.html>. *(Base do índice único parcial usado para "um pedido aberto por cliente".)*
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — 13.3. Explicit Locking**. 2026g. Disponível em: <https://www.postgresql.org/docs/current/explicit-locking.html>. *(`SELECT ... FOR UPDATE` / `FOR SHARE`.)*
- POSTGRESQL GLOBAL DEVELOPMENT GROUP. **PostgreSQL 18 Documentation — CREATE PROCEDURE**. 2026h. Disponível em: <https://www.postgresql.org/docs/current/sql-createprocedure.html>.

### Livros de Banco de Dados

- SILBERSCHATZ, A.; KORTH, H. F.; SUDARSHAN, S. **Database System Concepts**. 7. ed. New York: McGraw-Hill, 2020. ISBN 978-0-07-802215-9. Material de apoio: <https://www.db-book.com/>. *(Restrições de integridade, transações, controle de concorrência.)*
- ELMASRI, R.; NAVATHE, S. B. **Sistemas de Banco de Dados**. 7. ed. São Paulo: Pearson, 2018. *(Integridade de entidade, integridade referencial e restrições de domínio.)*
- DATE, C. J. **An Introduction to Database Systems**. 8. ed. Boston: Addison-Wesley, 2003. *(Tratamento clássico de integridade como parte do modelo relacional.)*
- GRAY, J.; REUTER, A. **Transaction Processing: Concepts and Techniques**. San Francisco: Morgan Kaufmann, 1993. *(Referência canônica sobre transações.)*

### Artigos científicos

- HÄRDER, T.; REUTER, A. Principles of Transaction-Oriented Database Recovery. **ACM Computing Surveys**, v. 15, n. 4, p. 287–317, dez. 1983. DOI: <https://doi.org/10.1145/289.291>. *(Artigo que consolidou o acrônimo ACID.)*
- BERENSON, H.; BERNSTEIN, P.; GRAY, J.; MELTON, J.; O'NEIL, E.; O'NEIL, P. A Critique of ANSI SQL Isolation Levels. In: **ACM SIGMOD International Conference on Management of Data**, 1995. *(Análise das anomalias de concorrência e das limitações da definição de níveis de isolamento do padrão.)*
- PORTS, D. R. K.; GRITTNER, K. Serializable Snapshot Isolation in PostgreSQL. **Proceedings of the VLDB Endowment**, v. 5, n. 12, p. 1850–1861, 2012. Disponível em: <https://arxiv.org/abs/1208.4179>. *(Implementação do nível `SERIALIZABLE` no PostgreSQL.)*

### Engenharia de Software e Arquitetura

- FOWLER, M. **Patterns of Enterprise Application Architecture**. Boston: Addison-Wesley, 2002. *(Padrões *Service Layer*, *Domain Model* e *Transaction Script*.)*
- FOWLER, M. **IntegrationDatabase** e **ApplicationDatabase**. martinfowler.com, 2004. Disponível em: <https://martinfowler.com/bliki/IntegrationDatabase.html> e <https://martinfowler.com/bliki/ApplicationDatabase.html>. *(Distinção usada no Cenário A.)*
- EVANS, E. **Domain-Driven Design: Tackling Complexity in the Heart of Software**. Boston: Addison-Wesley, 2003. *(Regras de domínio modeladas em objetos; conceito de invariante agregada.)*
- KLEPPMANN, M. **Designing Data-Intensive Applications**. Sebastopol: O'Reilly, 2017. *(Discussão prática sobre garantias de consistência e concorrência em sistemas reais.)*
- AMBLER, S. W.; SADALAGE, P. J. **Refactoring Databases: Evolutionary Database Design**. Boston: Addison-Wesley, 2006. *(Custo e técnica de evolução de esquemas em bases compartilhadas.)*
- SOMMERVILLE, I. **Engenharia de Software**. 10. ed. São Paulo: Pearson, 2019. *(Arquitetura em camadas e separação de responsabilidades.)*

### Normas e legislação

- BUSINESS RULES GROUP. **Defining Business Rules ~ What Are They Really?** 3. ed., jul. 2000. Disponível em: <https://www.businessrulesgroup.org/first_paper/BRG-whatisBR_3ed.pdf>. *(Definição de regra de negócio adotada na seção 1.1.)*
- ISO/IEC 9075-2:2023. **Information technology — Database languages — SQL — Part 2: Foundation (SQL/Foundation)**. Genebra: ISO, 2023. *(Padrão que define as restrições declarativas usadas.)*
- BRASIL. **Lei nº 10.406, de 10 de janeiro de 2002** (Código Civil), art. 5º. Disponível em: <https://www.planalto.gov.br/ccivil_03/leis/2002/l10406compilada.htm>. *(Maioridade civil aos 18 anos, base da regra de idade mínima.)*
- BRASIL. **Lei nº 13.709, de 14 de agosto de 2018** (Lei Geral de Proteção de Dados Pessoais). Disponível em: <https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm>. *(Referência do Cenário B.)*

---

## 4. Conclusões

### O banco ou a aplicação deve ser o "dono" da regra?

A pergunta, formulada assim, contém uma armadilha: pressupõe que "a regra" seja uma coisa só. Na prática, quase toda regra de negócio relevante tem um **componente de invariante** (o que precisa ser verdade sobre os dados) e um **componente de processo** (como, quando e por quem a operação acontece, e o que o usuário vê). Esses dois componentes têm donos diferentes.

A posição defendida por este grupo: **o banco é o dono da invariante; a aplicação é a dona do processo.** O banco não precisa — nem deve — conhecer a política de frete; a aplicação não pode — nem consegue — garantir que nenhuma outra escrita produza estoque negativo. Onde os dois componentes existem na mesma regra, ela aparece nas duas camadas com papéis diferentes, e isso **não é duplicação**: é defesa em profundidade, o mesmo princípio que faz um sistema validar entrada no cliente e no servidor.

### Quais regras precisam obrigatoriamente de proteção no banco?

Aquelas que satisfazem pelo menos um destes critérios:

1. **Violação irreversível** — duplicatas de identificadores de negócio (CPF, SKU), linhas órfãs, valores impossíveis. Depois de gravados, esses estados não têm correção automática confiável.
2. **Sensibilidade a concorrência** — estoque não negativo, unicidade condicional (um pedido aberto por cliente), qualquer regra do tipo "no máximo N". A aplicação não tem mecanismo para garanti-las; o banco tem.
3. **Necessidade de garantia auditável** — situações de exigência legal ou fiscal em que é preciso demonstrar que o estado inválido não pode ter ocorrido, independentemente do código que rodou.

Nesta ordem de preferência de implementação: primeiro restrição declarativa (`PRIMARY KEY`, `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, `CHECK`, índice único parcial, tipo adequado); depois comando atômico dentro de transação; e só então *trigger*, quando nenhuma das anteriores for capaz de expressar a regra.

### Quais são melhores na aplicação?

Fluxos e orquestração; cálculos de preço, desconto, frete e imposto; políticas comerciais com vigência; validação de formato e algoritmos de verificação (dígitos do CPF); elegibilidade dependente de contexto e de data de referência; integrações com serviços externos; e todas as decisões que determinam **o que o usuário vê e quando**. São regras que mudam por decisão de negócio, que ganham em clareza quando escritas em linguagem de aplicação e que se beneficiam de testes rápidos e *deploy* reversível.

### Quando usar ambos?

Sempre que a regra for, ao mesmo tempo, uma invariante e um ponto de interação com o usuário. O padrão é: **a aplicação previne, o banco garante, e a aplicação traduz a garantia quando ela é acionada.** No exemplo do estoque, isso significa consultar a disponibilidade e mostrá-la na vitrine (aplicação), executar a baixa com `UPDATE ... WHERE estoque >= :qtd` dentro da transação e manter `CHECK (estoque >= 0)` (banco), e converter `UPDATE 0` ou a violação da restrição em uma mensagem compreensível (aplicação). Nenhuma das três partes é redundante: elas cobrem, respectivamente, a experiência, a corrida e o caminho de escrita inesperado.

### Quais riscos existem quando a regra existe somente na aplicação?

- **Erosão silenciosa por múltiplos caminhos de escrita.** Toda importação, todo *script* de correção, todo serviço novo e todo acesso administrativo é uma oportunidade de violar a regra sem que nenhum erro seja emitido.
- **Condições de corrida.** Verificar-e-depois-gravar não é atômico. O problema aparece sob carga — justamente quando é mais caro — e é notoriamente difícil de reproduzir em ambiente de teste.
- **Divergência entre implementações.** Ver a próxima resposta.
- **Impossibilidade de afirmar propriedades sobre a base.** Sem restrição declarada, "não existem CPFs duplicados" é uma hipótese sobre o histórico do código, não um fato verificável sobre os dados.

### Quais riscos existem quando toda a regra fica somente no banco?

- **Rigidez diante de mudanças de negócio.** Cada revisão de política vira migração de esquema, com coordenação entre equipes e risco em produção — e regras rígidas sobre políticas voláteis acabam sendo removidas na primeira urgência, deixando o sistema pior do que se elas nunca tivessem existido ali.
- **Perda de legibilidade e de análise de impacto.** Lógica em *triggers* e procedimentos produz efeitos invisíveis para quem lê o código da aplicação; entender o comportamento do sistema passa a exigir inspeção do catálogo do banco.
- **Custo de teste e de escalabilidade.** Testar exige instância real e dados preparados; e a carga se concentra no componente mais caro de escalar da arquitetura.
- **Perda de portabilidade.** Código procedural amarra o sistema ao fornecedor do SGBD de forma difícil de reverter.
- **Experiência do usuário ruim.** Mensagens de violação de restrição são diagnósticos técnicos, inadequados como retorno ao usuário final.
- **Invalidação de dados históricos.** Restrições rígidas aplicadas a políticas que mudaram podem inviabilizar `restore` de *backups* e revalidações.

### O que acontece quando diferentes aplicações implementam a mesma regra de maneiras diferentes?

Acontece a pior categoria de falha: **inconsistência sem erro**. Nenhum sistema quebra, nenhum log registra exceção, todos os testes automatizados passam — e o banco acumula, mês após mês, linhas que violam a regra de negócio original. Como cada aplicação está correta segundo a sua própria definição, não há culpado técnico nem alerta possível; a divergência só aparece quando alguém compara os dados com a expectativa do negócio, tipicamente em uma apuração contábil, uma auditoria ou uma reclamação de cliente.

O custo da correção é muito superior ao da prevenção: é preciso decidir qual definição vale (decisão de negócio, não técnica), corrigir manualmente o histórico, reconciliar registros derivados que já foram gerados a partir dos dados inconsistentes e, só então, unificar as implementações. Uma restrição declarada no banco desde o início transforma esse cenário em um erro imediato, barato e localizado: a segunda aplicação falha na primeira gravação divergente, em desenvolvimento, com uma mensagem que diz exatamente qual restrição foi violada.

### Síntese final

A arquitetura híbrida defendida aqui não é um meio-termo confortável entre duas posições: é a consequência de dois fatos técnicos que nenhuma escolha de estilo altera — **o banco é o único componente que vê todas as escritas, e é o único que controla concorrência transacional**. Toda regra cuja correção dependa de um desses dois fatos precisa de proteção no banco. Todas as demais ficam melhor na aplicação, onde são mais fáceis de ler, testar, mudar e explicar ao usuário.

A pergunta útil, portanto, não é "banco ou aplicação?", e sim: **"o que exatamente esta regra garante, e o que acontece se ela for violada?"** Respondida essa pergunta, a alocação deixa de ser uma questão de preferência e passa a ser uma decisão de engenharia com critério verificável — que é, em última análise, o que este trabalho procurou demonstrar.

---

# Link do Repositório Git

**<https://github.com/PedroBirudevs/bd2026-regra-negocio-grupo-9>**

---

## Apêndice A — Revisão técnica (segunda leitura, na perspectiva do professor)

| # | Item verificado | Resultado |
|---|---|---|
| 1 | **Erros conceituais** | Corrigido o uso impreciso do "C" de ACID: a seção 1.2 explicita que o SGBD garante automaticamente apenas as restrições **declaradas**, e que regras não declaradas continuam sob responsabilidade da transação. Corrigida também a confusão comum entre restrição, *trigger*, procedimento e transação, tratados separadamente com definições distintas. |
| 2 | **Definições vagas** | As três categorias (regra de negócio, integridade de dados, validação de entrada) receberam definição, exemplo e consequência de ausência, além do exemplo unificado do CPF, que mostra as três atuando sobre o mesmo atributo. |
| 3 | **Exemplos tecnicamente incorretos** | O antipadrão `SELECT` + `UPDATE` foi explicitamente rotulado como incorreto e substituído pelo comando atômico. A regra de idade **não** foi implementada como `CHECK` com `CURRENT_DATE`, com justificativa técnica (avaliação apenas na gravação, expressão não imutável). |
| 4 | **O SQL funciona no PostgreSQL?** | Sim. Todo o DDL, os testes de violação, o *trigger*, o procedimento e o teste de concorrência com duas sessões foram executados em PostgreSQL 16.14; as saídas reproduzidas no texto são as retornadas pelo servidor. |
| 5 | **Transações e ACID** | Verificado: atomicidade demonstrada com `ROLLBACK`; consistência qualificada corretamente; isolamento descrito conforme os níveis reais do PostgreSQL (com a ressalva de que *Read Uncommitted* se comporta como *Read Committed*); durabilidade associada ao WAL. |
| 6 | **`CHECK`, `UNIQUE`, `FK`, *triggers*, procedimentos** | Verificado contra a documentação oficial: `CHECK` restrito à linha corrente e não adiável (testado: `ERROR: CHECK constraints cannot be marked DEFERRABLE`); `UNIQUE` tratando nulos como distintos por padrão (testado); `FOREIGN KEY` exigindo índice único do lado referenciado e **não** criando índice do lado referenciante; distinção entre `CREATE FUNCTION` e `CREATE PROCEDURE`. |
| 7 | **Comparação equilibrada** | A tabela 1.4 apresenta vantagem e limitação em cada célula, inclusive nos critérios em que uma das camadas é claramente superior. O Cenário A registra a alternativa arquitetural contrária (serviço único de escrita) e seus custos, em vez de descartá-la. |
| 8 | **A conclusão decorre dos argumentos?** | Sim. A posição híbrida é derivada de dois fatos técnicos declarados (visibilidade total das escritas e controle de concorrência) e não de conveniência; os quatro cenários testam a posição, incluindo o Cenário C, que argumenta contra colocar regras no banco. |
| 9 | **Referências existem e são confiáveis?** | Todas as fontes com link foram verificadas online. Livros e artigos são identificados por autor, título, edição/veículo e ano; DOI e repositório aberto foram incluídos quando existem. Nenhuma referência foi criada para preencher a seção. |
| 10 | **Exigências da atividade atendidas?** | Ver Checklist de conformidade abaixo. |

---

## Checklist de conformidade

| # | Exigência da atividade | Atendida | Onde aparece / como |
|---|---|:---:|---|
| 1 | Estrutura exata solicitada (cabeçalho, resumo, 1.1–1.5, 2.1–2.3, 3, 4, link do repositório) | ✅ | Documento inteiro, na ordem pedida |
| 2 | Resumo executivo com problema, diferença BD × aplicação e conclusão principal | ✅ | Seção "Resumo Executivo", em três parágrafos correspondentes |
| 3 | Conclusão não simplista, com posição fundamentada | ✅ | Resumo Executivo, 1.5 ("Posição fundamentada") e Seção 4 |
| 4 | Definição de regra de negócio, finalidade e exemplos em sistema de vendas | ✅ | 1.1, com definição do Business Rules Group e tabela de sete exemplos |
| 5 | Diferença entre regra de negócio, integridade de dados e validação de entrada | ✅ | 1.1, tabela comparativa + exemplo unificado do CPF |
| 6 | Regras que devem ser protegidas no banco (referencial, unicidade, obrigatoriedade, domínios) | ✅ | 1.1, subseção "Regras que precisam de proteção no banco de dados" |
| 7 | Regras adequadas à aplicação (fluxos, preços, políticas, complexas, voláteis) | ✅ | 1.1, subseção "Regras mais adequadas à camada de aplicação" |
| 8 | Divisão não tratada como absoluta | ✅ | 1.1, subseção "Por que a divisão não é absoluta"; reforçado em 2.3 |
| 9 | Explicação de `CHECK`, `FK`, `UNIQUE`, `NOT NULL`, `PK`, *triggers*, procedimentos, transações e ACID | ✅ | 1.2, cada item com definição, limitação e exemplo |
| 10 | Ao menos 2 vantagens e 2 desvantagens da implementação no banco | ✅ | 1.2, subseções dedicadas (2 + 2, com justificativa técnica) |
| 11 | Consistência centralizada, proteção contra múltiplas aplicações, concorrência, excesso de lógica no banco, portabilidade, manutenção e testes | ✅ | 1.2 (vantagens/desvantagens) e 1.4 (tabela) |
| 12 | Explicação de validação de entrada, camada de serviço, regras de domínio, arquitetura em camadas e separação de responsabilidades | ✅ | 1.3, cinco subseções |
| 13 | Ao menos 2 vantagens e 2 desvantagens da implementação na aplicação | ✅ | 1.3, subseções dedicadas (2 + 2) |
| 14 | Manutenção, testes, evolução, UX, risco de inconsistência e de esquecer/implementar errado a regra | ✅ | 1.3, vantagens 1–2 e desvantagens 1–2 (mais o parágrafo final sobre divergência entre equipes) |
| 15 | Tabela comparativa com os 9 critérios exigidos, não superficial | ✅ | 1.4, com análise técnica em cada célula |
| 16 | Resposta direta sobre existir opção vencedora absoluta | ✅ | 1.5, primeiro parágrafo ("Não", com justificativa) |
| 17 | Cenário A — múltiplas aplicações (web, mobile, API, administrativo) | ✅ | 1.5, Cenário A, com exemplo de divergência silenciosa e alternativa arquitetural |
| 18 | Cenário B — dados sensíveis / exigência legal ou fiscal | ✅ | 1.5, Cenário B, com dois argumentos técnicos e o limite do que o banco garante |
| 19 | Cenário C — regras que mudam com frequência | ✅ | 1.5, Cenário C, com exemplo de política de frete e a ressalva sobre invariantes |
| 20 | Cenário D — protótipo ou equipe pequena | ✅ | 1.5, Cenário D, com análise assimétrica de custo |
| 21 | Posição fundamentada ao final da seção 1.5 | ✅ | 1.5, subseção "Posição fundamentada" |
| 22 | Exemplo funcional em PostgreSQL com cliente, produto e pedido | ✅ | 2.1, esquema completo (mais `item_pedido`, necessária para o modelo fazer sentido) |
| 23 | Uso de `PK`, `FK`, `UNIQUE`, `NOT NULL`, `CHECK` e transação, com SQL correto e explicação de cada regra | ✅ | 2.1, esquema + tabela "Explicação regra a regra" + bloco transacional |
| 24 | Demonstração de que o CPF único é garantido pelo banco | ✅ | 2.1, subseção "CPF único: demonstração", com a saída real do erro |
| 25 | Regra de estoque com tratamento de concorrência explicado | ✅ | 2.1, subseção "Estoque e concorrência", com antipadrão, padrão correto e teste com duas sessões |
| 26 | *Triggers*/procedimentos apenas quando necessários, com justificativa | ✅ | 2.1: um *trigger* justificado por ser regra entre tabelas; o procedimento é apresentado com ressalva explícita de que **não** é a alocação recomendada |
| 27 | Exemplo de camada de serviço em pseudocódigo | ✅ | 2.2, com as quatro etapas (entrada, domínio, transação, tratamento de violações) |
| 28 | Indicação do que pertence à aplicação e do que permanece protegido no banco | ✅ | 2.2, tabela "O que pertence a cada camada" |
| 29 | Caso prático com pelo menos 3 regras (CPF, estoque, pedido em aberto) | ✅ | 2.3, com regra, camada principal, justificativa e proteção complementar em cada uma |
| 30 | Referências reais, confiáveis, priorizando documentação oficial, livros, artigos e material acadêmico | ✅ | Seção 3, organizada em cinco blocos, com links verificados |
| 31 | Associação entre afirmações técnicas e suas fontes | ✅ | Citações no corpo do texto (POSTGRESQL 2026a–h; HÄRDER; REUTER, 1983; PORTS; GRITTNER, 2012; FOWLER, 2002, 2004; EVANS, 2003; BUSINESS RULES GROUP, 2000) |
| 32 | Documentação do PostgreSQL da versão consultada | ✅ | Seção 3, nota de versão: documentação da v18 (corrente); exemplos executados em 16.14, informado em 2.1 |
| 33 | Conclusão respondendo às sete perguntas exigidas | ✅ | Seção 4, uma subseção por pergunta |
| 34 | Português brasileiro, linguagem acadêmica e clara | ✅ | Documento inteiro; termos técnicos explicados no primeiro uso |
| 35 | Conceitos explicados, não apenas citados | ✅ | Ex.: `CHECK` com suas três limitações; ACID com o "C" qualificado; MVCC explicado pelo comportamento observado |
| 36 | Distinção clara entre *constraint*, *trigger*, *stored procedure* e transação | ✅ | 1.2, quatro subseções separadas, com a diferença entre `FUNCTION` e `PROCEDURE` explicitada |
| 37 | ACID explicado corretamente | ✅ | 1.2, com a ressalva conceitual sobre consistência e demonstração prática da atomicidade em 2.1 |
| 38 | Não confundir regra de negócio com integridade de dados | ✅ | 1.1, tabela de três conceitos e exemplo do CPF |
| 39 | Não afirmar que toda regra de negócio deve estar no banco | ✅ | 1.1, 1.5 (Cenário C) e Seção 4, "riscos de tudo no banco" |
| 40 | Não afirmar que validação na aplicação substitui a integridade do banco | ✅ | 1.1 (parágrafo final), 1.3 (desvantagens) e 2.2 (linha "informativa, não garantia" da tabela) |
| 41 | Não inventar informações | ✅ | SQL executado em servidor real; referências verificadas; nenhuma fonte ou saída de comando foi inventada |
| 42 | Revisão técnica na perspectiva do professor, com os 10 pontos | ✅ | Apêndice A |
| 43 | Entrega em Markdown, pronto para salvar com o nome indicado | ✅ | Este arquivo: `ATIVIDADE-teorica-regra-negocio-grupo-XX-sobrenome-do-aluno.md` |
| 44 | Dados de identificação preenchidos | ✅ | Cabeçalho (alunos, turma, grupo, data, repositório) e seção "Link do Repositório Git" |
