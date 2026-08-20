# Apostila InterSystems ObjectScript Specialist
## Capítulo 12 — T3.5 Ensures Scalability and Performance (Escalabilidade e desempenho)

> Último tópico do domínio **T3 — IRIS Features**. Aqui você aprende a **medir** antes de mexer, a reconhecer os padrões que matam o desempenho, e a usar as ferramentas que o IRIS oferece para consertá-los.

---

## 1. O que você vai saber fazer ao terminar

1. **Medir** tempo e memória com `$ZHOROLOG`, `$ZTIMESTAMP`, `$STORAGE` e `$ZSTORAGE`.
2. Reconhecer e eliminar o **problema N+1**, o erro de desempenho mais comum em sistemas de objetos.
3. Ler um **plano de consulta** e entender quando um índice é usado e quando não é.
4. Saber por que uma **função aplicada à coluna** no `WHERE` impede o uso do índice.
5. Escolher entre índice **comum**, **bitmap**, **bitslice** e índice com **`Data =`** (índice de cobertura).
6. Coletar **estatísticas de tabela** para que o otimizador escolha bem.
7. Usar **dicas de otimizador** (`%PARALLEL`, `%NOINDEX`, `%IGNOREINDEX`, `%ALLINDEX`) com parcimônia.
8. Escrever laços eficientes sobre **globais**, evitando releituras e usando `MERGE`.
9. Entender o custo de **transações longas**, **travas** e **journaling**.
10. Processar em **lotes** e em **paralelo**, com noções do gerenciador de trabalho.
11. Conhecer os conceitos de **buffers globais**, **cache de consultas**, **ECP** e **sharding**.
12. Levar o projeto à versão **1.3**, com uma bancada de medição e otimizações reais.

---

## 2. O conceito em linguagem de gente

### 2.1 A regra número um: medir antes

Um programador experiente e um iniciante olham para o mesmo código lento. O iniciante diz: *"deve ser aquele laço"*. O experiente pergunta: *"quanto tempo cada parte está levando?"*.

A diferença não é conhecimento técnico. É método.

Otimizar sem medir tem três problemas:

- Você quase sempre otimiza a parte errada. A intuição sobre desempenho é notoriamente ruim.
- Você não sabe se melhorou. "Parece mais rápido" não é um dado.
- Você torna o código mais complicado em troca de nada — e complicação é dívida permanente.

**A ordem correta é sempre: medir → identificar o gargalo → corrigir → medir de novo.**

E há um corolário que economiza muito trabalho: **90% do tempo costuma estar em 10% do código.** Otimizar os outros 90% é esforço puro sem retorno.

### 2.2 O problema N+1: o erro que domina tudo

Este é o padrão que mais destrói desempenho em sistemas orientados a objetos, e você provavelmente já o escreveu sem perceber — aliás, você o escreveu nesta apostila, no Capítulo 3.

A situação: você quer listar 1000 pacientes com o nome do médico responsável de cada um.

**Jeito ingênuo:**

1. Uma consulta traz os 1000 pacientes.
2. Para cada paciente, você abre o médico dele com `%OpenId()`.

Total: **1 consulta + 1000 aberturas = 1001 idas ao banco.** Daí o nome N+1.

**Jeito correto:**

1. Uma consulta traz os 1000 pacientes **junto com** o nome do médico, usando junção.

Total: **1 ida ao banco.**

A analogia é literal e desconfortável: é a diferença entre pedir ao arquivista uma pilha com as mil fichas de uma vez, ou fazer mil viagens ao arquivo, uma por ficha, subindo e descendo a escada a cada vez.

E o traiçoeiro é que **em desenvolvimento ninguém percebe**. Com 5 registros de teste, 6 idas ao banco são instantâneas. Com 100 mil registros em produção, o sistema para.

**Sinal de alerta:** sempre que você vir `%OpenId()` — ou qualquer acesso ao banco — **dentro de um laço**, pare e pergunte se aquilo não podia ser uma consulta só.

### 2.3 Por que um índice às vezes não é usado

O índice é o fichário alfabético do Capítulo 2. Ele funciona porque os cartões estão **ordenados pelo valor da ficha**.

Agora imagine que você pede ao arquivista: *"me traga todos cujo nome, escrito ao contrário, comece com 'AIR'"*.

O fichário não serve para nada. Ele está ordenado pelos nomes normais, não pelos nomes invertidos. O arquivista precisa pegar **cada cartão**, inverter o nome mentalmente e conferir.

É exatamente isso que acontece quando você aplica uma **função à coluna** no `WHERE`:

```sql
-- O índice sobre Name NÃO pode ser usado
WHERE UPPER(Name) = 'MARIA'

-- O índice pode ser usado
WHERE Name = 'Maria'
```

A regra, que vale para qualquer banco de dados do mundo:

> **A coluna indexada tem que aparecer "nua" de um lado da comparação.** Se ela está dentro de uma função, de uma concatenação ou de uma conta, o índice deixa de servir.

E a solução, quando você realmente precisa da transformação: **guarde o valor já transformado**, numa propriedade calculada e indexada. Foi exatamente o que o IRIS fez sozinho ao normalizar os nomes em maiúsculas dentro do índice, como você viu no Capítulo 8.

### 2.4 O otimizador precisa saber o tamanho das coisas

Quando existem vários caminhos possíveis para uma consulta, o **otimizador** escolhe um. Para escolher bem, ele precisa de estatísticas: quantas linhas a tabela tem, quantos valores distintos cada coluna tem, quão seletivo é cada índice.

Sem estatísticas, ele chuta. E chuta com base em valores padrão que podem estar muito longe da realidade.

Analogia: um entregador que não sabe se a rua tem 5 ou 5000 casas não consegue decidir se vale a pena consultar a lista de moradores ou simplesmente bater de porta em porta.

Por isso, depois de carregar uma tabela com volume representativo, você deve **coletar as estatísticas**. É uma operação de manutenção, não de desenvolvimento — e é frequentemente esquecida.

### 2.5 Onde o tempo realmente vai

Vale ter na cabeça a ordem de grandeza das operações, porque ela orienta as decisões:

| Operação | Custo relativo |
|---|---|
| ler uma variável local | praticamente zero |
| ler um nó de global já em memória (buffer) | baixo |
| ler um nó de global que precisa vir do disco | **muito** mais alto |
| abrir um objeto (`%OpenId`) | vários nós, mais validação e construção |
| preparar uma consulta SQL nova | alto, mas uma vez só (fica em cache) |
| executar uma consulta preparada | proporcional às linhas percorridas |
| gravar (`%Save`) | leitura, escrita, índices, callbacks, triggers, journal |

Duas conclusões práticas:

- **Gravar é muito mais caro que ler.** Um laço que regrava tudo por precaução é caro por precaução.
- **O IRIS mantém um pool de buffers globais em memória.** Dados acessados com frequência ficam lá, e a leitura é rápida. Dados acessados uma vez só pagam o preço do disco. É por isso que a segunda execução de uma consulta costuma ser bem mais rápida que a primeira — e por que **medir só uma vez engana**.

### 2.6 Escalar: fazer caber mais

Desempenho é fazer a mesma coisa mais rápido. **Escalabilidade** é conseguir crescer sem que tudo desabe.

Três estratégias, em ordem de complexidade:

1. **Lotes** — em vez de processar um milhão de registros de uma vez, processe em blocos de mil, com progresso registrado. Mais lento no total, mas com uso de memória constante e possibilidade de retomar.
2. **Paralelismo** — dividir o trabalho entre vários processos. O IRIS tem um gerenciador de trabalho para isso, e o SQL tem execução paralela.
3. **Distribuição** — espalhar dados e processamento por várias máquinas. É onde entram o cache distribuído e o particionamento de tabelas.

E uma regra que vale para as três: **paralelizar código lento produz código lento em paralelo.** Otimize primeiro, distribua depois.

---

## 3. As ferramentas

### 3.1 Medindo tempo

```objectscript
set start = $ZHOROLOG
// ... o trabalho ...
set elapsed = $ZHOROLOG - start
write "levou ", $FNUMBER(elapsed, "", 4), " segundos", !
```

- **`$ZHOROLOG`** devolve os segundos decorridos desde o início do dia, **com fração**. É a ferramenta padrão para medir trechos de código.
- **`$FNUMBER(valor, "", 4)`** formata com 4 casas decimais.

Cuidado com a virada do dia: à meia-noite, `$ZHOROLOG` volta a zero e a subtração dá negativo. Para medições longas ou noturnas, use `$ZTIMESTAMP`, que traz data e hora em UTC com fração.

**Meça mais de uma vez.** A primeira execução paga o custo de trazer os dados do disco e de preparar a consulta; as seguintes rodam sobre dados já em memória. Reporte a **mediana de várias execuções**, ou reporte as duas coisas separadamente — "primeira execução" e "execuções seguintes" — porque as duas informações importam.

### 3.2 Medindo memória

```objectscript
write "memória disponível: ", $STORAGE, " bytes", !
write "limite do processo : ", $ZSTORAGE, " KB", !
```

- **`$STORAGE`** — quantos bytes de memória ainda restam para este processo.
- **`$ZSTORAGE`** — o limite máximo de memória do processo, em KB. Pode ser alterado.

Quando um processo estoura o limite, o erro é **`<STORE>`**. As causas típicas são acumular estruturas gigantes em variáveis locais, ou recursão descontrolada.

A defesa contra o primeiro caso você já conhece do Capítulo 8: **use uma global privada do processo em vez de variável local** quando o volume for grande. A PPG não consome memória do processo.

### 3.3 O plano de consulta

O plano é a explicação de **como** o IRIS pretende executar a consulta: quais índices usará, em que ordem juntará as tabelas, se fará varredura completa.

**Pelo Portal:** em **System Explorer → SQL**, escreva a consulta e clique em **Show Plan** em vez de executar.

**Por código:**

```objectscript
do $SYSTEM.SQL.Explain("SELECT Name FROM LabStudy.PATIENT WHERE RecordNumber = 'REG-000001'")
```

O que procurar no plano:

- Uma linha indicando **leitura por índice** (*read index map*) — bom sinal.
- Uma linha indicando **varredura do mapa mestre** (*read master map*) sobre a tabela inteira — sinal de que nenhum índice foi aproveitado.
- O **custo estimado** — útil para comparar duas versões da mesma consulta.

