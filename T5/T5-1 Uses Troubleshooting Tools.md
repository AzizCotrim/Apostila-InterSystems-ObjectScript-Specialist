# Apostila InterSystems ObjectScript Specialist
## Capítulo 20 — T5.1 Uses Troubleshooting Tools (Ferramentas de diagnóstico)

> Começa aqui o domínio **T5 — Handles and Resolves Errors** (12 questões de 76). Este primeiro tópico é sobre **descobrir o que está acontecendo**. Os dois próximos tratam de tratar e de reconhecer erros; este trata de enxergá-los.

---

## 1. O que você vai saber fazer ao terminar

1. Adotar um **método de diagnóstico** em vez de adivinhar.
2. Inspecionar variáveis com **`ZWRITE`** e bytes invisíveis com **`ZZDUMP`**.
3. Ler as variáveis especiais de erro: **`$ZERROR`**, **`$ECODE`**, **`$STACK`** e **`$ESTACK`**.
4. Examinar a **pilha de chamadas** com `$STACK(n, ...)`.
5. Pausar a execução com **`BREAK`** e com **`ZBREAK`**, e limpar pontos de parada.
6. Ligar o **rastreamento** de execução.
7. Registrar erros no **log de erros da aplicação** com `LOG^%ETN`, e consultá-lo.
8. Escrever no **`messages.log`** e saber quando isso é apropriado.
9. Usar o **Portal** para ver processos, travas, logs e erros.
10. Conhecer o **monitor linha a linha** e o relatório de desempenho do sistema.
11. Configurar o **depurador do VS Code**.
12. Construir uma camada de diagnóstico própria e levar o projeto à versão **2.1**.

---

## 2. O conceito em linguagem de gente

### 2.1 O método antes das ferramentas

Um sistema apresenta um problema. Existem dois jeitos de reagir.

**O jeito ruim:** ler o código, formar uma hipótese, mudar alguma coisa, ver se melhorou. Repetir.

Isso falha por três motivos: você quase sempre suspeita da parte errada; ao mudar várias coisas, não sabe qual resolveu; e, se o problema for intermitente, "parou de acontecer" não significa "foi corrigido".

**O jeito bom** tem cinco passos:

1. **Reproduzir.** Um problema que você consegue provocar é um problema que você consegue resolver. Um que só acontece "às vezes, em produção" é um problema que você vai perseguir por semanas.
2. **Observar.** Que erro exatamente? Em que linha? Com que dados? Qual o estado das variáveis naquele instante?
3. **Isolar.** Reduzir até o menor caso que ainda falha. Frequentemente, o próprio ato de reduzir revela a causa.
4. **Corrigir.** Uma coisa por vez.
5. **Verificar.** Reproduzir de novo o caso original e confirmar que passou — e que nada mais quebrou.

As ferramentas deste capítulo servem principalmente ao passo 2. Mas **nenhuma delas substitui o passo 1**: se você não consegue reproduzir, nenhuma ferramenta ajuda.

### 2.2 As camadas de observação

Existem quatro níveis de observação, do mais imediato ao mais estrutural. Escolher o nível certo economiza muito tempo.

**Nível 1 — olhar o estado.**
"Quanto vale esta variável agora?" Ferramentas: `ZWRITE`, `ZZDUMP`, `$ZERROR`.

**Nível 2 — parar e olhar.**
"Quero ver o estado exatamente naquele ponto." Ferramentas: `BREAK`, `ZBREAK`, depurador do VS Code.

**Nível 3 — registrar o que aconteceu.**
"Preciso saber o que ocorreu ontem às 3h da manhã." Ferramentas: log de erros da aplicação, `messages.log`, trilha de auditoria do Capítulo 6.

**Nível 4 — medir o comportamento.**
"O sistema está lento; onde?" Ferramentas: plano de consulta, monitor linha a linha, relatório de desempenho, bancada do Capítulo 12.

O erro clássico é usar o nível errado: tentar depurar com pontos de parada um problema que só acontece em produção às 3h — quando o que se precisa é de **log**.

### 2.3 O que o IRIS já registra sozinho

Antes de instrumentar qualquer coisa, saiba que o IRIS já mantém três registros:

**Log de erros da aplicação.** Quando um erro não tratado ocorre, ele pode ser registrado com todo o contexto: mensagem, pilha de chamadas, variáveis. Fica na global `^ERRORS` e é visível no Portal.

**`messages.log` (console log).** Eventos de operação da instância: inicialização, problemas de disco, avisos do sistema. É o log do administrador.

**Journal.** Registro técnico de alterações, para rollback e recuperação (Capítulo 8). Não é log de diagnóstico, mas às vezes responde "quando este dado mudou?".

Consultar esses três antes de instrumentar o próprio código economiza trabalho: a resposta pode já estar lá.

### 2.4 A pilha de chamadas

Quando o método A chama B, que chama C, e C falha, a informação valiosa não é apenas "falhou em C" — é **como se chegou até C**.

Essa cadeia se chama **pilha de chamadas**, e o IRIS a mantém acessível.

A analogia é a de um rastro de migalhas: você está perdido na floresta, mas pode olhar para trás e ver por onde veio. Sem a pilha, você só sabe onde está.

Isso importa especialmente quando o método que falha é genérico — um utilitário chamado de cinquenta lugares. Saber que ele falhou não ajuda; saber **quem** o chamou, sim.

---

## 3. Inspecionando o estado

### 3.1 `ZWRITE` — o inspetor universal

```
LABSTUDY>SET a = 42, b = "texto", c("x") = 1, c("y", "z") = 2

LABSTUDY>ZWRITE a
a=42

LABSTUDY>ZWRITE c
c("x")=1
c("y","z")=2

LABSTUDY>ZWRITE
a=42
b="texto"
c("x")=1
c("y","z")=2
```

- **Com argumento**, mostra aquela variável e toda a sua subárvore.
- **Sem argumento**, mostra **todas** as variáveis locais do contexto atual.
- Funciona igualmente com globais e com PPG.
- Mostra **aspas em texto e sem aspas em número**, o que revela imediatamente se um valor é canônico (Capítulo 13).

`ZWRITE` sem argumento, dentro de um método parado num ponto de interrupção, é a ferramenta mais usada de todo o diagnóstico.

Para objetos, `ZWRITE` sobre uma OREF mostra as propriedades:

```
LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>ZWRITE p
p=<OBJECT REFERENCE>[1@LabStudy.Patient]
+----------------- general information ---------------
|      oref value: 1
|      class name: LabStudy.Patient
| reference count: 2
+----------------- attribute values ------------------
|            Name = "Paciente 00001"
|    RecordNumber = "REG-000001"
...
```

Esse despejo mostra o nome da classe **real** do objeto, o que é útil para confirmar polimorfismo (Capítulo 18).

### 3.2 `ZZDUMP` — vendo os bytes

Quando um valor "parece certo" mas as comparações falham, quase sempre há um caractere invisível.

```
LABSTUDY>SET x = "abc"_$CHAR(13)

LABSTUDY>WRITE x, "|", !
abc|

LABSTUDY>WRITE $LENGTH(x), !
4

LABSTUDY>ZZDUMP x

0000: 61 62 63 0D                                   abc.
```

**`ZZDUMP`** mostra o conteúdo em hexadecimal, byte a byte. O `0D` no fim é o retorno de carro invisível.

Este é o comando que resolve a classe de problemas mais frustrante que existe: `"abc" = "abc"` devolvendo `0`. Os dois "parecem" iguais na tela e não são.

Use `ZZDUMP` sempre que:

- uma comparação de texto falha inexplicavelmente;
- `$LENGTH` devolve um número diferente do que você conta na tela;
- dados vindos de arquivo ou de rede se comportam de forma estranha.

### 3.3 As variáveis de erro

```
LABSTUDY>WRITE 1/0

WRITE 1/0
       ^
<DIVIDE>

LABSTUDY>WRITE $ZERROR, !
<DIVIDE>

LABSTUDY>WRITE $ECODE, !
,M9,
```

- **`$ZERROR`** — o texto do último erro, com a localização quando disponível. É a variável que você lê primeiro.
- **`$ECODE`** — a lista de códigos de erro, no formato `,código,`. Códigos que começam com `M` são do padrão da linguagem; com `Z`, específicos do IRIS; com `U`, definidos pelo usuário.

Dentro de um método, `$ZERROR` traz também a rotina e o deslocamento:

```
<DIVIDE>zMeuMetodo+7^LabStudy.Demo.Erros.1
```

Leia: erro de divisão, no método `MeuMetodo`, 7 linhas depois do rótulo, na rotina gerada a partir da classe `LabStudy.Demo.Erros`. **Esse deslocamento é o que permite achar a linha exata.**

Um cuidado: `$ZERROR` guarda o **último** erro do processo, inclusive erros já tratados. Se você a consultar tarde demais, pode estar vendo outro erro. Dentro de um `catch`, prefira o objeto de exceção — assunto do Capítulo 21.

### 3.4 A pilha de chamadas

```objectscript
write "nível atual: ", $STACK, !
write "níveis desde o NEW: ", $ESTACK, !

for i = 1:1:$STACK {
    write i, ": ", $STACK(i, "PLACE"), "  [", $STACK(i, "MCODE"), "]", !
}
```

- **`$STACK`** — o nível atual da pilha.
- **`$STACK(n, "PLACE")`** — onde está o nível `n`.
- **`$STACK(n, "MCODE")`** — a linha de código daquele nível.
- **`$STACK(n, "ECODE")`** — o código de erro associado, se houver.
- **`$ESTACK`** — conta níveis a partir de um ponto de referência que você marca com `NEW $ESTACK`.

A diferença entre `$STACK` e `$ESTACK` é útil: `NEW $ESTACK` no início de um método faz `$ESTACK` contar **a partir dali**, permitindo saber quantos níveis foram acrescentados desde a entrada.

Ver a pilha responde à pergunta "como cheguei aqui?", que costuma ser mais informativa do que "onde estou?".

---

## 4. Parando a execução

### 4.1 `BREAK` no código

```objectscript
ClassMethod Calcular(a As %Numeric, b As %Numeric) As %Numeric
{
    set intermediario = a * 2

    break                          // pausa aqui, no Terminal

    quit intermediario + b
}
```