A forma exata do texto do plano varia entre versões: **verificar na documentação oficial** para a interpretação detalhada. O que importa aqui é o hábito: **antes de otimizar uma consulta, olhe o plano dela**.

### 3.4 Estatísticas de tabela

```objectscript
do $SYSTEM.SQL.Stats.Table.GatherTableStats("LabStudy.PATIENT")
```

Isso examina os dados e registra na definição da classe informações como o tamanho da extensão e a seletividade de cada campo indexado.

Em versões anteriores, a operação se chamava *Tune Table* e era invocada por outro método. Também é possível fazê-la pelo Portal, em **System Explorer → SQL → Actions → Tune Table**. Os nomes e assinaturas mudam entre versões: **verificar na documentação oficial**.

**Quando rodar:**

- Depois de carregar dados iniciais num sistema novo.
- Depois de um crescimento grande de volume.
- Depois de acrescentar índices.
- **Não** rode com a tabela vazia ou com dados de teste não representativos — estatísticas erradas são piores que estatísticas ausentes, porque o otimizador confia nelas.

Depois de mudar estatísticas ou esquema, **expurgue as consultas em cache**, para que os planos antigos não continuem sendo usados. Pelo Portal: **System Explorer → SQL → Actions → Purge Cached Queries**.

### 3.5 Escolhendo o tipo de índice

| Tipo | Quando usar |
|---|---|
| comum | busca por igualdade ou faixa em coluna com muitos valores distintos |
| `Type = bitmap` | coluna com **poucos** valores distintos, em tabela grande (sexo, status, sim/não) |
| `Type = bitslice` | coluna **numérica** usada em **agregações** (`SUM`, `AVG`) sobre muitas linhas |
| com `Data = (...)` | quando a consulta lê poucas colunas sempre as mesmas |

O último merece explicação, porque é o mais subutilizado.

**Índice de cobertura** — um índice que carrega, além da chave, cópias de outras colunas:

```objectscript
Index NameIdx On Name [ Data = (RecordNumber, Sex) ];
```

Agora uma consulta como esta:

```sql
SELECT Name, RecordNumber, Sex FROM LabStudy.PATIENT WHERE Name %STARTSWITH 'Mar'
```

pode ser respondida **lendo apenas o índice**, sem tocar nos dados principais. Voltando à analogia: o cartão do fichário passa a conter também o telefone, então o arquivista responde sem ir ao armário buscar a ficha.

Custo: espaço e escrita um pouco mais lenta. Benefício: leitura muito mais rápida numa consulta frequente. É uma troca deliberada.

E `bitslice` merece uma nota: ele armazena os números de forma que somas e médias sobre grandes conjuntos sejam calculadas muito rapidamente. É especializado; para busca por valor, não serve.

### 3.6 Dicas de otimizador

O IRIS permite sugerir (ou impor) escolhas ao otimizador:

```sql
SELECT COUNT(*) FROM LabStudy.EXAM %PARALLEL

SELECT * FROM LabStudy.PATIENT WHERE Name %STARTSWITH 'Mar' %IGNOREINDEX NameIdx

SELECT * FROM LabStudy.PATIENT %ALLINDEX WHERE Sex = 'F' AND Active = 1
```

- **`%PARALLEL`** — autoriza o IRIS a dividir a execução entre vários processos. Ótimo para agregações sobre tabelas grandes; **contraproducente** para consultas pequenas, porque o custo de coordenar supera o ganho.
- **`%NOINDEX`** aplicado a uma condição — instrui a não usar índice para aquela condição específica.
- **`%IGNOREINDEX`** — proíbe um índice determinado.
- **`%ALLINDEX`** — incentiva o uso de todos os índices aplicáveis.

**Use dicas com muita parcimônia.** Elas congelam uma decisão que o otimizador reavaliaria sozinho quando os dados mudassem. Uma dica que ajuda hoje pode atrapalhar daqui a um ano. A ordem correta é: primeiro colete estatísticas, depois reescreva a consulta, e só em último caso force com dica — documentando por quê.

A lista completa de dicas e sua sintaxe exata: **verificar na documentação oficial**.

### 3.7 Escrevendo laços eficientes sobre globais

**Não releia o mesmo nó várias vezes:**

```objectscript
// Ruim: três leituras da mesma global
if ^Config("timeout") > 0 {
    write ^Config("timeout"), !
    set total = total + ^Config("timeout")
}

// Bom: uma leitura
set timeout = $GET(^Config("timeout"))
if timeout > 0 {
    write timeout, !
    set total = total + timeout
}
```

**Use `MERGE` em vez de laço para copiar subárvores:**

```objectscript
merge ^||Cache = ^Config          // uma operação
```

**Prefira `$ORDER` a `$QUERY` quando você conhece a estrutura.** `$ORDER` anda num nível e é mais barato; `$QUERY` percorre toda a árvore e monta a referência completa a cada passo.

**Aproveite o terceiro argumento do `$ORDER`:**

```objectscript
// Duas operações por volta
set k = $ORDER(^X(k))
set v = ^X(k)

// Uma operação por volta
set k = $ORDER(^X(k), 1, v)
```

**Desenhe subscritos curtos em globais de alto volume**, como discutido no Capítulo 8. Nomes descritivos custam bytes por nó — multiplique por dez milhões de nós.

### 3.8 Custos de transação, trava e journal

Recapitulando o Capítulo 5 sob a ótica do desempenho:

- **Transação longa** acumula journal e mantém travas, criando fila. Mantenha-as curtas.
- **Trava disputada** transforma paralelismo em espera. Se dois processos brigam pelo mesmo recurso, acrescentar um terceiro não ajuda.
- **Journal** custa escrita. Dados descartáveis não precisam dele — daí as globais temporárias do Capítulo 8.
- **`$INCREMENT`** é mais rápido e mais seguro que travar, ler, somar e gravar.

E um ponto de escalabilidade: uma sequência global gerada por `$INCREMENT` é um **ponto único de contenção**. Com muitos processos gravando ao mesmo tempo, todos disputam o mesmo nó. Sistemas de altíssimo volume distribuem isso, por exemplo com sequências por processo ou por faixa.

### 3.9 Processamento em lotes

O padrão para volumes grandes:

```objectscript
ClassMethod ProcessBatches(batchSize As %Integer = 1000) As %Integer
{
    set lastId = $GET(^ProcessProgress("lastId"), 0)
    set totalDone = 0

    for {
        set rs = ##class(%SQL.Statement).%ExecDirect(,
            "SELECT TOP ? %ID AS Id FROM LabStudy.EXAM WHERE %ID > ? ORDER BY %ID",
            batchSize, lastId)

        set inThisBatch = 0

        tstart
        while rs.%Next() {
            set id = rs.%Get("Id")
            // ... trabalho sobre o registro ...
            set lastId = id
            set inThisBatch = inThisBatch + 1
        }
        set ^ProcessProgress("lastId") = lastId
        tcommit

        set totalDone = totalDone + inThisBatch
        quit:inThisBatch=0

        hang 0.01     // dá espaço aos outros processos
    }

    quit totalDone
}
```

Três características que fazem esse padrão funcionar:

- **Cada lote é uma transação própria.** Falha no lote 500 não desfaz os 499 anteriores.
- **O progresso é gravado**, então dá para retomar.
- **`HANG 0.01`** cede um instante do processador. Sem isso, um processo de lote pode monopolizar recursos e degradar o sistema inteiro para os usuários.

### 3.10 Paralelismo

Além do `%PARALLEL` do SQL, o IRIS oferece um **gerenciador de trabalho** que distribui tarefas entre processos:

```objectscript
set queue = $SYSTEM.WorkMgr.%New()

for i = 1:1:10 {
    set sc = queue.Queue("##class(LabStudy.Bench).ProcessChunk", i)
}

set sc = queue.WaitForComplete()
```

A ideia: você **enfileira** unidades de trabalho, e o gerenciador as distribui entre processos disponíveis, esperando todos terminarem.

Cuidados essenciais:

- Cada unidade precisa ser **independente**. Se elas disputam a mesma trava ou a mesma global, o paralelismo vira fila.
- Cada processo tem seu **próprio contexto**: variáveis locais e PPG **não** são compartilhadas.
- Erros dentro de uma unidade precisam ser coletados e relatados.

A assinatura exata dos métodos e as opções disponíveis: **verificar na documentação oficial**.

### 3.11 Conceitos de escala que a prova menciona

**Buffers globais** — a memória que o IRIS reserva para manter blocos de banco. Quanto maior, menos leituras de disco. É configuração de instância, não de código, mas explica por que a segunda execução é mais rápida.

**Cache de consultas** — consultas SQL preparadas ficam guardadas e reaproveitadas. É por isso que parametrizar com `?` importa também para desempenho: valores colados geram uma entrada de cache por valor.

**ECP (Enterprise Cache Protocol)** — permite que vários servidores de aplicação compartilhem os dados de um servidor de banco, mantendo cache local coerente. É como se cada filial tivesse uma cópia atualizada do arquivo central.

**Sharding** — particiona uma tabela grande entre vários nós, e a consulta é executada em paralelo em todos eles. É a estratégia para volumes que não cabem numa máquina.

Nenhum dos dois últimos se configura em código de aplicação; para a certificação, basta saber **o que resolvem**.

---

## 4. Exemplo comentado

Uma bancada de medição e a demonstração do N+1:

Arquivo `src/LabStudy/Demo/Perf.cls`:

```objectscript
/// Measurement helpers and performance demonstrations.
Class LabStudy.Demo.Perf Extends %RegisteredObject
{

/// How many rows the sample data set has.
Parameter SAMPLESIZE = 2000;

/// Runs a class method several times and reports the timings.
/// Returns the best (fastest) time in seconds.
ClassMethod Time(className As %String, methodName As %String, runs As %Integer = 3) As %Numeric
{
    set best = ""
    set total = 0

    for run = 1:1:runs {
        set start = $ZHOROLOG
        do $CLASSMETHOD(className, methodName)
        set elapsed = $ZHOROLOG - start

        set total = total + elapsed
        if (best = "") || (elapsed < best) {
            set best = elapsed
        }

        write "  run ", run, ": ", $FNUMBER(elapsed, "", 4), "s", !
    }

    write "  best: ", $FNUMBER(best, "", 4), "s   avg: ",
          $FNUMBER(total / runs, "", 4), "s", !
    quit best
}

/// Creates a data set big enough for the difference to show.
ClassMethod Seed() As %Status
{
    do ##class(LabStudy.Demo.PerfExam).%KillExtent()
    do ##class(LabStudy.Demo.PerfPatient).%KillExtent()

    set start = $ZHOROLOG

    // 200 patients
    for i = 1:1:200 {
        set p = ##class(LabStudy.Demo.PerfPatient).%New()
        set p.Name = "Patient "_$TRANSLATE($JUSTIFY(i, 5), " ", "0")
        set p.City = $CASE(i # 4, 0: "Potirendaba", 1: "Rio Preto", 2: "Bady Bassitt", : "Mirassol")
        do p.%Save()
        set patientId(i) = p.%Id()
    }

    // 2000 exams spread over them
    for i = 1:1:..#SAMPLESIZE {
        set e = ##class(LabStudy.Demo.PerfExam).%New()
        set e.PatientRef = ##class(LabStudy.Demo.PerfPatient).%OpenId(patientId((i # 200) + 1))
        set e.Code = $CASE(i # 5, 0: "HGB", 1: "GLU", 2: "CHOL", 3: "TRIG", : "UREA")
        set e.Value = (i # 300) + 10
        do e.%Save()
    }

    write "seeded in ", $FNUMBER($ZHOROLOG - start, "", 3), "s", !
    quit $$$OK
}

/// The N+1 antipattern: one query, then one %OpenId per row.
ClassMethod ReportNPlusOne() As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.Demo.PerfExam")

    set count = 0
    set sum = 0

    while rs.%Next() {
        set exam = ##class(LabStudy.Demo.PerfExam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        // touching the related object forces another read
        set city = exam.PatientRef.City

        set count = count + 1
        set sum = sum + exam.Value
    }

    quit count
}

/// The same information in a single query.
ClassMethod ReportSingleQuery() As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.Code, e.Value, e.PatientRef->City AS City "
        _"FROM LabStudy.Demo.PerfExam e")

    set count = 0
    set sum = 0

    while rs.%Next() {
        set city = rs.%Get("City")
        set count = count + 1
        set sum = sum + rs.%Get("Value")
    }

    quit count
}

/// Even better when only the aggregate is needed: let SQL do the work.
ClassMethod ReportAggregate() As %Integer
{
    new SQLCODE, %msg, total, sum

    &sql(SELECT COUNT(*), SUM(Value) INTO :total, :sum
         FROM LabStudy.Demo.PerfExam)

    quit +$GET(total)
}

/// Compares the three approaches.
ClassMethod Compare() As %Status
{
    write "=== N+1 (one %OpenId per row) ===", !
    do ..Time($CLASSNAME(), "ReportNPlusOne")

    write !, "=== single query with arrow join ===", !
    do ..Time($CLASSNAME(), "ReportSingleQuery")

    write !, "=== aggregate in SQL ===", !
    do ..Time($CLASSNAME(), "ReportAggregate")

    quit $$$OK
}

}
```

E as duas classes de apoio:

```objectscript
/// Sample patient for the performance demonstration.
Class LabStudy.Demo.PerfPatient Extends %Persistent
{
Property Name As %String(MAXLEN = 100);
Property City As %String(MAXLEN = 50);
Index NameIdx On Name;
}
```

```objectscript
/// Sample exam for the performance demonstration.
Class LabStudy.Demo.PerfExam Extends %Persistent
{
Property Code As %String(MAXLEN = 20);
Property Value As %Numeric;
Property PatientRef As LabStudy.Demo.PerfPatient;
Index CodeIdx On Code;
}
```

Comentando as decisões:

- **`Time` roda várias vezes e reporta o melhor e a média.** O melhor tempo é o mais representativo do custo real do algoritmo, porque elimina interferências momentâneas de outros processos. A média mostra a variação. Reportar apenas uma execução seria irresponsável.
- **`Time` recebe o nome do método e usa `$CLASSMETHOD`.** Assim a bancada serve para qualquer método sem argumentos, sem duplicação de código.
- **`Seed` guarda os IDs num array local** em vez de reconsultar. Numa rotina de carga, isso já é uma otimização.
- **`$CASE(expressão, valor: resultado, ...)`** escolhe entre alternativas conforme o valor. É primo do `$SELECT`, mas compara **um valor** contra várias opções, e o último ramo sem valor antes dos dois-pontos é o padrão.
- **`ReportNPlusOne` toca `exam.PatientRef.City` de propósito.** É esse acesso que dispara a leitura do objeto relacionado — o *swizzling* do Capítulo 2. Sem tocá-lo, o objeto do paciente nem seria carregado, e a demonstração perderia metade do efeito.
- **`ReportSingleQuery` usa a seta** para trazer a cidade na mesma consulta. Uma ida ao banco.
- **`ReportAggregate` nem percorre**: pede a resposta pronta.

### 4.1 Usando no Terminal

```
LABSTUDY>DO ##class(LabStudy.Demo.Perf).Seed()
seeded in 3.412s

LABSTUDY>DO ##class(LabStudy.Demo.Perf).Compare()
=== N+1 (one %OpenId per row) ===
  run 1: 1.8420s
  run 2: 1.6135s
  run 3: 1.5981s
  best: 1.5981s   avg: 1.6845s

=== single query with arrow join ===
  run 1: 0.1104s
  run 2: 0.0731s
  run 3: 0.0718s
  best: 0.0718s   avg: 0.0851s

=== aggregate in SQL ===
  run 1: 0.0089s
  run 2: 0.0021s
  run 3: 0.0019s
  best: 0.0019s   avg: 0.0043s
```

*(Os números da sua máquina serão diferentes. O que importa são as proporções.)*

O que observar:

- **A consulta única foi cerca de 22 vezes mais rápida** que o N+1, com o mesmo resultado. E note que são só 2000 registros. Com 200 mil, a proporção piora, porque o N+1 cresce com o número de idas ao banco enquanto a consulta única aproveita leituras sequenciais.
- **A agregação foi mais 37 vezes mais rápida** que a consulta única — porque nem percorre linha a linha no ObjectScript. Quando você só precisa do total, **peça o total**.
- **A primeira execução de cada bloco é sempre a mais lenta.** Os dados subiram para os buffers na primeira passagem. Isso é o comportamento normal e é exatamente por isso que se mede mais de uma vez.
- **O `Seed` levou 3,4 segundos para gravar 2200 objetos.** Compare com os 0,07 segundos para **ler** 2000. Gravar é caríssimo em comparação com ler — conforme a tabela da seção 2.5.

---

## 5. Variações e detalhes

### 5.1 O efeito de um índice, medido

Para ver o índice trabalhar, compare a mesma consulta com e sem ele:

```
LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT %ID FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'")
```

Com o índice `CodeIdx`, o plano mostra leitura por índice. Se você remover o índice, recompilar e repetir, o plano passa a mostrar varredura completa da tabela.

E a diferença de tempo aparece quando o volume cresce. Com 2000 linhas, uma varredura completa é rápida; com 2 milhões, é a diferença entre milissegundos e minutos.

**Uma armadilha:** com poucos dados, o otimizador pode escolher **não** usar o índice, porque varrer a tabela inteira é mais barato do que ler o índice e depois buscar as linhas. Isso é uma decisão **correta** dele. Testar desempenho com volume de brinquedo leva a conclusões erradas sobre índices.

### 5.2 Índices que não ajudam

Nem todo índice é útil. Um índice é **inútil ou prejudicial** quando:

- a coluna quase nunca aparece em filtro ou ordenação;
- a coluna tem valor único em quase toda linha e as buscas são por faixa ampla;
- a tabela é pequena;
- a coluna é atualizada com muita frequência (cada atualização mexe também no índice).

Índices custam espaço e tornam cada `%Save()` mais lento. Antes de criar um, pergunte: **qual consulta específica ele acelera?** Se não houver resposta, não crie.

### 5.3 `SELECT *` e colunas desnecessárias

```sql
-- Ruim
SELECT * FROM LabStudy.PATIENT WHERE Sex = 'F'

-- Bom
SELECT Name, RecordNumber FROM LabStudy.PATIENT WHERE Sex = 'F'
```

Motivos:

- Trazer colunas que ninguém usa custa leitura e transmissão.
- Um `SELECT *` **impede** o uso de índice de cobertura, porque o índice não tem todas as colunas.
- Se alguém acrescentar uma coluna de stream à tabela, o `SELECT *` passa a carregar o stream inteiro.

### 5.4 Cache de resultado: quando vale

Você construiu um cache em PPG no Capítulo 8. Vale a pena quando:

- o cálculo é caro;
- o resultado é reutilizado várias vezes na mesma sessão;
- os dados de origem **não mudam** durante a sessão, ou uma leve defasagem é aceitável.

E **não** vale quando:

- o dado muda o tempo todo e a defasagem é inaceitável;
- o cálculo é barato (você acrescenta complexidade para economizar microssegundos);
- o resultado é grande e usado uma vez só.

O erro clássico de cache não é de desempenho: é de **correção**. Um cache que devolve dados velhos é pior que nenhum cache.

### 5.5 Quando o SQL não é a resposta

Este capítulo elogiou muito o SQL, então vale o contraponto. Acesso direto por objeto ou por global é melhor quando:

- você quer **um** registro conhecido pelo ID (`%OpenId` é mais direto);
- você precisa aplicar **regras de negócio** com callbacks e validação;
- a estrutura é uma árvore de global que **não** é uma tabela;
- você já tem o objeto na mão.

O erro simétrico ao N+1 existe: rodar uma consulta SQL para buscar algo que você já tem carregado na memória.

### 5.6 Medir em produção, não só no laboratório

Duas ferramentas do IRIS que vale conhecer de nome:

- **Monitor de desempenho do Portal** — em **System Operation**, mostra atividade de processos, uso de buffers, taxa de leitura e escrita.
- **Análise de consultas** — o Portal registra estatísticas das consultas executadas, permitindo descobrir quais consomem mais tempo no sistema real.

A diferença entre otimizar no laboratório e otimizar com dados de produção é a diferença entre adivinhar e saber. Os nomes exatos das telas e das ferramentas de instrumentação variam por versão: **verificar na documentação oficial**.

---

## 6. Pegadinhas e erros comuns

**1) Otimizar sem medir.**
Você otimiza o lugar errado, não sabe se melhorou, e complica o código de graça.