O comando **`BREAK`**, executado num contexto interativo, **pausa** e devolve o controle ao Terminal, mantendo todo o estado. Ali você pode:

```
LABSTUDY>ZWRITE
LABSTUDY>WRITE intermediario
LABSTUDY>SET intermediario = 100
LABSTUDY>GOTO
```

- **`ZWRITE`** mostra tudo.
- Você pode **alterar** variáveis e continuar.
- **`GOTO`** (sem argumento) retoma a execução.

**Restrições importantes:**

- `BREAK` só funciona em contexto **interativo**. Num método chamado por SQL, API ou processo em segundo plano, ele não faz o que você espera.
- **Nunca deixe um `BREAK` em código que vai para produção.** Ele congela o processo esperando alguém que não existe.

### 4.2 `ZBREAK` — pontos de parada sem alterar o código

```
LABSTUDY>ZBREAK Calcular+3^LabStudy.Demo.Erros

LABSTUDY>ZBREAK *
```

- **`ZBREAK local`** define um ponto de parada numa posição, **sem** editar a classe.
- **`ZBREAK *`** remove todos os pontos de parada.
- É possível associar **condições** e **ações** ao ponto de parada, executando código automaticamente quando ele é atingido.

A vantagem sobre o `BREAK` no código é evidente: você não precisa alterar e recompilar a classe, e não corre o risco de esquecer o comando lá dentro.

A sintaxe completa de `ZBREAK`, com condições, contadores e ações, é extensa: **verificar na documentação oficial**.

### 4.3 Rastreamento

```
LABSTUDY>ZBREAK /TRACE:ON

LABSTUDY>DO ##class(LabStudy.Demo.Erros).Calcular(2, 3)
   (cada linha executada é exibida)

LABSTUDY>ZBREAK /TRACE:OFF
```

O rastreamento exibe **cada linha executada**. É extremamente verboso — e extremamente útil quando você não faz ideia de por onde o código está passando.

Use em trechos curtos. Ligar o rastreamento e chamar um método que percorre 2000 registros produz uma quantidade de saída inútil.

### 4.4 O depurador do VS Code

A extensão do IRIS para VS Code oferece depuração gráfica: pontos de parada clicando na margem, inspeção de variáveis em painel, execução passo a passo.

A configuração vive num arquivo `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "objectscript",
      "request": "launch",
      "name": "Executar método",
      "program": "##class(LabStudy.Demo.Erros).Calcular(2,3)"
    },
    {
      "type": "objectscript",
      "request": "attach",
      "name": "Anexar a um processo",
      "processId": "${command:PickProcess}"
    }
  ]
}
```

- **`launch`** executa um método e para nos pontos de parada.
- **`attach`** conecta a um processo **já em execução** — útil para investigar um processo travado ou um serviço.

O formato exato do `launch.json` e os recursos disponíveis variam com a versão da extensão: **verificar na documentação oficial**.

**Quando o depurador gráfico vence:** lógica complexa, muitas variáveis, necessidade de percorrer passo a passo.

**Quando o Terminal vence:** verificação rápida, ambiente sem VS Code configurado, processo remoto, problema que só ocorre sob carga.

---

## 5. Registrando o que aconteceu

### 5.1 O log de erros da aplicação

Quando um erro ocorre, você pode registrá-lo com todo o contexto:

```objectscript
try {
    // ...
}
catch e {
    do LOG^%ETN
    // ... tratamento ...
}
```

**`LOG^%ETN`** é uma rotina do sistema que grava, na global **`^ERRORS`**, o erro atual com: mensagem, pilha de chamadas, variáveis do contexto, usuário, processo e horário.

Para consultar, o caminho normal é o Portal: **System Operation → System Logs → Application Error Log**.

Programaticamente, a global pode ser inspecionada:

```
LABSTUDY>ZWRITE ^ERRORS
```

Ela é organizada por data e por número sequencial. O formato interno pode mudar entre versões: **verificar na documentação oficial** antes de escrever código que dependa dele.

**Por que isso é valioso:** o log de erros captura o estado das variáveis **no momento da falha**. Numa investigação posterior, isso é frequentemente a única forma de reconstruir o que aconteceu.

### 5.2 O log do sistema

```objectscript
do ##class(%SYS.System).WriteToConsoleLog("LabStudy: importacao noturna concluida, 1284 registros")
```

Recapitulando o Capítulo 6: use com parcimônia. Este arquivo é do administrador, e enchê-lo de mensagens de aplicação atrapalha quem precisa dele.

Reserve para: início e fim de processos longos, falhas de integração, condições anormais que exigem atenção operacional.

### 5.3 Log próprio da aplicação

Para diagnóstico contínuo, uma trilha própria costuma ser mais prática, como visto no Capítulo 6. A estrutura mínima útil:

```objectscript
ClassMethod Log(level As %String, area As %String, message As %String) As %Status
{
    set seq = $INCREMENT(^LabStudyLog("counter"))

    set ^LabStudyLog("entry", seq, "when") = $ZDATETIME($HOROLOG, 3)
    set ^LabStudyLog("entry", seq, "level") = level
    set ^LabStudyLog("entry", seq, "area") = area
    set ^LabStudyLog("entry", seq, "message") = message
    set ^LabStudyLog("entry", seq, "job") = $JOB
    set ^LabStudyLog("entry", seq, "user") = $USERNAME

    quit $$$OK
}
```

O ramo `"entry"` parece supérfluo agora, com uma coisa só na global — mas é o que permite acrescentar índices (`"byLevel"`, `"byArea"`) e um contador sem misturar números de sequência com subscritos de texto no mesmo nível. Como você viu no Capítulo 13, texto ordena **depois** de número, e essa mistura estraga qualquer leitura reversa. O exercício 20.3 mostra a estrutura completa.

Com **níveis** (`DEBUG`, `INFO`, `WARN`, `ERROR`), você pode ligar e desligar a verbosidade sem alterar o código:

```objectscript
quit:level="DEBUG"&&'$GET(^LabStudyConfig("debug"), 0) $$$OK
```

Isso resolve o dilema entre "quero muita informação quando algo dá errado" e "não quero encher o disco em operação normal".

### 5.4 O Portal como painel de diagnóstico

Telas que vale conhecer, todas em **System Operation**:

| Tela | Responde a |
|---|---|
| **Processes** | quais processos existem, o que estão executando, quanto consomem |
| **Locks** | quem está segurando qual trava (Capítulo 5) |
| **Journals** | arquivos de journal e seu conteúdo |
| **System Logs → Messages Log** | o log do sistema |
| **System Logs → Application Error Log** | os erros registrados |
| **Databases** | tamanho e uso das bases |

E em **System Explorer**:

| Tela | Responde a |
|---|---|
| **SQL → Show Plan** | como uma consulta será executada (Capítulo 12) |
| **Globals** | conteúdo bruto dos dados (Capítulo 8) |
| **Classes** | o que está compilado |

**A tela de processos é a primeira parada quando algo está travado.** Ela mostra o que cada processo executa neste instante, e permite examinar variáveis e até encerrar processos.

### 5.5 Monitores de desempenho

Para problemas de lentidão, além da bancada do Capítulo 12:

**Monitor linha a linha:**

```
LABSTUDY>DO ^%SYS.MONLBL
```

Ele mede o tempo gasto **em cada linha** das rotinas selecionadas. É a ferramenta definitiva para descobrir onde o tempo vai dentro de um método — e é cara, então roda-se por períodos curtos e sobre um conjunto restrito de rotinas.

**Relatório de desempenho do sistema:**

```
LABSTUDY>DO ^SystemPerformance
```

Coleta métricas de todo o sistema por um período e produz um relatório completo, usado no diagnóstico de problemas de infraestrutura e frequentemente solicitado pelo suporte da InterSystems.

Os nomes e as opções dessas rotinas variam por versão: **verificar na documentação oficial**.

### 5.6 Inspecionando outros processos

No namespace `%SYS`, é possível consultar processos por SQL:

```objectscript
zn "%SYS"

set rs = ##class(%SQL.Statement).%ExecDirect(,
    "SELECT Pid, UserName, Namespace, Routine, State, CommandsExecuted "
    _"FROM %SYS.ProcessQuery_CONTROLPANEL")

while rs.%Next() {
    write rs.%Get("Pid"), " ", rs.%Get("Routine"), " ", rs.%Get("State"), !
}
```

O nome exato da consulta e as colunas disponíveis variam por versão: **verificar na documentação oficial**. O caminho garantido é o Portal.

E para encerrar um processo travado:

```objectscript
do $SYSTEM.Process.Terminate(pid)
```

Use com cuidado: encerrar um processo no meio de uma transação provoca rollback, e encerrar um processo do sistema pode desestabilizar a instância.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/Diag.cls`:

```objectscript
/// Troubleshooting tools, demonstrated on deliberately broken code.
Class LabStudy.Demo.Diag Extends %RegisteredObject
{

/// Inspecting values, including invisible characters.
ClassMethod Inspect() As %Status
{
    set clean = "REG-000001"
    set dirty = "REG-000001"_$CHAR(13)
    set spaced = " REG-000001"

    write "-- os tres parecem iguais na tela --", !
    write "  [", clean, "]", !
    write "  [", dirty, "]", !
    write "  [", spaced, "]", !

    write !, "-- mas nao sao --", !
    write "  clean  = dirty  : ", (clean = dirty), !
    write "  clean  = spaced : ", (clean = spaced), !
    write "  comprimentos    : ", $LENGTH(clean), " ", $LENGTH(dirty), " ", $LENGTH(spaced), !

    write !, "-- ZZDUMP revela --", !
    write "clean:", !
    zzdump clean
    write "dirty:", !
    zzdump dirty
    write "spaced:", !
    zzdump spaced

    write !, "-- limpeza --", !
    write "  apos ZSTRIP: ", ($ZSTRIP(dirty, "<>W") = clean), !

    quit $$$OK
}

/// Shows the call stack from inside a nested call.
ClassMethod Level1() As %Status
{
    write "-- iniciando em Level1 --", !
    do ..Level2()
    quit $$$OK
}

ClassMethod Level2() As %Status
{
    do ..Level3()
    quit $$$OK
}

ClassMethod Level3() As %Status
{
    write "  $STACK  : ", $STACK, !
    write "  $ESTACK : ", $ESTACK, !
    write !, "  pilha de chamadas:", !

    for i = 1:1:$STACK {
        write "    ", $JUSTIFY(i, 2), ": ", $STACK(i, "PLACE"), !
    }
    quit $$$OK
}

/// Produces an error and reads the error variables.
ClassMethod CauseError(kind As %String = "divide") As %Status
{
    // clear before, so we know the error is ours
    set $ZERROR = ""

    write "-- provocando erro do tipo: ", kind, " --", !

    try {
        if kind = "divide" {
            set x = 1 / 0
        } elseif kind = "undefined" {
            write naoExiste
        } elseif kind = "oref" {
            set o = ""
            write o.Propriedade
        } elseif kind = "subscript" {
            set y = $LIST($LISTBUILD("a"), 9)
        } else {
            set z = $ZDATEH("nao e uma data", 3)
        }
    }
    catch e {
        write "  $ZERROR  : ", $ZERROR, !
        write "  $ECODE   : ", $ECODE, !
        write "  exception: ", e.DisplayString(), !
        write "  name     : ", e.Name, !
        write "  location : ", e.Location, !
    }

    quit $$$OK
}

/// A bug to be investigated: the average comes out wrong.
ClassMethod BuggyAverage(ByRef values) As %Numeric
{
    set total = 0
    set count = 0
    set k = ""

    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""

        set count = count + 1
        set total = total + v          // empty values silently become zero
    }

    quit $SELECT(count > 0: total / count, 1: "")
}

/// The same, instrumented so the problem becomes visible.
ClassMethod InstrumentedAverage(ByRef values, debug As %Boolean = 1) As %Numeric
{
    set total = 0, count = 0, skipped = 0, k = ""

    write:debug "  -- percorrendo --", !

    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""

        if v = "" {
            set skipped = skipped + 1
            write:debug "    [", k, "] VAZIO -- ignorado", !
            continue
        }

        set count = count + 1
        set total = total + v
        write:debug "    [", k, "] = ", v, "  acumulado: ", total, " em ", count, !
    }

    if debug {
        write "  -- resumo --", !
        write "    considerados : ", count, !
        write "    ignorados    : ", skipped, !
        write "    soma         : ", total, !
    }

    quit $SELECT(count > 0: total / count, 1: "")
}

ClassMethod CompareAverages() As %Status
{
    kill vals
    set vals(1) = 10, vals(2) = 20, vals(3) = "", vals(4) = 30, vals(5) = ""

    write "-- dados --", !
    zwrite vals

    write !, "-- versao com bug --", !
    write "  resultado: ", ..BuggyAverage(.vals), !

    write !, "-- versao instrumentada --", !
    write "  resultado: ", ..InstrumentedAverage(.vals, 1), !

    quit $$$OK
}

/// Logs the current error into the application error log.
ClassMethod LogAnError() As %Status
{
    try {
        set x = 1 / 0
    }
    catch e {
        do LOG^%ETN
        write "  erro registrado em ^ERRORS", !
        write "  consulte no Portal: System Operation > System Logs > Application Error Log", !
    }
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Inspect()      write !
    do ..Level1()       write !

    for kind = "divide", "undefined", "oref", "subscript", "date" {
        do ..CauseError(kind)
        write !
    }

    do ..CompareAverages()

    quit $$$OK
}

}
```

### 6.1 Executando

```
LABSTUDY>DO ##class(LabStudy.Demo.Diag).Inspect()
-- os tres parecem iguais na tela --
  [REG-000001]
  [REG-000001]
  [ REG-000001]

-- mas nao sao --
  clean  = dirty  : 0
  clean  = spaced : 0
  comprimentos    : 10 11 11

-- ZZDUMP revela --
clean:

0000: 52 45 47 2D 30 30 30 30 30 31                 REG-000001

dirty:

0000: 52 45 47 2D 30 30 30 30 30 31 0D              REG-000001.

spaced:

0000: 20 52 45 47 2D 30 30 30 30 30 31              .REG-000001

-- limpeza --
  apos ZSTRIP: 1
```

- **Os três textos parecem idênticos na tela**, mas dois têm 11 caracteres. `$LENGTH` já denuncia; `ZZDUMP` mostra exatamente **onde** está a diferença: `0D` no fim de um, `20` no início do outro.
- **Este é o cenário do "não estou entendendo, os dois são iguais"**, resolvido em segundos. Sem `ZZDUMP`, ele consome horas.

```
LABSTUDY>DO ##class(LabStudy.Demo.Diag).Level1()
-- iniciando em Level1 --
  $STACK  : 4
  $ESTACK : 4

  pilha de chamadas:
     1: DO ##class(LabStudy.Demo.Diag).Level1()
     2: zLevel1+2^LabStudy.Demo.Diag.1
     3: zLevel2+1^LabStudy.Demo.Diag.1
     4: zLevel3+1^LabStudy.Demo.Diag.1
```

- **A pilha mostra o caminho inteiro**, do comando digitado no Terminal até o método mais interno. Cada nível traz o rótulo e o deslocamento.
- **Isso é o que você quer ver quando um utilitário genérico falha.** Saber que `Text.Clean()` deu erro é inútil; saber que foi chamado por `ImportPatientsCsv` na linha tal é a informação.

```
LABSTUDY>DO ##class(LabStudy.Demo.Diag).CauseError("oref")
-- provocando erro do tipo: oref --
  $ZERROR  : <INVALID OREF>zCauseError+14^LabStudy.Demo.Diag.1
  $ECODE   : ,M8,
  exception: <INVALID OREF>zCauseError+14^LabStudy.Demo.Diag.1
  name     : <INVALID OREF>
  location : zCauseError+14^LabStudy.Demo.Diag.1
```

- **`$ZERROR` traz mensagem e localização juntas.** O `+14` é o deslocamento em linhas a partir do rótulo do método — é assim que se encontra a linha exata.
- **O objeto de exceção separa as partes** (`Name`, `Location`), o que é mais prático para registrar em log estruturado. Isso é assunto do próximo capítulo.

```
LABSTUDY>DO ##class(LabStudy.Demo.Diag).CompareAverages()
-- dados --
vals(1)=10
vals(2)=20
vals(3)=""
vals(4)=30
vals(5)=""

-- versao com bug --
  resultado: 12

-- versao instrumentada --
  -- percorrendo --
    [1] = 10  acumulado: 10 em 1
    [2] = 20  acumulado: 30 em 2
    [3] VAZIO -- ignorado
    [4] = 30  acumulado: 60 em 3
    [5] VAZIO -- ignorado
  -- resumo --
    considerados : 3
    ignorados    : 2
    soma         : 60
  resultado: 20
```

**Este é o exemplo mais importante do capítulo.** Observe:

- **`ZWRITE vals` mostrou imediatamente que dois valores estavam vazios** — com aspas, distinguindo-os de zero. A causa do problema estava visível antes de qualquer depuração.
- **A versão com bug devolveu 12; a correta, 20.** O bug é o do Capítulo 10: vazio virando zero na soma e contando no divisor.
- **A instrumentação não corrigiu nada — ela tornou o problema visível.** Cada iteração mostra o acumulado e a contagem. Olhando a saída, fica evidente que a versão com bug contaria 5 elementos onde há 3.
- **A instrumentação é controlada por um parâmetro `debug`.** Em produção, chama-se com `0` e nada é impresso. Este é o padrão que permite deixar a instrumentação **no código**, pronta para ser ligada quando necessário — em vez de acrescentá-la e removê-la a cada investigação.

---

## 7. Pegadinhas e erros comuns

**1) Mudar código antes de reproduzir o problema.**
Você não saberá se corrigiu ou se apenas mascarou.

**2) Mudar várias coisas de uma vez.**
Se melhorar, você não sabe qual foi.

**3) Confiar em "parou de acontecer".**
Problemas intermitentes exigem entender a causa, não observar a ausência.

**4) Comparar textos sem verificar bytes invisíveis.**
Use `ZZDUMP` quando `"a" = "a"` der `0`.

**5) Ler `$ZERROR` tarde demais.**
Ela guarda o **último** erro do processo, que pode ser outro. Leia imediatamente, ou use o objeto de exceção.

**6) Deixar `BREAK` em código de produção.**
Congela o processo esperando alguém que não está lá.

**7) Usar `BREAK` em contexto não interativo.**
Não funciona como esperado em processos de segundo plano, SQL ou APIs.

**8) Esquecer de limpar pontos de parada.**
`ZBREAK *` remove todos.

**9) Ligar rastreamento sobre um processamento longo.**
Produz uma avalanche de saída inútil. Use em trechos curtos.

**10) Tentar depurar com pontos de parada um problema de produção intermitente.**
O nível certo é **log**, não depurador.

**11) Encher o `messages.log` com mensagens de aplicação.**
É o log do administrador. Use uma trilha própria.

**12) Não registrar erros com contexto.**
"Deu erro" não ajuda. `LOG^%ETN` captura variáveis, pilha, usuário e horário.

**13) Acrescentar e remover instrumentação a cada investigação.**
Deixe-a no código, controlada por um parâmetro de depuração.

**14) Encerrar processos sem entender o que fazem.**
Pode provocar rollback de transações em andamento ou desestabilizar a instância.

**15) Ignorar a tela de processos do Portal.**
É a primeira parada quando algo está travado — e frequentemente a última.

**16) Confiar num único ponto de medição de tempo.**
Meça várias vezes (Capítulo 12).

---

## 8. MÃO NA MASSA

---

### Exercício 20.1 — Caça ao caractere invisível

**a) Enunciado:** Crie `LabStudy.Demo.Dg1` que reproduza e resolva o problema clássico de dados "iguais que não são iguais":

1. `ClassMethod Contaminate()` — cria um array com valores contaminados de cinco formas diferentes: espaço no início, espaço no fim, retorno de carro no fim, tabulação no meio, e um caractere de controle invisível.
2. `ClassMethod Compare(esperado)` — compara cada valor com o esperado, mostrando o resultado da comparação, o comprimento e o despejo hexadecimal apenas dos que diferem.
3. `ClassMethod Diagnose(valor)` — descreve o que há de errado num valor: lista as posições e os códigos de todos os caracteres não imprimíveis.
4. `ClassMethod Sanitize(valor)` — limpa e devolve o valor corrigido.
5. `ClassMethod Report()` — junta tudo: contamina, compara, diagnostica, limpa e compara de novo.

**b) Dica:** Um caractere é "não imprimível" quando seu código é menor que 32 ou igual a 127.

**c) Como testar:** Depois da limpeza, todos devem passar na comparação.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Dg1.cls`:

```objectscript
/// Hunting for invisible characters.
Class LabStudy.Demo.Dg1 Extends %RegisteredObject
{

Parameter EXPECTED = "REG-000001";

/// Builds values contaminated in five different ways.
ClassMethod Contaminate(Output values) As %Status
{
    kill values

    set values("limpo")      = ..#EXPECTED
    set values("espaco-ini") = " "_..#EXPECTED
    set values("espaco-fim") = ..#EXPECTED_"  "
    set values("cr-fim")     = ..#EXPECTED_$CHAR(13)
    set values("tab-meio")   = "REG-"_$CHAR(9)_"000001"
    set values("controle")   = ..#EXPECTED_$CHAR(1)

    quit $$$OK
}

/// Describes what is wrong with a value.
ClassMethod Diagnose(value As %String) As %String
{
    set problems = ""

    // leading and trailing whitespace
    if $EXTRACT(value, 1) = " " {
        set problems = problems_$LISTBUILD("espaco no inicio")
    }
    if $EXTRACT(value, *) = " " {
        set problems = problems_$LISTBUILD("espaco no fim")
    }

    // non printable characters
    for i = 1:1:$LENGTH(value) {
        set code = $ASCII(value, i)
        continue:(code >= 32) && (code '= 127)

        set problems = problems_$LISTBUILD(
            "caractere de controle na posicao "_i_" (codigo "_code_")")
    }

    quit:problems="" "(nenhum problema detectado)"
    quit ##class(LabStudy.ListUtil).ToDisplay(problems, "; ")
}

/// Cleans a value.
ClassMethod Sanitize(value As %String) As %String
{
    set value = $ZSTRIP(value, "*C")        // control characters
    set value = $ZSTRIP(value, "<>W")       // outer whitespace
    quit value
}

/// Compares every value against the expected one.
ClassMethod Compare(ByRef values, expected As %String = {..#EXPECTED}, showDump As %Boolean = 1) As %Integer
{
    set W = $LISTBUILD(14, 8, 6, 40)
    set A = $LISTBUILD("L", "C", "R", "L")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("rotulo", "igual?", "tam", "diagnostico"), W, A)

    set bad = 0, k = ""
    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""

        set same = (v = expected)
        set:'same bad = bad + 1

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(k,
                       $SELECT(same: "sim", 1: "NAO"),
                       $LENGTH(v),
                       $SELECT(same: "", 1: ..Diagnose(v))),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(72)
    write "  ", bad, " valores diferentes do esperado", !

    if showDump && bad {
        write !, "  -- despejo hexadecimal dos divergentes --", !
        set k = ""
        for {
            set k = $ORDER(values(k), 1, v)
            quit:k=""
            continue:v=expected

            write "  ", k, ":", !
            zzdump v
        }
    }

    quit bad
}

ClassMethod Report() As %Status
{
    do ..Contaminate(.values)

    write "=== antes da limpeza ===", !
    set before = ..Compare(.values, ..#EXPECTED, 1)

    write !, "=== depois da limpeza ===", !

    kill cleaned
    set k = ""
    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""
        set cleaned(k) = ..Sanitize(v)
    }

    set after = ..Compare(.cleaned, ..#EXPECTED, 0)

    write !, "  antes: ", before, " divergentes   depois: ", after, " divergentes", !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Dg1).Report()
=== antes da limpeza ===
rotulo         igual?    tam  diagnostico
------------------------------------------------------------------------
controle        NAO       11  caractere de controle na posicao 11 (codigo 1)
cr-fim          NAO       11  caractere de controle na posicao 11 (codigo 13)
espaco-fim      NAO       12  espaco no fim
espaco-ini      NAO       11  espaco no inicio
limpo           sim       10
tab-meio        NAO       11  caractere de controle na posicao 5 (codigo 9)
------------------------------------------------------------------------
  5 valores diferentes do esperado

  -- despejo hexadecimal dos divergentes --
  controle:

0000: 52 45 47 2D 30 30 30 30 30 31 01              REG-000001.

  cr-fim:

0000: 52 45 47 2D 30 30 30 30 30 31 0D              REG-000001.

  espaco-fim:

0000: 52 45 47 2D 30 30 30 30 30 31 20 20           REG-000001..

  espaco-ini:

0000: 20 52 45 47 2D 30 30 30 30 30 31              .REG-000001

  tab-meio:

0000: 52 45 47 2D 09 30 30 30 30 30 31              REG-.000001

=== depois da limpeza ===
rotulo         igual?    tam  diagnostico
------------------------------------------------------------------------
controle        sim       10
cr-fim          sim       10
espaco-fim      sim       10
espaco-ini      sim       10
limpo           sim       10
tab-meio        sim       10
------------------------------------------------------------------------
  0 valores diferentes do esperado

  antes: 5 divergentes   depois: 0 divergentes
```

**Por que cada decisão:**

- **A tabela mostra comprimento e diagnóstico lado a lado.** O comprimento sozinho já denuncia: `10` contra `11` ou `12`. É a primeira coisa a olhar, e é gratuita.
- **`Diagnose` percorre caractere a caractere procurando códigos abaixo de 32.** Ele não adivinha: ele **verifica**. E informa a **posição**, que é o que permite entender de onde o problema veio — um caractere no fim sugere leitura de arquivo; no meio, sugere colagem de outro sistema.
- **O despejo hexadecimal confirma o diagnóstico.** `0D` é retorno de carro, `09` é tabulação, `01` é um caractere de controle, `20` é espaço. Depois de ver alguns, você passa a reconhecê-los.
- **`Sanitize` remove controles ANTES de remover espaços.** Se a ordem fosse invertida, um valor terminado em `" "_$CHAR(13)` teria o `$CHAR(13)` removido, mas o espaço já teria deixado de ser final na primeira passagem. **A ordem das limpezas importa** — a mesma lição do `Slug` no Capítulo 15.
- **A limpeza resolveu todos os cinco casos**, incluindo a tabulação no meio, que `$ZSTRIP(v, "<>W")` sozinho **não** resolveria — porque ela não está nas pontas. Foi o `"*C"` que a pegou.
- **O relatório compara antes e depois com um número.** "5 divergentes → 0 divergentes" é uma verificação objetiva, não uma impressão.

---

### Exercício 20.2 — Lendo a pilha e o erro

**a) Enunciado:** Crie `LabStudy.Demo.Dg2` para investigar uma falha que vem de longe:

1. Uma cadeia de quatro métodos: `Api()` → `Service()` → `Repository()` → `Helper()`, onde `Helper()` falha.
2. `ClassMethod Trace()` — dentro de `Helper()`, antes de falhar, imprime a pilha completa com `$STACK`.
3. `ClassMethod CatchAndReport()` — chama `Api()` dentro de um `try`, e no `catch` imprime: mensagem, código, localização e a pilha **no momento do erro**.
4. `ClassMethod ErrorCatalog()` — provoca seis erros diferentes e monta uma tabela com nome, `$ECODE` e uma explicação em português de cada um.
5. `ClassMethod StackDepth()` — demonstra a diferença entre `$STACK` e `$ESTACK`, usando `NEW $ESTACK`.

**b) Dica:** Dentro do `catch`, a pilha do erro está disponível pelo objeto de exceção; a pilha atual, por `$STACK`.

**c) Como testar:** A tabela de erros deve mostrar códigos diferentes para cada tipo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Dg2.cls`:

```objectscript
/// Reading the stack and the error variables.
Class LabStudy.Demo.Dg2 Extends %RegisteredObject
{

/// Prints the current call stack.
ClassMethod PrintStack(label As %String = "pilha") As %Status
{
    write "  -- ", label, " (profundidade ", $STACK, ") --", !

    for i = 1:1:$STACK {
        write "    ", $JUSTIFY(i, 2), ": ", $STACK(i, "PLACE"), !
    }
    quit $$$OK
}

// ---- a chain of calls, failing at the bottom ----

ClassMethod Api() As %Status
{
    quit ..Service()
}

ClassMethod Service() As %Status
{
    quit ..Repository()
}

ClassMethod Repository() As %Status
{
    quit ..Helper()
}

ClassMethod Helper() As %Status
{
    do ..PrintStack("dentro de Helper, antes de falhar")

    set x = 1 / 0                    // boom
    quit $$$OK
}

/// Calls the chain and reports everything about the failure.
ClassMethod CatchAndReport() As %Status
{
    set $ZERROR = ""

    try {
        do ..Api()
    }
    catch e {
        write !, "  -- informacoes do erro --", !
        write "    nome      : ", e.Name, !
        write "    codigo    : ", e.Code, !
        write "    local     : ", e.Location, !
        write "    texto     : ", e.DisplayString(), !
        write "    $ZERROR   : ", $ZERROR, !
        write "    $ECODE    : ", $ECODE, !

        write !
        do ..PrintStack("pilha no momento do catch")
    }

    quit $$$OK
}

/// Provokes several kinds of error and catalogues them.
ClassMethod ErrorCatalog() As %Status
{
    set W = $LISTBUILD(14, 18, 10, 34)
    set A = $LISTBUILD("L", "L", "L", "L")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("tipo", "erro", "$ECODE", "significado"), W, A)

    set kinds = $LISTBUILD("divide", "undefined", "oref", "list",
                           "syntax", "maxstring", "illegal")

    set ptr = 0
    while $LISTNEXT(kinds, ptr, kind) {
        set name = "", code = ""

        try {
            do ..Provoke(kind)
        }
        catch e {
            set name = e.Name
            set code = $ECODE
        }

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(kind, name, code, ..Explain(kind)), W, A)
    }

    do ##class(LabStudy.Formatter).Line(76)
    quit $$$OK
}