**2) Medir uma vez só.**
A primeira execução paga disco e preparação de consulta. Meça várias e reporte o melhor.

**3) `%OpenId()` dentro de um laço.**
É o N+1. Substitua por uma consulta com junção.

**4) Aplicar função à coluna no `WHERE`.**
`UPPER(Name) = 'MARIA'` impede o uso do índice. Compare a coluna nua.

**5) Testar desempenho com cinco registros.**
Não revela nada. O otimizador se comporta diferente, e o N+1 fica invisível.

**6) Criar índice sem saber qual consulta ele acelera.**
Custa espaço e torna toda gravação mais lenta, em troca de nada.

**7) Esquecer de coletar estatísticas depois de carregar dados.**
O otimizador escolhe mal por falta de informação.

**8) Não expurgar consultas em cache depois de mudar esquema ou estatísticas.**
Planos antigos continuam sendo usados.

**9) Usar `%PARALLEL` em consulta pequena.**
O custo de coordenar os processos supera o ganho.

**10) Abusar de dicas de otimizador.**
Você congela uma decisão que deveria se adaptar ao crescimento dos dados.

**11) `SELECT *` por hábito.**
Impede índice de cobertura e pode arrastar streams.

**12) Transação envolvendo o processamento inteiro.**
Journal enorme, travas longas, e falha no fim desfaz tudo.

**13) Acumular estruturas gigantes em variáveis locais.**
Erro `<STORE>`. Use PPG.

**14) Reler a mesma global várias vezes na mesma expressão.**
Guarde numa variável local.

**15) Paralelizar antes de otimizar.**
Código lento em paralelo continua lento, e agora disputa recursos.

**16) Cache que devolve dado velho.**
Deixa de ser problema de desempenho e vira problema de correção.

**17) Rodar processo de lote sem ceder tempo ao sistema.**
Um `HANG` pequeno entre lotes evita degradar a experiência dos usuários.

---

## 7. MÃO NA MASSA

---

### Exercício 12.1 — Construindo a bancada de medição

**a) Enunciado:** Crie `LabStudy.Demo.Bench` com:

1. `ClassMethod Start()` — devolve um marcador de início.
2. `ClassMethod Stop(marker, label)` — imprime o rótulo e o tempo decorrido com 4 casas.
3. `ClassMethod Run(className, methodName, runs)` — executa um método várias vezes e reporta melhor, pior e média.
4. `ClassMethod Memory()` — imprime memória disponível e limite do processo.
5. `ClassMethod StressMemory(kb)` — acumula dados numa variável local até aproximar do limite, mostrando `$STORAGE` diminuir; depois faz o mesmo numa PPG e mostra que `$STORAGE` **não** diminui.

**b) Dica:** Para gerar texto de tamanho conhecido, `$TRANSLATE($JUSTIFY("", 1000), " ", "x")` produz mil caracteres.

**c) Como testar:** No item 5, a diferença entre variável local e PPG deve ficar evidente nos números.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Bench.cls`:

```objectscript
/// Simple measurement bench: time and memory.
Class LabStudy.Demo.Bench Extends %RegisteredObject
{

/// Returns a start marker.
ClassMethod Start() As %Numeric [ CodeMode = expression ]
{
$ZHOROLOG
}

/// Prints the elapsed time since the marker.
ClassMethod Stop(marker As %Numeric, label As %String = "elapsed") As %Numeric
{
    set elapsed = $ZHOROLOG - marker

    // guard against midnight rollover
    if elapsed < 0 {
        set elapsed = elapsed + 86400
    }

    write $JUSTIFY(label, 30), " : ", $FNUMBER(elapsed, "", 4), "s", !
    quit elapsed
}

/// Runs a method several times and reports best, worst and average.
ClassMethod Run(className As %String, methodName As %String, runs As %Integer = 5) As %Numeric
{
    set best = "", worst = "", total = 0

    write "-- ", className, ".", methodName, " (", runs, " runs) --", !

    for run = 1:1:runs {
        set marker = ..Start()
        do $CLASSMETHOD(className, methodName)
        set elapsed = $ZHOROLOG - marker

        set total = total + elapsed
        if (best = "") || (elapsed < best) { set best = elapsed }
        if (worst = "") || (elapsed > worst) { set worst = elapsed }

        write "   run ", run, ": ", $FNUMBER(elapsed, "", 4), "s", !
    }

    write "   best ", $FNUMBER(best, "", 4), "s | worst ", $FNUMBER(worst, "", 4),
          "s | avg ", $FNUMBER(total / runs, "", 4), "s", !
    quit best
}

/// Prints the memory situation of this process.
ClassMethod Memory() As %Status
{
    write "available ($STORAGE) : ", $STORAGE, " bytes", !
    write "limit    ($ZSTORAGE) : ", $ZSTORAGE, " KB", !
    quit $$$OK
}

/// Fills a local variable and then a PPG, comparing the memory impact.
ClassMethod StressMemory(chunks As %Integer = 200) As %Status
{
    set block = $TRANSLATE($JUSTIFY("", 1000), " ", "x")

    write "=== local variable ===", !
    kill localData
    write "before: ", $STORAGE, !
    for i = 1:1:chunks {
        set localData(i) = block
    }
    write "after : ", $STORAGE, !
    kill localData
    write "freed : ", $STORAGE, !

    write !, "=== process private global ===", !
    kill ^||ppgData
    write "before: ", $STORAGE, !
    for i = 1:1:chunks {
        set ^||ppgData(i) = block
    }
    write "after : ", $STORAGE, !
    kill ^||ppgData

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Bench).Memory()
available ($STORAGE) : 267726848 bytes
limit    ($ZSTORAGE) : 262144 KB

LABSTUDY>DO ##class(LabStudy.Demo.Bench).StressMemory(200)
=== local variable ===
before: 267726848
after : 267492104
freed : 267726120

=== process private global ===
before: 267726120
after : 267726120
```

*(Os números serão diferentes na sua máquina.)*

**Por que cada decisão:**

- **O tratamento da virada de meia-noite em `Stop`** parece exagero até a noite em que um processo de lote registra um tempo negativo e ninguém entende o relatório. Custou três linhas.
- **`Run` reporta melhor, pior e média.** O melhor tempo é a estimativa mais limpa do custo do algoritmo. O pior revela variabilidade — se ele for muito maior, há disputa por recursos, e isso é informação valiosa.
- **`Start` é `CodeMode = expression`** porque é literalmente uma expressão. Menos ruído.
- **O experimento de memória é a demonstração do que o Capítulo 8 afirmou.** A variável local consumiu memória do processo — `$STORAGE` caiu. A PPG, guardando exatamente o mesmo volume, **não consumiu nada** do processo, porque vive no banco temporário. Numa rotina que acumula muitos dados, essa é a diferença entre funcionar e receber `<STORE>`.
- **`kill localData` devolveu a memória**, mas note que não voltou exatamente ao valor original: há fragmentação e sobrecarga. Memória raramente volta redondinha.

---

### Exercício 12.2 — Matando o N+1

**a) Enunciado:** Usando as classes `PerfPatient` e `PerfExam` do exemplo (ou criando as suas), escreva **quatro** versões de um relatório que soma os valores de exames por cidade do paciente:

1. `Version1_NPlusOne()` — cursor sobre exames, `%OpenId` do exame, e acesso a `exam.PatientRef.City`.
2. `Version2_Join()` — uma consulta com a seta, somando no ObjectScript.
3. `Version3_GroupBy()` — uma consulta com `GROUP BY` fazendo a soma no banco.
4. `Version4_Cached()` — como a 3, mas guardando o resultado numa PPG e reaproveitando na segunda chamada.

Meça as quatro com a bancada e explique a ordem dos resultados.

**b) Dica:** Todas devem produzir **o mesmo resultado**. Verifique isso antes de comparar tempos — comparar velocidades de algoritmos que dão respostas diferentes não significa nada.

**c) Como testar:** A diferença entre a 1 e a 3 deve ser de uma ou duas ordens de grandeza.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Acrescente a `src/LabStudy/Demo/Perf.cls`:

```objectscript
/// Prints a total per city. Kept in one place so all versions agree.
ClassMethod ShowTotals(ByRef totals) As %Status [ Private ]
{
    set city = ""
    for {
        set city = $ORDER(totals(city))
        quit:city=""
        write "  ", $JUSTIFY(city, 15), " : ", totals(city), !
    }
    quit $$$OK
}

/// Version 1: the N+1 antipattern.
ClassMethod Version1NPlusOne(Output totals) As %Integer
{
    kill totals

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.Demo.PerfExam")

    set rows = 0
    while rs.%Next() {
        set exam = ##class(LabStudy.Demo.PerfExam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        set city = exam.PatientRef.City
        set totals(city) = $GET(totals(city), 0) + exam.Value
        set rows = rows + 1
    }

    quit rows
}

/// Version 2: one query with the arrow, summing in ObjectScript.
ClassMethod Version2Join(Output totals) As %Integer
{
    kill totals

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.Value, e.PatientRef->City AS City FROM LabStudy.Demo.PerfExam e")

    set rows = 0
    while rs.%Next() {
        set city = rs.%Get("City")
        set totals(city) = $GET(totals(city), 0) + rs.%Get("Value")
        set rows = rows + 1
    }

    quit rows
}

/// Version 3: let the database group and sum.
ClassMethod Version3GroupBy(Output totals) As %Integer
{
    kill totals

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT e.PatientRef->City AS City, SUM(e.Value) AS Total "
        _"FROM LabStudy.Demo.PerfExam e GROUP BY e.PatientRef->City")

    set rows = 0
    while rs.%Next() {
        set totals(rs.%Get("City")) = rs.%Get("Total")
        set rows = rows + 1
    }

    quit rows
}

/// Version 4: same as 3, but cached in a process private global.
ClassMethod Version4Cached(Output totals) As %Integer
{
    kill totals

    if $DATA(^||PerfTotals) {
        merge totals = ^||PerfTotals
        quit +$GET(^||PerfTotals("__rows"), 0)
    }

    set rows = ..Version3GroupBy(.totals)

    kill ^||PerfTotals
    merge ^||PerfTotals = totals
    set ^||PerfTotals("__rows") = rows

    quit rows
}