/// Produces one specific kind of error.
ClassMethod Provoke(kind As %String) As %Status [ Private ]
{
    if kind = "divide" {
        set x = 1 / 0
    } elseif kind = "undefined" {
        write naoDefinida
    } elseif kind = "oref" {
        set o = ""
        write o.Qualquer
    } elseif kind = "list" {
        write $LIST($LISTBUILD("a"), 5)
    } elseif kind = "syntax" {
        xecute "isto nao e codigo valido"
    } elseif kind = "maxstring" {
        set big = ""
        for i = 1:1:100 {
            set big = big_big_"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
        }
    } else {
        write $SELECT(1 = 2: "nunca")
    }
    quit $$$OK
}

ClassMethod Explain(kind As %String) As %String [ Private ]
{
    quit $CASE(kind,
        "divide":    "divisao por zero",
        "undefined": "variavel nao existe",
        "oref":      "referencia de objeto invalida",
        "list":      "posicao inexistente numa lista",
        "syntax":    "codigo invalido em XECUTE",
        "maxstring": "string maior que o limite",
        "illegal":   "$SELECT sem condicao verdadeira",
        :            "(desconhecido)")
}

/// $STACK versus $ESTACK.
ClassMethod StackDepth() As %Status
{
    write "-- no nivel do Terminal --", !
    write "  $STACK  : ", $STACK, !
    write "  $ESTACK : ", $ESTACK, !

    do ..DepthLevel1()
    quit $$$OK
}

ClassMethod DepthLevel1() As %Status
{
    // marks a new reference point for $ESTACK
    new $ESTACK

    write !, "-- apos NEW $ESTACK em DepthLevel1 --", !
    write "  $STACK  : ", $STACK, "   (continua contando desde o inicio)", !
    write "  $ESTACK : ", $ESTACK, "   (recomecou a contar aqui)", !

    do ..DepthLevel2()
    quit $$$OK
}

ClassMethod DepthLevel2() As %Status
{
    do ..DepthLevel3()
    quit $$$OK
}

ClassMethod DepthLevel3() As %Status
{
    write !, "-- tres niveis abaixo --", !
    write "  $STACK  : ", $STACK, !
    write "  $ESTACK : ", $ESTACK, "   (niveis acrescentados desde o NEW)", !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..CatchAndReport()
    write !
    do ..ErrorCatalog()
    write !
    do ..StackDepth()
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Dg2).Demo()
  -- dentro de Helper, antes de falhar (profundidade 7) --
     1: DO ##class(LabStudy.Demo.Dg2).Demo()
     2: zDemo+1^LabStudy.Demo.Dg2.1
     3: zCatchAndReport+4^LabStudy.Demo.Dg2.1
     4: zApi+1^LabStudy.Demo.Dg2.1
     5: zService+1^LabStudy.Demo.Dg2.1
     6: zRepository+1^LabStudy.Demo.Dg2.1
     7: zHelper+1^LabStudy.Demo.Dg2.1

  -- informacoes do erro --
    nome      : <DIVIDE>
    codigo    : 9
    local     : zHelper+3^LabStudy.Demo.Dg2.1
    texto     : <DIVIDE>zHelper+3^LabStudy.Demo.Dg2.1
    $ZERROR   : <DIVIDE>zHelper+3^LabStudy.Demo.Dg2.1
    $ECODE    : ,M9,

  -- pilha no momento do catch (profundidade 3) --
     1: DO ##class(LabStudy.Demo.Dg2).Demo()
     2: zDemo+1^LabStudy.Demo.Dg2.1
     3: zCatchAndReport+4^LabStudy.Demo.Dg2.1

tipo           erro               $ECODE     significado
----------------------------------------------------------------------------
divide         <DIVIDE>           ,M9,       divisao por zero
undefined      <UNDEFINED>        ,M6,       variavel nao existe
oref           <INVALID OREF>     ,M8,       referencia de objeto invalida
list           <LIST>             ,M...,     posicao inexistente numa lista
syntax         <SYNTAX>           ,M...,     codigo invalido em XECUTE
maxstring      <MAXSTRING>        ,M...,     string maior que o limite
illegal        <ILLEGAL VALUE>    ,M...,     $SELECT sem condicao verdadeira
----------------------------------------------------------------------------

-- no nivel do Terminal --
  $STACK  : 2
  $ESTACK : 2

-- apos NEW $ESTACK em DepthLevel1 --
  $STACK  : 4   (continua contando desde o inicio)
  $ESTACK : 0   (recomecou a contar aqui)

-- tres niveis abaixo --
  $STACK  : 6
  $ESTACK : 2   (niveis acrescentados desde o NEW)
```

**Por que cada resultado:**

- **A pilha dentro de `Helper` mostra os sete níveis**, do comando digitado até o método que falhou. **Esse é o rastro de migalhas.** Sem ele, você saberia apenas que houve divisão por zero em `Helper` — e `Helper` poderia ser chamado de cinquenta lugares.
- **A pilha no `catch` tem apenas três níveis**, porque os níveis abaixo já foram desfeitos quando a exceção subiu. **É por isso que a pilha do erro precisa vir do objeto de exceção, e não de `$STACK` no `catch`.** Este é um ponto sutil e frequentemente mal compreendido.
- **`Location` do erro (`zHelper+3`) difere do que a pilha mostrou (`zHelper+1`)**, porque a pilha foi impressa antes, no início do método. Os deslocamentos apontam linhas diferentes do mesmo método.
- **O catálogo de erros associa nome, código e significado.** Vale imprimir e ter à mão: reconhecer `<INVALID OREF>` como "referência de objeto vazia" economiza minutos toda vez que ele aparece. O Capítulo 22 aprofunda cada um.
- **`$ESTACK` reiniciou em zero depois do `NEW $ESTACK`**, enquanto `$STACK` continuou contando desde o Terminal. É essa a diferença: `$STACK` é absoluto, `$ESTACK` é relativo a um marco que você define. Em código que precisa saber "quantos níveis eu acrescentei", `$ESTACK` é a resposta.

---

### Exercício 20.3 — PROJETO CONTÍNUO: camada de diagnóstico

**a) Enunciado:** Leve o sistema à versão **2.1** com uma camada de diagnóstico completa:

1. `LabStudy.Log` — log estruturado da aplicação:
   - níveis `DEBUG`, `INFO`, `WARN`, `ERROR`;
   - `Write(nivel, area, mensagem, detalhe)`, e atalhos `Debug()`, `Info()`, `Warn()`, `Error()`;
   - `SetLevel(nivel)` e `GetLevel()` — controla a verbosidade sem alterar código, guardando a configuração numa global;
   - `Show(quantidade, nivelMinimo, area)` — exibe as últimas entradas, com filtro;
   - `Purge(dias)` — remove entradas antigas, com simulação por padrão;
   - `Stats()` — quantas entradas por nível e por área.
2. `LabStudy.Diagnostics`:
   - `Context()` — imprime o contexto completo do processo (útil ao registrar um erro);
   - `StackTrace()` — devolve a pilha como lista;
   - `DumpValue(rotulo, valor)` — imprime valor, comprimento, tipo aparente e despejo hexadecimal quando houver caracteres não imprimíveis;
   - `SelfCheck()` — verifica a saúde do sistema: esquema atualizado, índices consistentes, exames sem paciente, pacientes sem número de registro, e reporta cada item como OK ou PROBLEMA.
3. Instrumente `LabStudy.Exam.SetResult()` e `LabStudy.FileIO.ImportPatientsCsv()` com chamadas ao log.
4. Acrescente as opções ao menu e suba `LabStudy.App` para `"2.1"`.

**b) Dica:** No `SelfCheck`, cada verificação deve ser uma consulta SQL que devolve zero quando está tudo bem.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Log).SetLevel("DEBUG")
LABSTUDY>DO ##class(LabStudy.Diagnostics).SelfCheck()
LABSTUDY>DO ##class(LabStudy.Log).Show(20)
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Log.cls`:

```objectscript
/// Structured application log with levels.
/// Verbosity is configured at run time, so instrumentation can stay in the code.
Class LabStudy.Log Extends %RegisteredObject
{

Parameter LOGGLOBAL = "^LabStudyLog";

Parameter CONFIGGLOBAL = "^LabStudyLogConfig";

/// Level order, from most to least verbose.
Parameter LEVELS = "DEBUG,INFO,WARN,ERROR";

/// Numeric weight of a level. Higher means more severe.
ClassMethod Weight(level As %String) As %Integer
{
    set level = $ZCONVERT(level, "U")
    set n = $LISTFIND($LISTFROMSTRING(..#LEVELS, ","), level)
    quit $SELECT(n: n, 1: 0)
}

/// Current minimum level. Anything below is discarded.
ClassMethod GetLevel() As %String
{
    quit $GET(@..#CONFIGGLOBAL@("level"), "INFO")
}

ClassMethod SetLevel(level As %String) As %Status
{
    set level = $ZCONVERT(##class(LabStudy.Text).Clean(level), "U")

    if '..Weight(level) {
        write "  nivel invalido: [", level, "]. Use um de: ", ..#LEVELS, !
        quit $$$ERROR($$$GeneralError, "nivel invalido")
    }

    set @..#CONFIGGLOBAL@("level") = level
    write "  nivel de log definido como ", level, !
    quit $$$OK
}

/// Writes one entry, if the level is high enough.
ClassMethod Write(level As %String, area As %String, message As %String, detail As %String = "") As %Status
{
    set level = $ZCONVERT(level, "U")

    // the whole point of levels: cheap discard
    quit:..Weight(level) < ..Weight(..GetLevel()) $$$OK

    set seq = $INCREMENT(@..#LOGGLOBAL@("counter"))

    // entries live under their OWN branch, separate from the indexes.
    // Mixing numeric sequence numbers with subscripts like "byLevel" in
    // the same level would make a reverse $ORDER return the indexes first.
    set @..#LOGGLOBAL@("entry", seq, "when")    = ##class(LabStudy.DateTime).NowTimestamp()
    set @..#LOGGLOBAL@("entry", seq, "day")     = ##class(LabStudy.DateTime).Today()
    set @..#LOGGLOBAL@("entry", seq, "level")   = level
    set @..#LOGGLOBAL@("entry", seq, "area")    = area
    set @..#LOGGLOBAL@("entry", seq, "message") = $EXTRACT(message, 1, 500)
    set @..#LOGGLOBAL@("entry", seq, "job")     = $JOB
    set @..#LOGGLOBAL@("entry", seq, "user")    = $USERNAME

    set:detail'="" @..#LOGGLOBAL@("entry", seq, "detail") = $EXTRACT(detail, 1, 1000)

    // index by level and by area, for fast filtering
    set @..#LOGGLOBAL@("byLevel", level, seq) = ""
    set @..#LOGGLOBAL@("byArea", area, seq) = ""

    quit $$$OK
}

ClassMethod Debug(area As %String, message As %String, detail As %String = "") As %Status [ CodeMode = expression ]
{
..Write("DEBUG", area, message, detail)
}

ClassMethod Info(area As %String, message As %String, detail As %String = "") As %Status [ CodeMode = expression ]
{
..Write("INFO", area, message, detail)
}

ClassMethod Warn(area As %String, message As %String, detail As %String = "") As %Status [ CodeMode = expression ]
{
..Write("WARN", area, message, detail)
}

/// Records an error, adding the process context automatically.
ClassMethod Error(area As %String, message As %String, detail As %String = "") As %Status
{
    set context = "job="_$JOB_" user="_$USERNAME_" ns="_$NAMESPACE
    set:$ZERROR'="" context = context_" zerror="_$ZERROR

    quit ..Write("ERROR", area, message, detail_" | "_context)
}

/// Shows the most recent entries, newest first.
ClassMethod Show(count As %Integer = 20, minLevel As %String = "DEBUG", area As %String = "") As %Integer
{
    set minWeight = ..Weight(minLevel)

    set W = $LISTBUILD(20, 6, 12, 44)
    set A = $LISTBUILD("L", "L", "L", "L")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("quando", "nivel", "area", "mensagem"), W, A)

    set shown = 0, seq = ""

    for {
        // only sequence numbers live under "entry", so walking it
        // backwards is unambiguous: newest first, nothing else mixed in
        set seq = $ORDER(@..#LOGGLOBAL@("entry", seq), -1)
        quit:seq=""
        quit:shown>=count

        set level = $GET(@..#LOGGLOBAL@("entry", seq, "level"))
        continue:..Weight(level) < minWeight

        set entryArea = $GET(@..#LOGGLOBAL@("entry", seq, "area"))
        continue:(area '= "") && (entryArea '= area)

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD($GET(@..#LOGGLOBAL@("entry", seq, "when")),
                       level,
                       entryArea,
                       $GET(@..#LOGGLOBAL@("entry", seq, "message"))),
            W, A)

        set shown = shown + 1
    }

    do ##class(LabStudy.Formatter).Line(84)
    write "  ", shown, " entradas (nivel atual: ", ..GetLevel(), ")", !
    quit shown
}

/// Counts entries by level and by area.
ClassMethod Stats() As %Status
{
    write "=== log por nivel ===", !

    for i = 1:1:$LENGTH(..#LEVELS, ",") {
        set level = $PIECE(..#LEVELS, ",", i)

        set n = 0, seq = ""
        for {
            set seq = $ORDER(@..#LOGGLOBAL@("byLevel", level, seq))
            quit:seq=""
            set n = n + 1
        }

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(level, n), $LISTBUILD(10, 8), $LISTBUILD("L", "R"))
    }

    write !, "=== log por area ===", !

    set area = ""
    for {
        set area = $ORDER(@..#LOGGLOBAL@("byArea", area))
        quit:area=""

        set n = 0, seq = ""
        for {
            set seq = $ORDER(@..#LOGGLOBAL@("byArea", area, seq))
            quit:seq=""
            set n = n + 1
        }

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(area, n), $LISTBUILD(16, 8), $LISTBUILD("L", "R"))
    }

    quit $$$OK
}

/// Removes entries older than the given number of days.
/// SIMULATION IS THE DEFAULT.
ClassMethod Purge(days As %Integer = 30, reallyDelete As %Boolean = 0) As %Integer
{
    set limit = ##class(LabStudy.DateTime).Today() - days

    write "  ", $SELECT(reallyDelete: "APAGANDO", 1: "simulacao"),
          " entradas com mais de ", days, " dias", !

    set n = 0, seq = ""
    for {
        set seq = $ORDER(@..#LOGGLOBAL@("entry", seq))
        quit:seq=""
        continue:$GET(@..#LOGGLOBAL@("entry", seq, "day")) > limit

        set n = n + 1
        continue:'reallyDelete

        set level = $GET(@..#LOGGLOBAL@("entry", seq, "level"))
        set area = $GET(@..#LOGGLOBAL@("entry", seq, "area"))

        // the indexes go together with the entry, or they become orphans
        kill @..#LOGGLOBAL@("byLevel", level, seq)
        kill @..#LOGGLOBAL@("byArea", area, seq)
        kill @..#LOGGLOBAL@("entry", seq)
    }

    write "  ", n, " entradas", !
    quit n
}

ClassMethod Clear() As %Status
{
    kill @..#LOGGLOBAL
    write "  log limpo", !
    quit $$$OK
}

}
```

`src/LabStudy/Diagnostics.cls`:

```objectscript
/// Diagnostic helpers for the LabStudy system.
Class LabStudy.Diagnostics Extends %RegisteredObject
{

/// Full process context. Useful when recording an error.
ClassMethod Context() As %String
{
    quit "job="_$JOB
         _" user="_$USERNAME
         _" roles="_$ROLES
         _" ns="_$NAMESPACE
         _" when="_##class(LabStudy.DateTime).NowTimestamp()
         _" stack="_$STACK
}

/// Current call stack as a list.
ClassMethod StackTrace() As %List
{
    set out = ""
    for i = 1:1:$STACK {
        set out = out_$LISTBUILD($STACK(i, "PLACE"))
    }
    quit out
}

ClassMethod PrintStack(label As %String = "pilha") As %Status
{
    set stack = ..StackTrace()

    write "  ", label, " (", $LISTLENGTH(stack), " niveis):", !
    for i = 1:1:$LISTLENGTH(stack) {
        write "    ", $JUSTIFY(i, 2), ": ", $LISTGET(stack, i), !
    }
    quit $$$OK
}

/// Detailed dump of a value.
ClassMethod DumpValue(label As %String, value As %String) As %Status
{
    write "  ", label, ":", !
    write "    valor      : [", value, "]", !
    write "    comprimento: ", $LENGTH(value), !
    write "    numerico?  : ", $ISVALIDNUM(value), !
    write "    canonico?  : ", $SELECT(value = +value: "sim", 1: "nao"), !

    set hasControl = 0
    for i = 1:1:$LENGTH(value) {
        set code = $ASCII(value, i)
        if (code < 32) || (code = 127) {
            set hasControl = 1
            write "    !! controle na posicao ", i, " (codigo ", code, ")", !
        }
    }

    if hasControl {
        write "    despejo hexadecimal:", !
        zzdump value
    }

    quit $$$OK
}

/// One health check. Returns the number of problems found.
ClassMethod Check(label As %String, sql As %String, expected As %Integer = 0) As %Integer [ Private ]
{
    new SQLCODE, %msg

    set rs = ##class(%SQL.Statement).%ExecDirect(, sql)

    if rs.%SQLCODE < 0 {
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(label, "ERRO", rs.%Message),
            $LISTBUILD(34, 10, 24), $LISTBUILD("L", "C", "L"))
        quit -1
    }

    set n = 0
    if rs.%Next() {
        set n = +rs.%GetData(1)
    }

    set ok = (n = expected)

    do ##class(LabStudy.Formatter).Row(
        $LISTBUILD(label,
                   $SELECT(ok: "OK", 1: "PROBLEMA"),
                   $SELECT(ok: "", 1: n_" ocorrencias")),
        $LISTBUILD(34, 10, 24), $LISTBUILD("L", "C", "L"))

    quit $SELECT(ok: 0, 1: n)
}

/// Full health check of the system.
ClassMethod SelfCheck() As %Integer
{
    do ##class(LabStudy.Log).Info("diagnostics", "auto verificacao iniciada")

    write "=== auto verificacao do LabStudy ===", !

    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("verificacao", "situacao", "detalhe"),
        $LISTBUILD(34, 10, 24), $LISTBUILD("L", "C", "L"))

    set problems = 0

    set problems = problems + ..Check("pacientes sem numero de registro",
        "SELECT COUNT(*) FROM LabStudy.PATIENT WHERE RecordNumber IS NULL OR RecordNumber = ''")

    set problems = problems + ..Check("pacientes sem nome",
        "SELECT COUNT(*) FROM LabStudy.PATIENT WHERE Name IS NULL OR Name = ''")

    set problems = problems + ..Check("numeros de registro duplicados",
        "SELECT COUNT(*) FROM (SELECT RecordNumber FROM LabStudy.PATIENT "
        _"GROUP BY RecordNumber HAVING COUNT(*) > 1)")

    set problems = problems + ..Check("registros fora do padrao REG-000000",
        "SELECT COUNT(*) FROM LabStudy.PATIENT "
        _"WHERE RecordNumber IS NOT NULL AND RecordNumber NOT LIKE 'REG-______'")

    set problems = problems + ..Check("exames sem paciente",
        "SELECT COUNT(*) FROM LabStudy.EXAM WHERE Patient IS NULL")

    set problems = problems + ..Check("exames sem estado",
        "SELECT COUNT(*) FROM LabStudy.EXAM WHERE ResultStatus IS NULL OR ResultStatus = ''")

    set problems = problems + ..Check("exames finais sem valor",
        "SELECT COUNT(*) FROM LabStudy.EXAM WHERE ResultStatus = 'final' AND ResultValue IS NULL")

    set problems = problems + ..Check("exames pendentes com valor",
        "SELECT COUNT(*) FROM LabStudy.EXAM WHERE ResultStatus = 'pending' AND ResultValue IS NOT NULL")

    set problems = problems + ..Check("nascimentos no futuro",
        "SELECT COUNT(*) FROM LabStudy.PATIENT WHERE BirthDate > "_##class(LabStudy.DateTime).Today())

    do ##class(LabStudy.Formatter).Line(70)

    // schema version, checked separately: it is not a query
    set current = ##class(LabStudy.Schema).Version()
    set latest = ##class(LabStudy.Schema).#LATEST
    set schemaOk = (current = latest)

    do ##class(LabStudy.Formatter).Row(
        $LISTBUILD("versao do esquema",
                   $SELECT(schemaOk: "OK", 1: "PROBLEMA"),
                   current_" de "_latest),
        $LISTBUILD(34, 10, 24), $LISTBUILD("L", "C", "L"))

    set:'schemaOk problems = problems + 1

    do ##class(LabStudy.Formatter).Line(70)

    if problems = 0 {
        write "  nenhum problema encontrado", !
        do ##class(LabStudy.Log).Info("diagnostics", "auto verificacao: tudo OK")
    } else {
        write "  ", problems, " problemas encontrados", !
        do ##class(LabStudy.Log).Warn("diagnostics",
            "auto verificacao encontrou problemas", problems_" ocorrencias")
    }

    quit problems
}

}
```