/// Confirms that all versions agree, then compares their speed.
ClassMethod CompareVersions() As %Status
{
    kill ^||PerfTotals

    do ..Version1NPlusOne(.t1)
    do ..Version2Join(.t2)
    do ..Version3GroupBy(.t3)

    write "=== results ===", !
    do ..ShowTotals(.t1)

    write !, "same as version 2? "
    write $SELECT(..SameTotals(.t1, .t2): "yes", 1: "NO"), !
    write "same as version 3? "
    write $SELECT(..SameTotals(.t1, .t3): "yes", 1: "NO"), !

    write !, "=== timings ===", !
    do ##class(LabStudy.Demo.Bench).Run($CLASSNAME(), "Bench1")
    do ##class(LabStudy.Demo.Bench).Run($CLASSNAME(), "Bench2")
    do ##class(LabStudy.Demo.Bench).Run($CLASSNAME(), "Bench3")

    kill ^||PerfTotals
    do ##class(LabStudy.Demo.Bench).Run($CLASSNAME(), "Bench4")

    quit $$$OK
}

/// Compares two total arrays.
ClassMethod SameTotals(ByRef a, ByRef b) As %Boolean [ Private ]
{
    set key = ""
    for {
        set key = $ORDER(a(key))
        quit:key=""
        if $GET(a(key)) '= $GET(b(key)) { quit 0 }
    }

    set key = ""
    for {
        set key = $ORDER(b(key))
        quit:key=""
        if $GET(a(key)) '= $GET(b(key)) { quit 0 }
    }

    quit 1
}

ClassMethod Bench1() { do ..Version1NPlusOne(.t) }
ClassMethod Bench2() { do ..Version2Join(.t) }
ClassMethod Bench3() { do ..Version3GroupBy(.t) }
ClassMethod Bench4() { do ..Version4Cached(.t) }

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Perf).CompareVersions()
=== results ===
     Bady Bassitt : 254120
         Mirassol : 253880
      Potirendaba : 254380
        Rio Preto : 253620

same as version 2? yes
same as version 3? yes

=== timings ===
-- LabStudy.Demo.Perf.Bench1 (5 runs) --
   run 1: 1.7902s
   ...
   best 1.5844s | worst 1.7902s | avg 1.6412s
-- LabStudy.Demo.Perf.Bench2 (5 runs) --
   best 0.0694s | worst 0.0982s | avg 0.0761s
-- LabStudy.Demo.Perf.Bench3 (5 runs) --
   best 0.0146s | worst 0.0301s | avg 0.0183s
-- LabStudy.Demo.Perf.Bench4 (5 runs) --
   run 1: 0.0159s
   run 2: 0.0004s
   run 3: 0.0003s
   best 0.0003s | worst 0.0159s | avg 0.0037s
```

**Por que cada resultado:**

- **A verificação de igualdade vem antes das medições, e isso não é decoração.** Um algoritmo mais rápido que dá resposta diferente não é uma otimização; é um bug. Essa deveria ser a primeira linha de qualquer comparação de desempenho.
- **A versão 1 é ~22 vezes mais lenta que a 2.** Cada linha custou uma abertura de objeto mais uma leitura do objeto relacionado. São 4000 idas ao banco onde bastava uma.
- **A versão 3 é ~5 vezes mais rápida que a 2**, embora ambas façam uma consulta só. A diferença: a 2 traz 2000 linhas para o ObjectScript e soma aqui; a 3 traz **4 linhas** já somadas. Transferir menos dados é uma otimização por si só.
- **A versão 4 mostra o padrão típico de cache**: a primeira execução paga o preço integral (0,0159s, igual à versão 3), e as seguintes custam quase nada (0,0003s). O `best` de 0,0003 é enganoso se reportado sozinho — por isso a bancada reporta também o `worst`.
- **E o alerta sobre o cache:** ele guarda os totais numa PPG que **não é invalidada quando um exame novo é gravado**. Se um exame for inserido depois da primeira chamada, a versão 4 continuará devolvendo o total antigo. Isso é aceitável num relatório de painel atualizado a cada minuto, e inaceitável numa tela de conferência financeira. **O cache mais rápido é o que responde errado.**
- **`__rows` guardado dentro da mesma PPG** com um nome que não colide com nenhuma cidade real. É um truque comum e legítimo; alternativa mais limpa seria usar um ramo separado.

---

### Exercício 12.3 — O índice em ação

**a) Enunciado:** Usando `LabStudy.Demo.PerfExam`:

1. Veja o plano da consulta `WHERE Code = 'HGB'` com o índice `CodeIdx` presente.
2. Meça o tempo dessa consulta.
3. Remova o índice, recompile, purgue as consultas em cache e repita os dois passos.
4. Recoloque o índice, reconstrua com `%BuildIndices()`, colete estatísticas e repita.
5. Agora compare `WHERE Code = 'HGB'` com `WHERE UPPER(Code) = 'HGB'`. Veja os dois planos.

**b) Dica:** Se a diferença de tempo for pequena, aumente o volume de dados. Com poucas linhas, a varredura completa é competitiva — e isso, por si, já é um aprendizado.

**c) Como testar:** No item 5, o plano da versão com `UPPER` deve indicar varredura completa.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

```
LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT %ID FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'")
   (o plano indica leitura pelo mapa de índice CodeIdx)

LABSTUDY>SET m=##class(LabStudy.Demo.Bench).Start()
LABSTUDY>SET rs=##class(%SQL.Statement).%ExecDirect(,"SELECT COUNT(*) AS N FROM LabStudy.Demo.PerfExam WHERE Code='HGB'")
LABSTUDY>DO rs.%Next()
LABSTUDY>WRITE rs.%Get("N"), " linhas", !
400 linhas
LABSTUDY>DO ##class(LabStudy.Demo.Bench).Stop(m, "com indice")
                    com indice : 0.0038s
```

Remova `Index CodeIdx On Code;` da classe, recompile, purgue as consultas em cache pelo Portal, e repita:

```
LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT %ID FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'")
   (o plano agora indica leitura do mapa mestre — varredura completa)

LABSTUDY>DO ##class(LabStudy.Demo.Bench).Stop(m, "sem indice")
                    sem indice : 0.0271s
```

Recoloque o índice e refaça o ciclo completo:

```
LABSTUDY>DO ##class(LabStudy.Demo.PerfExam).%BuildIndices()
LABSTUDY>DO $SYSTEM.SQL.Stats.Table.GatherTableStats("LabStudy.Demo.PerfExam")
```

E a comparação final:

```
LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT %ID FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'")
   (leitura por índice)

LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT %ID FROM LabStudy.Demo.PerfExam WHERE UPPER(Code) = 'HGB'")
   (varredura completa — o índice não serve)
```

**Por que cada resultado:**

- **A diferença de tempo aqui é de milissegundos**, porque 2000 linhas cabem inteiras nos buffers. Isso é honesto e importante: **com pouco volume, o índice quase não faz diferença.** Ele existe para quando a tabela tiver milhões de linhas — e aí a diferença é entre milissegundos e minutos.
- **Purgar as consultas em cache entre as medições é obrigatório.** Sem isso, o IRIS reutiliza o plano antigo, e você mede a consulta errada acreditando estar medindo a nova. Este é o erro de metodologia mais comum ao testar índices.
- **`UPPER(Code)` matou o índice**, exatamente como a seção 2.3 previu. E note: o resultado da consulta é o mesmo, os dados são os mesmos, e a única diferença é uma função aparentemente inofensiva em volta da coluna.
- **A solução, se você realmente precisasse da comparação insensível a maiúsculas**, seria guardar o valor já em maiúsculas numa propriedade `SqlComputed` e indexá-la — o padrão que o próprio IRIS usa internamente nos índices de texto, como você viu no Capítulo 8.
- **Coletar estatísticas depois de reconstruir o índice** completa o ciclo: o otimizador passa a saber a seletividade real da coluna.

---

### Exercício 12.4 — Índice de cobertura e processamento em lotes

**a) Enunciado:**

Parte A — índice de cobertura:
1. Meça a consulta `SELECT Code, Value FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'`.
2. Acrescente `Index CodeCover On Code [ Data = (Value) ];`, reconstrua, purgue o cache e meça de novo.
3. Compare os planos.

Parte B — lotes:
4. Escreva `ClassMethod BulkUpdate(batchSize)` que percorre todos os exames e arredonda `Value` para o inteiro mais próximo, processando em lotes, com transação por lote e progresso gravado.
5. Compare o tempo dela com uma versão que faz tudo numa única transação.
6. Interrompa a versão em lotes no meio (`Ctrl+C`) e rode de novo. Ela deve retomar de onde parou.

**b) Dica:** Para o progresso, guarde o último ID processado numa global.

**c) Como testar:** No item 6, a segunda execução deve processar menos registros que a primeira.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

Parte A: acrescente à classe `PerfExam`:

```objectscript
/// Covering index: answers queries that need only Code and Value.
Index CodeCover On Code [ Data = (Value) ];
```

```
LABSTUDY>DO ##class(LabStudy.Demo.PerfExam).%BuildIndices($LISTBUILD("CodeCover"))
LABSTUDY>DO $SYSTEM.SQL.Stats.Table.GatherTableStats("LabStudy.Demo.PerfExam")
   (purgue as consultas em cache pelo Portal)

LABSTUDY>DO $SYSTEM.SQL.Explain("SELECT Code, Value FROM LabStudy.Demo.PerfExam WHERE Code = 'HGB'")
   (o plano agora lê apenas o mapa do índice CodeCover, sem tocar no mapa mestre)
```

Parte B, acrescente a `src/LabStudy/Demo/Perf.cls`:

```objectscript
/// Rounds every Value, one batch per transaction, recording progress.
ClassMethod BulkUpdateBatched(batchSize As %Integer = 200) As %Integer
{
    set lastId = $GET(^PerfProgress("lastId"), 0)
    set done = 0

    write "resuming from id ", lastId, !

    for {
        set rs = ##class(%SQL.Statement).%ExecDirect(,
            "SELECT TOP ? %ID AS Id FROM LabStudy.Demo.PerfExam "
            _"WHERE %ID > ? ORDER BY %ID", batchSize, lastId)

        set inBatch = 0
        set entryLevel = $TLEVEL
        tstart

        while rs.%Next() {
            set id = rs.%Get("Id")
            set exam = ##class(LabStudy.Demo.PerfExam).%OpenId(id)

            if $ISOBJECT(exam) {
                set exam.Value = $NUMBER(exam.Value, 0)
                do exam.%Save()
            }

            set lastId = id
            set inBatch = inBatch + 1
        }

        set ^PerfProgress("lastId") = lastId
        tcommit

        set done = done + inBatch
        quit:inBatch=0

        hang 0.01
    }

    write "processed ", done, " rows", !
    quit done
}

/// The same work in a single transaction, for comparison.
ClassMethod BulkUpdateSingleTx() As %Integer
{
    set entryLevel = $TLEVEL
    tstart

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.Demo.PerfExam ORDER BY %ID")

    set done = 0
    while rs.%Next() {
        set exam = ##class(LabStudy.Demo.PerfExam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        set exam.Value = $NUMBER(exam.Value, 0)
        do exam.%Save()
        set done = done + 1
    }

    tcommit
    write "processed ", done, " rows in one transaction", !
    quit done
}

/// Clears the progress marker.
ClassMethod ResetProgress() As %Status
{
    kill ^PerfProgress
    write "progress reset", !
    quit $$$OK
}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Perf).ResetProgress()

LABSTUDY>SET m=##class(LabStudy.Demo.Bench).Start()
LABSTUDY>DO ##class(LabStudy.Demo.Perf).BulkUpdateBatched(200)
resuming from id 0
processed 2000 rows
LABSTUDY>DO ##class(LabStudy.Demo.Bench).Stop(m, "em lotes")
                      em lotes : 3.8210s

LABSTUDY>SET m=##class(LabStudy.Demo.Bench).Start()
LABSTUDY>DO ##class(LabStudy.Demo.Perf).BulkUpdateSingleTx()
processed 2000 rows in one transaction
LABSTUDY>DO ##class(LabStudy.Demo.Bench).Stop(m, "transacao unica")
               transacao unica : 3.4055s

LABSTUDY>DO ##class(LabStudy.Demo.Perf).ResetProgress()
LABSTUDY>DO ##class(LabStudy.Demo.Perf).BulkUpdateBatched(200)
   (interrompa com Ctrl+C depois de alguns segundos)

LABSTUDY>WRITE ^PerfProgress("lastId"), !
843

LABSTUDY>DO ##class(LabStudy.Demo.Perf).BulkUpdateBatched(200)
resuming from id 843
processed 1157 rows
```

**Por que cada resultado:**

- **O índice de cobertura eliminou o acesso ao mapa mestre.** Como o índice já carrega `Value`, a consulta se resolve inteiramente dentro dele. O ganho aparece de verdade quando a tabela tem muitas colunas e a consulta precisa de duas.
- **A versão em lotes foi ligeiramente MAIS LENTA que a de transação única** — 3,82 contra 3,41 segundos. E isso está correto e é o ponto do exercício: **lotes não são uma otimização de velocidade.** Eles custam um pouco mais (mais transações, mais consultas, o `HANG`) e compram três coisas que a velocidade não compra: uso de journal constante, travas de curta duração, e **capacidade de retomar**.
- **A prova disso é a última parte.** A execução interrompida deixou o marcador em 843, e a execução seguinte processou apenas os 1157 restantes. A versão de transação única, interrompida, teria desfeito **tudo**, e você recomeçaria do zero — perdendo o dobro do tempo que os lotes custaram a mais.
- **`$NUMBER(valor, 0)`** arredonda para zero casas decimais. Funções numéricas são o Capítulo 15.
- **O `HANG 0.01` entre lotes** é o que impede esse processo de degradar o sistema para os usuários. Num processo noturno, pode ser desnecessário; num processo que roda em horário comercial, é essencial.

---

### Exercício 12.5 — PROJETO CONTÍNUO: otimizando o laboratório

**a) Enunciado:** Aplique tudo ao sistema real:

1. Crie `LabStudy.Bench` — a bancada de medição do exercício 12.1, promovida a classe do projeto.
2. Crie `LabStudy.Demo.LoadTest` com `ClassMethod Generate(patients, examsPerPatient)` que popula o sistema com volume realista, medindo e reportando o tempo.
3. Em `LabStudy.Reports`, acrescente `ClassMethod Benchmark()` que mede as consultas principais do painel e reporta cada uma.
4. Identifique e corrija o N+1 que ainda existe no projeto: o método `Show` de `LabStudy.Patient` percorre `patient.Exams` abrindo cada exame. Crie `ClassMethod ShowFast(id)` que traz tudo em duas consultas, e compare.
5. Acrescente a `LabStudy.Exam` um índice de cobertura adequado ao painel, reconstrua e colete estatísticas.
6. Crie `ClassMethod Optimize()` em `LabStudy.Schema` que reconstrói índices e coleta estatísticas de todas as classes do sistema.
7. Suba `LabStudy.App` para `"1.3"` e faça `Run()` reportar também o resultado do `Benchmark()`.

**b) Dica:** No item 4, a segunda consulta traz todos os exames do paciente de uma vez, em vez de um por um.