Instrumente `src/LabStudy/Exam.cls`:

```objectscript
Method SetResult(value As %Numeric, unit As %String = "") As %Status
{
    do ##class(LabStudy.Log).Debug("exam",
        "SetResult chamado para "_..TestCode,
        "valor="_value_" unidade="_unit)

    if value = "" {
        do ##class(LabStudy.Log).Warn("exam",
            "tentativa de lancar resultado vazio em "_..TestCode)

        quit $$$ERROR($$$GeneralError,
            "A result value is required. Use Cancel() to discard the exam.")
    }

    set ..ResultValue = value
    set:unit'="" ..Unit = unit
    set ..ResultStatus = "final"
    set ..ResultDate = ##class(LabStudy.DateTime).NowTimestamp()

    do ##class(LabStudy.Log).Info("exam",
        "resultado lancado: "_..TestCode_" = "_value)

    quit $$$OK
}
```

E o início e o fim de `LabStudy.FileIO.ImportPatientsCsv`:

```objectscript
    do ##class(LabStudy.Log).Info("import",
        "importacao iniciada", "arquivo="_path_" simulacao="_'reallyImport)

    // ... corpo existente ...

    do ##class(LabStudy.Log).Info("import",
        "importacao concluida",
        "criados="_created_" existentes="_skipped_" falhas="_failed)
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "2.1";
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Log).Clear()
  log limpo

LABSTUDY>DO ##class(LabStudy.Log).SetLevel("DEBUG")
  nivel de log definido como DEBUG

LABSTUDY>SET e = ##class(LabStudy.Exam).%OpenId(1)
LABSTUDY>DO e.SetResult(88, "mg/dL")
LABSTUDY>DO $SYSTEM.Status.DisplayError(e.SetResult("", ""))
ERROR #5001: A result value is required. Use Cancel() to discard the exam.

LABSTUDY>DO ##class(LabStudy.Diagnostics).SelfCheck()
=== auto verificacao do LabStudy ===
verificacao                        situacao  detalhe
----------------------------------------------------------------------
pacientes sem numero de registro      OK
pacientes sem nome                    OK
numeros de registro duplicados        OK
registros fora do padrao REG-000000   OK
exames sem paciente                   OK
exames sem estado                     OK
exames finais sem valor               OK
exames pendentes com valor            OK
nascimentos no futuro                 OK
----------------------------------------------------------------------
versao do esquema                     OK        5 de 5
----------------------------------------------------------------------
  nenhum problema encontrado

LABSTUDY>DO ##class(LabStudy.Log).Show(10)
quando               nivel  area         mensagem
------------------------------------------------------------------------------------
2026-08-19 17:22:04  INFO   diagnostics  auto verificacao: tudo OK
2026-08-19 17:22:03  INFO   diagnostics  auto verificacao iniciada
2026-08-19 17:21:50  WARN   exam         tentativa de lancar resultado vazio em GLU
2026-08-19 17:21:50  DEBUG  exam         SetResult chamado para GLU
2026-08-19 17:21:44  INFO   exam         resultado lancado: GLU = 88
2026-08-19 17:21:44  DEBUG  exam         SetResult chamado para GLU
------------------------------------------------------------------------------------
  6 entradas (nivel atual: DEBUG)

LABSTUDY>DO ##class(LabStudy.Log).SetLevel("WARN")
  nivel de log definido como WARN

LABSTUDY>DO e.SetResult(90, "mg/dL")
LABSTUDY>DO ##class(LabStudy.Log).Show(10)
quando               nivel  area         mensagem
------------------------------------------------------------------------------------
2026-08-19 17:23:11  INFO   diagnostics  auto verificacao: tudo OK
...
   (a chamada com nivel WARN nao registrou o DEBUG nem o INFO)
```

**Por que cada decisão:**

- **O nível é verificado ANTES de gravar qualquer coisa.** A linha `quit:..Weight(level) < ..Weight(..GetLevel())` é o que torna viável deixar chamadas `Debug()` espalhadas pelo código: em operação normal, elas custam uma leitura de global e um `quit`. **É isso que resolve o dilema entre instrumentar e não encher o disco.**
- **A instrumentação ficou no código permanentemente.** Compare com o padrão comum de acrescentar `write` durante a investigação e removê-los depois: aquele padrão garante que, na próxima vez, você começará do zero.
- **O log tem índices por nível e por área.** Filtrar sem varrer tudo é o Capítulo 13 aplicado a uma estrutura auxiliar.
- **As entradas ficam sob `("entry", seq, campo)`, e os índices sob `("byLevel", ...)` e `("byArea", ...)`.** Essa separação não é organização estética: é o que torna a leitura reversa do `Show` correta. Se os números de sequência e os subscritos `"byLevel"` e `"byArea"` convivessem no mesmo nível, um `$ORDER(..., -1)` devolveria **primeiro os índices**, porque texto ordena depois de número (Capítulo 13). Daria para contornar com um `continue` que os pula — e funcionaria —, mas seria uma estrutura que só está correta por causa de um remendo. **Quando dois tipos de subscrito convivem num nível, quase sempre eles deveriam estar em ramos diferentes.**
- **O contador também saiu da raiz**, para `("counter")`. Guardá-lo na raiz funcionaria, mas a raiz de uma global com ramos nomeados é justamente onde alguém vai esquecer que existe alguma coisa.
- **`Show` percorre com `$ORDER(..., -1)`**, do mais recente para o mais antigo. É a ordem que interessa num log — e sai de graça, sem ordenação.
- **`Error()` acrescenta o contexto automaticamente**, incluindo `$ZERROR` quando presente. Quem registra o erro não precisa lembrar de capturar o contexto.
- **`SelfCheck` transforma cada verificação numa consulta que deve devolver zero.** Esse formato uniforme permite acrescentar verificações com uma linha, e produz um relatório legível sem lógica especial por item.
- **As nove verificações traduzem, em consultas, as regras aprendidas na apostila:** obrigatoriedade (Capítulo 2), unicidade (Capítulo 2), formato (Capítulo 15), coerência entre estado e valor (Capítulo 10), integridade referencial (Capítulo 2) e validação de data (Capítulo 16). **Um `SelfCheck` é a materialização do modelo de dados como código executável** — e roda em segundos, a qualquer momento.
- **A verificação de versão do esquema é feita fora do padrão das consultas**, porque não é uma consulta. Forçá-la no mesmo formato produziria código pior.
- **`Purge` simula por padrão**, como todas as operações destrutivas desta apostila.

---

## 9. Quiz do capítulo

**Q1.** Qual é o primeiro passo de um diagnóstico?

- A) Ler o código procurando o erro.
- B) Reproduzir o problema.
- C) Acrescentar pontos de parada.
- D) Consultar o log.

---

**Q2.** Qual comando mostra **todas** as variáveis locais do contexto atual?

- A) `WRITE`
- B) `ZWRITE` sem argumento
- C) `ZZDUMP`
- D) `$STACK`

---

**Q3.** `"abc" = "abc"` devolveu `0`. Qual ferramenta usar?

- A) `ZBREAK`
- B) `ZZDUMP`, para ver os bytes e encontrar o caractere invisível.
- C) `$ECODE`
- D) `$STACK`

---

**Q4.** O que `$ZERROR` contém?

- A) Todos os erros do processo.
- B) O texto e a localização do **último** erro do processo.
- C) A pilha de chamadas.
- D) O código de erro numérico apenas.

---

**Q5.** Por que ler `$ZERROR` tarde pode enganar?

- A) Ela é apagada automaticamente.
- B) Ela guarda o último erro, que pode ser outro ocorrido depois.
- C) Ela só funciona fora de `try`/`catch`.
- D) Ela não tem localização.

---

**Q6.** O que `$STACK` indica?

- A) A quantidade de memória usada.
- B) O nível atual da pilha de chamadas.
- C) O último erro.
- D) O número do processo.

---

**Q7.** Qual a diferença entre `$STACK` e `$ESTACK`?

- A) Nenhuma.
- B) `$STACK` é absoluto; `$ESTACK` conta a partir de um marco definido com `NEW $ESTACK`.
- C) `$ESTACK` conta erros.
- D) `$STACK` só funciona em rotinas.

---

**Q8.** Dentro de um `catch`, a pilha obtida com `$STACK` é a do momento do erro?

- A) Sim, sempre.
- B) Não: os níveis abaixo já foram desfeitos. A pilha do erro vem do objeto de exceção.
- C) Só se você usar `NEW $ESTACK`.
- D) Só em métodos de classe.

---

**Q9.** O comando `BREAK` funciona em processo de segundo plano?

- A) Sim, normalmente.
- B) Não como esperado: ele exige contexto interativo.
- C) Só com `ZBREAK` antes.
- D) Só no namespace `%SYS`.

---

**Q10.** Como remover todos os pontos de parada definidos com `ZBREAK`?

- A) `ZBREAK OFF`
- B) `ZBREAK *`
- C) `ZBREAK CLEAR`
- D) Reiniciar o processo.

---

**Q11.** Qual rotina registra o erro atual no log de erros da aplicação?

- A) `LOG^%ETN`
- B) `##class(%SYS.System).WriteToConsoleLog()`
- C) `$SYSTEM.Status.DisplayError()`
- D) `ZWRITE ^ERRORS`

---

**Q12.** Onde consultar os erros registrados pela aplicação, no Portal?

- A) System Operation → Processes
- B) System Operation → System Logs → Application Error Log
- C) System Explorer → Globals
- D) System Administration → Security

---

**Q13.** Qual é o nível de observação correto para um problema intermitente que só ocorre em produção de madrugada?

- A) Pontos de parada.
- B) Log com contexto.
- C) Rastreamento ligado.
- D) Depurador do VS Code.

---

**Q14.** Por que controlar a verbosidade do log por configuração?

- A) Para o código ficar menor.
- B) Para poder deixar a instrumentação permanentemente no código, ligando-a quando necessário sem recompilar.
- C) Porque o IRIS exige.
- D) Para economizar memória.

---

**Q15.** Qual é a primeira tela do Portal a consultar quando algo está travado?

- A) Globals
- B) System Operation → Processes
- C) SQL → Show Plan
- D) Databases

---

**Q16.** O que faz o monitor `^%SYS.MONLBL`?

- A) Mostra o plano de uma consulta.
- B) Mede o tempo gasto em cada **linha** das rotinas selecionadas.
- C) Lista os processos.
- D) Registra erros.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** um problema reproduzível é um problema resolúvel. Sem reprodução, nenhuma ferramenta ajuda e não há como verificar a correção.
- **A, C e D estão erradas:** são passos posteriores, e todos dependem de você conseguir provocar o problema.

**Q2 — Resposta: B.**
- **B está certa:** `ZWRITE` sem argumento despeja todas as variáveis locais do contexto.
- **A está errada:** `WRITE` exige um valor.
- **C está errada:** `ZZDUMP` mostra bytes de um valor.
- **D está errada:** `$STACK` trata da pilha.

**Q3 — Resposta: B.**
- **B está certa:** `ZZDUMP` revela caracteres invisíveis, que são a causa quase certa.
- **A, C e D estão erradas:** não mostram o conteúdo byte a byte.

**Q4 — Resposta: B.**
- **B está certa:** `$ZERROR` traz mensagem e localização do último erro do processo.
- **A está errada:** guarda apenas o último.
- **C está errada:** a pilha se obtém com `$STACK`.
- **D está errada:** o código está em `$ECODE`.

**Q5 — Resposta: B.**
- **B está certa:** ela é sobrescrita por qualquer erro posterior, inclusive tratados.
- **A está errada:** ela não se apaga sozinha.
- **C e D estão erradas:** funciona nos dois contextos e traz localização.

**Q6 — Resposta: B.**
- **B está certa:** `$STACK` é o nível atual da pilha de chamadas.
- **A, C e D estão erradas:** são outras informações.

**Q7 — Resposta: B.**
- **B está certa:** `NEW $ESTACK` define um marco, e `$ESTACK` passa a contar a partir dele.
- **A está errada:** o comportamento difere.
- **C está errada:** ele conta níveis, não erros.
- **D está errada:** funciona em classes também.

**Q8 — Resposta: B.**
- **B está certa:** quando a exceção sobe até o `catch`, os níveis abaixo já foram desfeitos. A pilha do erro precisa vir do objeto de exceção.
- **A está errada:** é a suposição intuitiva e errada.
- **C e D estão erradas:** não mudam esse comportamento.

**Q9 — Resposta: B.**
- **B está certa:** `BREAK` devolve o controle a um usuário interativo; sem um, não faz o esperado.
- **A está errada:** justamente o oposto.
- **C e D estão erradas:** não resolvem a ausência de contexto interativo.

**Q10 — Resposta: B.**
- **B está certa:** `ZBREAK *` limpa todos os pontos de parada.
- **A, C e D estão erradas:** não são a forma correta.

**Q11 — Resposta: A.**
- **A está certa:** `LOG^%ETN` grava o erro atual em `^ERRORS`, com contexto completo.
- **B está errada:** escreve no log do sistema, sem o contexto do erro.
- **C está errada:** apenas exibe na tela.
- **D está errada:** é leitura, não gravação.

**Q12 — Resposta: B.**
- **B está certa:** é a tela do log de erros da aplicação.
- **A, C e D estão erradas:** mostram outras informações.

**Q13 — Resposta: B.**
- **B está certa:** problemas que ocorrem sem você presente exigem registro com contexto.
- **A, C e D estão erradas:** exigem alguém acompanhando no momento da falha.

**Q14 — Resposta: B.**
- **B está certa:** com níveis, a instrumentação pode ficar no código e ser ligada quando necessário, sem alterar nem recompilar.
- **A está errada:** o código fica maior, e vale a pena.
- **C está errada:** não é exigência do produto.
- **D está errada:** o ganho é de diagnóstico, não de memória.

**Q15 — Resposta: B.**
- **B está certa:** a tela de processos mostra o que cada processo executa agora, e permite examiná-los.
- **A, C e D estão erradas:** úteis para outras perguntas.

**Q16 — Resposta: B.**
- **B está certa:** o monitor linha a linha mede o tempo por linha de código.
- **A está errada:** isso é o Show Plan.
- **C e D estão erradas:** são outras ferramentas.

---

## 10. Resumo relâmpago

1. **Método antes de ferramenta:** reproduzir → observar → isolar → corrigir → verificar. **Sem reprodução, nada funciona.**
2. **Uma mudança por vez.** Várias ao mesmo tempo impedem saber o que resolveu.
3. Quatro níveis de observação: **estado** (`ZWRITE`), **parar e olhar** (`BREAK`), **registrar** (log) e **medir** (plano, monitor).
4. **`ZWRITE`** sem argumento despeja todas as variáveis locais; com argumento, a subárvore. Mostra aspas em texto e não em número.
5. **`ZZDUMP`** mostra os bytes em hexadecimal. É a resposta para `"a" = "a"` devolvendo `0`.
6. **`$ZERROR`** traz mensagem e localização do **último** erro do processo; **`$ECODE`** traz os códigos.
7. **`$ZERROR` pode estar desatualizada.** Dentro de um `catch`, prefira o objeto de exceção.
8. **`$STACK`** é o nível atual; **`$STACK(n, "PLACE")`** e **`"MCODE"`** revelam cada nível; **`$ESTACK`** conta a partir de um `NEW $ESTACK`.
9. **No `catch`, a pilha já foi desfeita.** A pilha do erro vem do objeto de exceção.
10. **`BREAK`** pausa em contexto **interativo**. Nunca deixe em produção.
11. **`ZBREAK`** define pontos de parada **sem alterar o código**; **`ZBREAK *`** limpa todos.
12. **Rastreamento** exibe cada linha executada. Use em trechos curtos.
13. **`LOG^%ETN`** registra o erro atual em `^ERRORS`, com variáveis, pilha, usuário e horário. Consulte no Portal.
14. **`messages.log`** é do administrador. Use uma trilha própria para a aplicação.
15. **Log com níveis** permite deixar a instrumentação **permanentemente no código**, ligando-a por configuração.
16. Portal: **Processes** (travamentos), **Locks**, **Journals**, **System Logs**, **SQL → Show Plan**, **Globals**.
17. Desempenho: **`^%SYS.MONLBL`** mede linha a linha; **`^SystemPerformance`** produz relatório do sistema.
18. **Um `SelfCheck`** que traduz as regras do modelo em consultas verifica a saúde do sistema em segundos.
19. Problemas intermitentes de produção exigem **log**, não depurador.

---

## 11. Cartões de memorização

**Frente:** Qual é o primeiro passo de um diagnóstico?
**Verso:** Reproduzir o problema. Sem reprodução, nenhuma ferramenta ajuda e não há como verificar a correção.

**Frente:** Como ver todas as variáveis locais de uma vez?
**Verso:** `ZWRITE` sem argumento.

**Frente:** `"abc" = "abc"` devolveu 0. O que fazer?
**Verso:** `ZZDUMP` nos dois valores — há um caractere invisível.

**Frente:** O que `$ZERROR` contém?
**Verso:** O texto e a localização do **último** erro do processo.

**Frente:** Por que não confiar em `$ZERROR` lida tarde?
**Verso:** Porque qualquer erro posterior, mesmo tratado, a sobrescreve.

**Frente:** Como ver a pilha de chamadas?
**Verso:** `$STACK` dá o nível; `$STACK(n, "PLACE")` e `$STACK(n, "MCODE")` dão cada nível.

**Frente:** Diferença entre `$STACK` e `$ESTACK`.
**Verso:** `$STACK` é absoluto. `$ESTACK` conta a partir de um marco definido com `NEW $ESTACK`.

**Frente:** No `catch`, `$STACK` mostra a pilha do erro?
**Verso:** Não. Os níveis abaixo já foram desfeitos. A pilha do erro vem do objeto de exceção.

**Frente:** O que faz `BREAK`?
**Verso:** Pausa a execução e devolve o controle ao Terminal — **apenas em contexto interativo**. Nunca em produção.

**Frente:** Como definir um ponto de parada sem alterar o código?
**Verso:** `ZBREAK`. E `ZBREAK *` remove todos.

**Frente:** Qual rotina registra o erro atual com contexto completo?
**Verso:** `LOG^%ETN`, que grava em `^ERRORS`.

**Frente:** Onde ver os erros registrados?
**Verso:** Portal → System Operation → System Logs → Application Error Log.

**Frente:** Qual nível de observação usar para problema intermitente de produção?
**Verso:** **Log com contexto**. Pontos de parada exigem alguém presente no momento da falha.

**Frente:** Por que log com níveis?
**Verso:** Permite deixar a instrumentação permanentemente no código, ligando-a por configuração sem recompilar.

**Frente:** Qual a primeira tela do Portal quando algo está travado?
**Verso:** System Operation → Processes. Depois, Locks.

**Frente:** O que faz `^%SYS.MONLBL`?
**Verso:** Mede o tempo gasto em cada **linha** das rotinas selecionadas.

**Frente:** O que é um `SelfCheck`?
**Verso:** Um conjunto de consultas que traduzem as regras do modelo e devem devolver zero. Verifica a saúde do sistema em segundos.

**Frente:** Quantas coisas mudar por vez ao corrigir?
**Verso:** Uma. Senão você não sabe qual resolveu.

---

Digite CONTINUAR para o próximo capítulo.