**c) Como testar:** Com 200 pacientes e 10 exames cada, a diferença entre `Show` e `ShowFast` deve ser mensurável.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Bench.cls` — igual ao exercício 12.1, apenas renomeada para o pacote do projeto.

`src/LabStudy/Demo/LoadTest.cls`:

```objectscript
/// Generates a realistic volume of data for performance work.
Class LabStudy.Demo.LoadTest Extends %RegisteredObject
{

Parameter CODES = "HGB,GLU,CHOL,TRIG,UREA,CREA,TGO,TGP";

/// Wipes everything and generates a fresh data set.
ClassMethod Generate(patients As %Integer = 200, examsPerPatient As %Integer = 10) As %Status
{
    write "clearing...", !
    do ##class(LabStudy.Exam).%KillExtent()
    do ##class(LabStudy.Patient).%KillExtent()
    do ##class(LabStudy.AuditEntry).%KillExtent()
    kill ^LabStudySeq

    set codeCount = $LENGTH(..#CODES, ",")

    set marker = ##class(LabStudy.Bench).Start()

    for i = 1:1:patients {
        set p = ##class(LabStudy.Patient).%New()
        set p.Name = "Paciente "_$TRANSLATE($JUSTIFY(i, 5), " ", "0")
        set p.BirthDate = +$HOROLOG - (7000 + (i # 15000))
        set p.RecordNumber = ##class(LabStudy.Sequence).NewRecordNumber()
        set p.Sex = $SELECT(i # 2: "M", 1: "F")
        set p.Address.City = $CASE(i # 4, 0: "Potirendaba", 1: "Rio Preto",
                                          2: "Bady Bassitt", : "Mirassol")

        set sc = p.%Save()
        continue:$$$ISERR(sc)

        for j = 1:1:examsPerPatient {
            set e = ##class(LabStudy.Exam).%New($PIECE(..#CODES, ",", ((i + j) # codeCount) + 1))
            set e.Patient = p
            set e.Unit = "mg/dL"
            do e.SetResult(((i * j) # 400) + 5)
            do e.%Save()
        }

        if i # 50 = 0 {
            write "  ", i, " patients...", !
        }
    }

    do ##class(LabStudy.Bench).Stop(marker, "generation")

    write "generated ", patients, " patients and ", patients * examsPerPatient, " exams", !
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Same output as Show(), but without opening one object per exam.
ClassMethod ShowFast(id As %String) As %Status
{
    new SQLCODE, %msg

    set stmt = ##class(%SQL.Statement).%New()
    set stmt.%SelectMode = 1

    set sc = stmt.%Prepare(
        "SELECT Name, RecordNumber, BirthDate, Sex, Address_City, CreatedOn "
        _"FROM LabStudy.PATIENT WHERE %ID = ?")
    if $$$ISERR(sc) { quit sc }

    set rs = stmt.%Execute(id)

    if 'rs.%Next() {
        write "Patient not found: ", id, !
        quit $$$OK
    }

    write "------------------------------", !
    write "Id:      ", id, !
    write "Name:    ", rs.%Get("Name"), !
    write "Record:  ", rs.%Get("RecordNumber"), !
    write "Birth:   ", rs.%Get("BirthDate"), !
    write "Sex:     ", rs.%Get("Sex"), !
    write "City:    ", rs.%Get("Address_City"), !

    // Second query: every exam of this patient, at once.
    set stmt2 = ##class(%SQL.Statement).%New()
    set sc = stmt2.%Prepare(
        "SELECT TestCode, ResultValue, Unit, ResultStatus "
        _"FROM LabStudy.EXAM WHERE Patient = ? ORDER BY TestCode")
    if $$$ISERR(sc) { quit sc }

    set rs2 = stmt2.%Execute(id)

    set n = 0
    set buffer = ""
    while rs2.%Next() {
        set n = n + 1
        set text = $CASE(rs2.%Get("ResultStatus"),
                         "pending":   "(pendente)",
                         "cancelled": "(cancelado)",
                         : rs2.%Get("ResultValue")_" "_rs2.%Get("Unit"))
        set buffer = buffer_"  - "_rs2.%Get("TestCode")_": "_text_$CHAR(13,10)
    }

    write "Exams (", n, "):", !
    write buffer
    write "------------------------------", !
    quit $$$OK
}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Covering index for the dashboard: answers without touching the main data.
Index PatientCover On Patient [ Data = (TestCode, ResultValue, Unit, ResultStatus) ];
```

Acrescente a `src/LabStudy/Reports.cls`:

```objectscript
/// Measures the main queries of the system.
ClassMethod Benchmark() As %Status
{
    write "=== benchmark ===", !

    set m = ##class(LabStudy.Bench).Start()
    do ##class(LabStudy.Patient).Statistics(.p, .e)
    do ##class(LabStudy.Bench).Stop(m, "Statistics")

    set m = ##class(LabStudy.Bench).Start()
    set rs = ..PatientListFunc(1)
    while rs.%Next() { }
    do ##class(LabStudy.Bench).Stop(m, "PatientList")

    set m = ##class(LabStudy.Bench).Start()
    set rs = ..AbnormalResultsFunc(300)
    while rs.%Next() { }
    do ##class(LabStudy.Bench).Stop(m, "AbnormalResults")

    set m = ##class(LabStudy.Bench).Start()
    do ##class(LabStudy.Patient).Show(1)
    do ##class(LabStudy.Bench).Stop(m, "Show (objects, N+1)")

    set m = ##class(LabStudy.Bench).Start()
    do ##class(LabStudy.Patient).ShowFast(1)
    do ##class(LabStudy.Bench).Stop(m, "ShowFast (2 queries)")

    quit $$$OK
}
```

Acrescente a `src/LabStudy/Schema.cls`:

```objectscript
/// Rebuilds indices and gathers statistics for every class of the system.
ClassMethod Optimize() As %Status
{
    set classes = $LISTBUILD("LabStudy.Patient", "LabStudy.Exam",
                             "LabStudy.AuditEntry", "LabStudy.User")

    for i = 1:1:$LISTLENGTH(classes) {
        set class = $LIST(classes, i)

        write "-- ", class, " --", !

        set m = ##class(LabStudy.Bench).Start()
        set sc = $CLASSMETHOD(class, "%BuildIndices")
        do ##class(LabStudy.Bench).Stop(m, "  build indices")

        if $$$ISERR(sc) {
            do $SYSTEM.Status.DisplayError(sc)
        }

        set m = ##class(LabStudy.Bench).Start()
        do $SYSTEM.SQL.Stats.Table.GatherTableStats(class)
        do ##class(LabStudy.Bench).Stop(m, "  gather stats")
    }

    write !, "remember to purge cached queries in the Portal", !
    quit $$$OK
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.3";

ClassMethod Run() As %Status
{
    do ..About()
    write !
    do ##class(LabStudy.Schema).Report()
    write !
    do ##class(LabStudy.Reports).Dashboard()
    write !
    do ##class(LabStudy.Reports).Benchmark()
    quit $$$OK
}
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Demo.LoadTest).Generate(200, 10)
clearing...
  50 patients...
  100 patients...
  150 patients...
  200 patients...
                    generation : 14.8021s
generated 200 patients and 2000 exams

LABSTUDY>DO ##class(LabStudy.Schema).Optimize()
-- LabStudy.Patient --
                 build indices : 0.1204s
                  gather stats : 0.3891s
-- LabStudy.Exam --
                 build indices : 0.4432s
                  gather stats : 0.5117s
...

LABSTUDY>DO ##class(LabStudy.Reports).Benchmark()
=== benchmark ===
                    Statistics : 0.0064s
                   PatientList : 0.0912s
               AbnormalResults : 0.0208s
            Show (objects, N+1) : 0.0114s
           ShowFast (2 queries) : 0.0021s
```

**Por que cada decisão:**

- **A geração de 2200 objetos levou quase 15 segundos.** Isso não é lentidão: é o custo real de `%Save()` com validação, índices, callbacks, triggers de auditoria e journal. Ele confirma a tabela da seção 2.5 no seu próprio sistema, com as suas próprias regras. Guarde esse número: ele é o teto de quantos registros o sistema consegue absorver por segundo por processo.
- **`Show` contra `ShowFast`: 5 vezes mais rápido** para um paciente com 10 exames. Parece pouco em valor absoluto — 11 milissegundos contra 2. Mas essa é a tela mais usada do sistema, e a proporção **cresce com o número de exames**: com 100 exames por paciente, a diferença seria muito maior, porque `Show` abre um objeto por exame e `ShowFast` continua fazendo duas consultas.
- **`ShowFast` acumula a saída num buffer** e escreve de uma vez. Escrever no dispositivo dentro do laço, entre duas leituras do cursor, prende o cursor aberto mais tempo do que o necessário. É um detalhe pequeno e um bom hábito.
- **O índice de cobertura em `Exam`** foi escolhido olhando para a consulta real: o painel e o `ShowFast` filtram por `Patient` e leem `TestCode`, `ResultValue`, `Unit` e `ResultStatus`. Exatamente essas colunas foram para o `Data`. **Índice de cobertura só faz sentido quando desenhado a partir de uma consulta concreta** — criar um "por precaução" é desperdício garantido.
- **`Optimize()` mede cada etapa.** Numa base de produção, saber que reconstruir os índices de `Exam` leva 40 minutos é o que permite planejar a janela de manutenção.
- **`Optimize()` termina lembrando de purgar o cache de consultas**, porque essa etapa não é automática e é a mais esquecida — como você viu no exercício 12.3.
- **`Benchmark()` entrou no `Run()`.** Um sistema que reporta os próprios tempos a cada execução torna a degradação visível **antes** de o usuário reclamar. Se `PatientList` estava em 0,09s no mês passado e está em 2,3s hoje, alguém precisa olhar.
- **`Show` continua existindo, não foi apagado.** Ele passa pelos objetos, portanto respeita `ResultText()`, callbacks e qualquer regra futura. `ShowFast` é uma otimização de leitura que **duplica a lógica de exibição** — e essa duplicação é a dívida que a otimização cobrou. Reconhecer o preço de cada otimização, e não fingir que ela foi de graça, é o que separa engenharia de esperteza.

---

## 8. Quiz do capítulo

**Q1.** Qual é a primeira coisa a fazer ao otimizar um código lento?

- A) Acrescentar índices.
- B) Medir para descobrir onde o tempo é gasto.
- C) Paralelizar o processamento.
- D) Aumentar a memória do processo.

---

**Q2.** Qual função devolve os segundos decorridos, com fração, para medir trechos de código?

- A) `$HOROLOG`
- B) `$ZHOROLOG`
- C) `$ZTIMESTAMP`
- D) `$JOB`

---

**Q3.** O que é o problema N+1?

- A) Um erro de índice fora de faixa.
- B) Uma consulta que traz N registros, seguida de N acessos individuais ao banco, um por registro.
- C) Uma transação com N níveis aninhados.
- D) Um laço que executa uma iteração a mais.

---

**Q4.** Qual é o sinal mais claro de um provável N+1?

- A) Um `SELECT` com muitas colunas.
- B) Um `%OpenId()` ou consulta dentro de um laço.
- C) Um `GROUP BY`.
- D) Uma transação longa.

---

**Q5.** Por que `WHERE UPPER(Name) = 'MARIA'` não usa o índice sobre `Name`?

- A) Porque `UPPER` não existe no IRIS.
- B) Porque o índice está ordenado pelos valores da coluna, não pelos valores transformados.
- C) Porque índices não funcionam com texto.
- D) Porque falta coletar estatísticas.

---

**Q6.** Para que serve coletar estatísticas de tabela?

- A) Para popular índices vazios.
- B) Para que o otimizador conheça tamanho e seletividade e escolha o melhor plano.
- C) Para compactar os dados.
- D) Para validar as linhas existentes.

---

**Q7.** Depois de mudar o esquema ou coletar estatísticas, o que também deve ser feito?

- A) Reiniciar a instância.
- B) Expurgar as consultas em cache, para que planos antigos não continuem sendo usados.
- C) Rodar `%KillExtent()`.
- D) Nada.

---

**Q8.** O que é um índice de cobertura?

- A) Um índice sobre todas as colunas da tabela.
- B) Um índice que carrega cópias de outras colunas com `Data = (...)`, permitindo responder consultas sem ler os dados principais.
- C) Um índice bitmap.
- D) Um índice único.

---

**Q9.** Quando um índice `Type = bitmap` é adequado?

- A) Em colunas com muitos valores distintos.
- B) Em colunas com poucos valores distintos, em tabelas grandes.
- C) Em qualquer coluna, sempre.
- D) Apenas em colunas de data.

---

**Q10.** O que faz a dica `%PARALLEL` numa consulta?

- A) Torna qualquer consulta mais rápida.
- B) Autoriza o IRIS a dividir a execução entre vários processos — útil em agregações grandes, contraproducente em consultas pequenas.
- C) Executa a consulta duas vezes para conferir.
- D) Cria um índice automaticamente.

---

**Q11.** Você acumula um grande volume de dados numa variável local e recebe `<STORE>`. Qual é a solução mais adequada?

- A) Reiniciar o processo.
- B) Usar uma global privada do processo (`^||`), que não consome memória do processo.
- C) Usar `KILL` mais vezes.
- D) Converter os dados para JSON.

---

**Q12.** Qual é a principal vantagem de processar em lotes em vez de uma transação única?

- A) É mais rápido no total.
- B) Usa journal de forma constante, mantém travas curtas e permite **retomar** após uma falha.
- C) Dispensa índices.
- D) Elimina a necessidade de transações.

---

**Q13.** Por que se deve medir mais de uma vez?

- A) Para gastar mais tempo.
- B) Porque a primeira execução paga o custo de trazer dados do disco e preparar a consulta.
- C) Porque `$ZHOROLOG` é impreciso.
- D) Porque o resultado muda a cada execução aleatoriamente.

---

**Q14.** Antes de comparar o tempo de dois algoritmos, o que é obrigatório verificar?

- A) Que ambos usam índices.
- B) Que ambos produzem exatamente o mesmo resultado.
- C) Que ambos estão na mesma classe.
- D) Que ambos usam SQL.

---

**Q15.** Qual afirmação sobre `SELECT *` está correta?

- A) É mais rápido que listar colunas.
- B) Traz colunas desnecessárias, impede o uso de índice de cobertura e pode arrastar streams.
- C) É obrigatório em consultas dinâmicas.
- D) Não tem impacto de desempenho.

---

**Q16.** O que o ECP (Enterprise Cache Protocol) resolve?

- A) Particionar uma tabela entre vários nós.
- B) Permitir que vários servidores de aplicação compartilhem os dados de um servidor de banco, com cache local coerente.
- C) Comprimir globais.
- D) Substituir o journal.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** medir primeiro é o método. Sem medição você otimiza no escuro.
- **A, C e D estão erradas:** são possíveis soluções, aplicadas **depois** de descobrir o gargalo. Aplicá-las antes é adivinhação.

**Q2 — Resposta: B.**
- **B está certa:** `$ZHOROLOG` devolve segundos com fração desde o início do dia.
- **A está errada:** `$HOROLOG` tem resolução de um segundo.
- **C está errada:** `$ZTIMESTAMP` serve, mas é data e hora em UTC — útil para medições longas, não é a resposta padrão.
- **D está errada:** `$JOB` é o número do processo.

**Q3 — Resposta: B.**
- **B está certa:** uma consulta traz N linhas e o código faz N acessos individuais — 1+N idas ao banco.
- **A, C e D estão erradas:** descrevem outras coisas.

**Q4 — Resposta: B.**
- **B está certa:** acesso ao banco dentro de laço é a assinatura do padrão.
- **A está errada:** muitas colunas é outro problema, menor.
- **C está errada:** `GROUP BY` é justamente uma das soluções.
- **D está errada:** transação longa é outro problema.

**Q5 — Resposta: B.**
- **B está certa:** o índice guarda os valores da coluna; um valor transformado não corresponde à ordenação do índice, então é preciso varrer e transformar linha a linha.
- **A está errada:** `UPPER` existe.
- **C está errada:** índices funcionam muito bem com texto.
- **D está errada:** estatísticas não mudam essa limitação estrutural.

**Q6 — Resposta: B.**
- **B está certa:** o otimizador precisa saber tamanhos e seletividades para escolher o plano.
- **A está errada:** popular índices é `%BuildIndices()`.
- **C está errada:** não há compactação envolvida.
- **D está errada:** validação é `%ValidateObject()`.

**Q7 — Resposta: B.**
- **B está certa:** planos antigos ficam em cache e continuariam sendo usados.
- **A está errada:** reiniciar não é necessário.
- **C está errada:** isso apagaria os dados.
- **D está errada:** é justamente o passo esquecido com mais frequência.

**Q8 — Resposta: B.**
- **B está certa:** com `Data = (...)`, o índice carrega colunas extras e responde sem tocar nos dados principais.
- **A está errada:** cobrir tudo seria duplicar a tabela.
- **C e D estão erradas:** são outros tipos de índice, com outra finalidade.

**Q9 — Resposta: B.**
- **B está certa:** bitmap é para baixa cardinalidade em conjuntos grandes.
- **A está errada:** com muitos valores distintos ele consome espaço sem entregar ganho.
- **C está errada:** não existe tipo universalmente melhor.
- **D está errada:** datas têm alta cardinalidade — caso ruim para bitmap.

**Q10 — Resposta: B.**
- **B está certa:** paraleliza a execução; o custo de coordenação só compensa em volume alto.
- **A está errada:** em consultas pequenas ele piora.
- **C e D estão erradas:** não é o que a dica faz.

**Q11 — Resposta: B.**
- **B está certa:** a PPG vive no banco temporário e não consome memória do processo.
- **A está errada:** resolve momentaneamente e o erro volta.
- **C está errada:** não ajuda se o volume for necessário.
- **D está errada:** JSON em memória consome memória igualmente.

**Q12 — Resposta: B.**
- **B está certa:** lotes trocam um pouco de velocidade por robustez e por uso previsível de recursos.
- **A está errada:** em geral são **um pouco mais lentos** no total.
- **C e D estão erradas:** índices e transações continuam necessários.

**Q13 — Resposta: B.**
- **B está certa:** a primeira execução carrega buffers e prepara a consulta; as seguintes refletem melhor o custo do algoritmo.
- **A está errada:** é método, não desperdício.
- **C está errada:** `$ZHOROLOG` é adequado para isso.
- **D está errada:** a variação tem causas identificáveis.

**Q14 — Resposta: B.**
- **B está certa:** comparar velocidades de algoritmos que dão respostas diferentes não significa nada.
- **A, C e D estão erradas:** são irrelevantes para a validade da comparação.

**Q15 — Resposta: B.**
- **B está certa:** custa leitura e transmissão desnecessárias e impede o índice de cobertura.
- **A está errada:** é mais lento, não mais rápido.
- **C está errada:** consultas dinâmicas podem listar colunas normalmente.
- **D está errada:** o impacto existe e cresce com o tamanho da tabela.

**Q16 — Resposta: B.**
- **B está certa:** ECP distribui o acesso mantendo cache local coerente entre servidores.
- **A está errada:** isso é sharding.
- **C e D estão erradas:** não têm relação com o ECP.

---

## 9. Resumo relâmpago

1. **Medir → identificar → corrigir → medir de novo.** Otimizar sem medir é adivinhar.
2. **`$ZHOROLOG`** para medir trechos (segundos com fração); `$ZTIMESTAMP` para medições longas; cuidado com a virada da meia-noite.
3. **Meça várias vezes** e reporte o melhor e a variação. A primeira execução paga disco e preparação.
4. **`$STORAGE`** (memória restante) e **`$ZSTORAGE`** (limite). Estourar dá **`<STORE>`** — a solução é PPG em vez de variável local.
5. **N+1** é o erro dominante: uma consulta mais um acesso por linha. Sinal de alerta: **acesso ao banco dentro de laço**.
6. Ordem de eficiência típica: agregação no SQL > uma consulta com junção > N+1.
7. **Gravar é muito mais caro que ler.** Não regrave por precaução.
8. **Função aplicada à coluna no `WHERE` mata o índice.** A coluna precisa aparecer nua na comparação.
9. **Plano de consulta**: Portal → Show Plan, ou `$SYSTEM.SQL.Explain()`. Procure "leitura por índice" contra "varredura do mapa mestre".
10. **Colete estatísticas** com volume representativo, e **expurgue as consultas em cache** depois.
11. Tipos de índice: **comum** (alta cardinalidade), **bitmap** (baixa cardinalidade, tabela grande), **bitslice** (agregação numérica), **`Data = (...)`** (cobertura).
12. **Índice de cobertura se desenha a partir de uma consulta concreta**, nunca por precaução.
13. Índice custa espaço e torna cada gravação mais lenta. Antes de criar, saiba **qual consulta** ele acelera.
14. **Dicas de otimizador** (`%PARALLEL`, `%NOINDEX`, `%IGNOREINDEX`, `%ALLINDEX`) por último e documentadas — elas congelam decisões.
15. **`%PARALLEL` só compensa em volume alto.**
16. Evite `SELECT *`: custa leitura, impede cobertura e pode arrastar streams.
17. Em globais: não releia o mesmo nó, use `MERGE` para subárvores, prefira `$ORDER` a `$QUERY`, aproveite o terceiro argumento do `$ORDER`.
18. **Transações curtas**, travas curtas, e `$INCREMENT` em vez de ler-somar-gravar.
19. **Lotes** não são mais rápidos: compram journal constante, travas curtas e **capacidade de retomar**. Grave o progresso e use `HANG` entre lotes.
20. **Paralelize depois de otimizar.** Unidades de trabalho precisam ser independentes.
21. **Cache que responde errado é pior que nenhum cache.** Pense na invalidação antes de implementar.
22. Conceitos de escala: **buffers globais**, **cache de consultas**, **ECP** (compartilhar dados entre servidores) e **sharding** (particionar entre nós).
23. **Antes de comparar tempos, verifique que os algoritmos dão o mesmo resultado.**

---

## 10. Cartões de memorização

**Frente:** Qual é a ordem correta ao otimizar?
**Verso:** Medir → identificar o gargalo → corrigir → medir de novo.

**Frente:** Qual função usar para medir o tempo de um trecho de código?
**Verso:** `$ZHOROLOG` — segundos com fração desde o início do dia.

**Frente:** Por que medir mais de uma vez?
**Verso:** A primeira execução paga o custo de trazer dados do disco e preparar a consulta.

**Frente:** O que é o problema N+1?
**Verso:** Uma consulta traz N linhas e o código faz mais N acessos individuais ao banco — 1+N idas onde bastaria uma.

**Frente:** Qual o sinal de alerta de um N+1?
**Verso:** `%OpenId()` ou qualquer acesso ao banco **dentro de um laço**.

**Frente:** Por que `WHERE UPPER(Nome) = 'X'` não usa o índice?
**Verso:** O índice está ordenado pelos valores da coluna, não pelos transformados. A coluna precisa aparecer nua na comparação.

**Frente:** Para que servem as estatísticas de tabela?
**Verso:** Para o otimizador conhecer tamanho e seletividade e escolher o melhor plano.

**Frente:** O que fazer depois de coletar estatísticas ou mudar o esquema?
**Verso:** Expurgar as consultas em cache, senão os planos antigos continuam sendo usados.

**Frente:** O que é um índice de cobertura?
**Verso:** Um índice com `Data = (colunas)`, que responde a consulta sem tocar nos dados principais.

**Frente:** Quando usar bitmap? E bitslice?
**Verso:** Bitmap: poucos valores distintos em tabela grande. Bitslice: coluna numérica usada em agregações.

**Frente:** Quando `%PARALLEL` ajuda?
**Verso:** Em agregações sobre volumes grandes. Em consultas pequenas, atrapalha.

**Frente:** Por que evitar `SELECT *`?
**Verso:** Custa leitura desnecessária, impede índice de cobertura e pode arrastar streams.

**Frente:** Recebi `<STORE>`. O que fazer?
**Verso:** Trocar a variável local por uma global privada do processo (`^||`), que não consome memória do processo.

**Frente:** Lotes são mais rápidos que uma transação única?
**Verso:** Não — costumam ser um pouco mais lentos. Eles compram journal constante, travas curtas e **retomada após falha**.

**Frente:** O que colocar entre lotes de um processamento pesado?
**Verso:** Um `HANG` pequeno, para não monopolizar o sistema.

**Frente:** Como percorrer uma global economizando operações?
**Verso:** `set k = $ORDER(^X(k), 1, v)` — traz o subscrito e o valor numa só chamada.

**Frente:** Como copiar uma subárvore de global rapidamente?
**Verso:** `MERGE destino = origem` — uma operação, em vez de um laço.

**Frente:** Qual o maior risco de um cache?
**Verso:** Devolver dado velho. Isso deixa de ser problema de desempenho e vira problema de correção.

**Frente:** O que verificar antes de comparar o tempo de dois algoritmos?
**Verso:** Que os dois produzem exatamente o mesmo resultado.

**Frente:** O que o ECP resolve? E o sharding?
**Verso:** ECP: vários servidores de aplicação compartilham os dados de um servidor de banco com cache coerente. Sharding: particiona uma tabela grande entre vários nós.

**Frente:** Vale paralelizar um processo lento?
**Verso:** Só depois de otimizá-lo. Código lento em paralelo continua lento e ainda disputa recursos.

**Frente:** Antes de criar um índice, que pergunta fazer?
**Verso:** "Qual consulta específica ele acelera?" Sem resposta, não crie.

---

Fim do domínio **T3 — IRIS Features**. O próximo capítulo abre o **T4 — Functions & APIs**, o domínio de **maior peso na prova** (26 das 76 questões), começando por 4.1: percorrer e ordenar arrays.

Digite CONTINUAR para o próximo capítulo.
