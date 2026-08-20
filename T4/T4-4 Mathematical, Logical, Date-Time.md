# Apostila InterSystems ObjectScript Specialist
## Capítulo 16 — T4.4 Mathematical, Logical, Date/Time (Matemática, lógica, data e hora)

> Ainda em **T4 — Functions & APIs**. Este capítulo tem uma surpresa logo no início: a regra de avaliação de expressões do ObjectScript é **diferente da matemática que você aprendeu na escola**, e ignorar isso produz erros de cálculo silenciosos. Depois disso, o resto é conhecer funções.

---

## 1. O que você vai saber fazer ao terminar

1. Aplicar a regra de **avaliação estritamente da esquerda para a direita** e usar parênteses corretamente.
2. Usar os operadores aritméticos, incluindo **`\`** (divisão inteira), **`#`** (resto) e **`**`** (potência).
3. Usar as funções matemáticas: `$ZABS`, `$ZSQR`, `$ZPOWER`, `$ZLN`, `$ZLOG`, `$RANDOM`.
4. Arredondar e validar números com **`$NUMBER`**, **`$NORMALIZE`**, **`$ISVALIDNUM`**, `$JUSTIFY` e `$FNUMBER`.
5. Entender a diferença entre aritmética **decimal** e de **ponto flutuante** (`$DOUBLE`, `$DECIMAL`).
6. Usar os operadores lógicos **`'`**, **`&`**, **`!`**, **`&&`**, **`||`** e entender **curto-circuito**.
7. Escolher entre **`$SELECT`** e **`$CASE`**.
8. Entender o formato **`$HOROLOG`** e sua origem, e usar **`$ZTIMESTAMP`** e **`$NOW`**.
9. Converter datas e horas com **`$ZDATE`**, **`$ZDATEH`**, **`$ZTIME`**, **`$ZTIMEH`**, **`$ZDATETIME`** e **`$ZDATETIMEH`**.
10. Fazer **aritmética de datas**: somar dias, calcular diferenças, descobrir o dia da semana.
11. Calcular **idade corretamente** — e corrigir o atalho errado usado nos capítulos anteriores.
12. Tratar **fuso horário**, **anos bissextos** e **fim de mês**.
13. Levar o projeto à versão **1.7**, com uma camada de data e hora confiável.

---

## 2. O conceito em linguagem de gente

### 2.1 A regra que muda tudo: esquerda para a direita

Quanto vale `2 + 3 * 4`?

Na matemática da escola, 14: a multiplicação vem antes.

**No ObjectScript, vale 20.**

Porque o ObjectScript avalia **estritamente da esquerda para a direita**, sem hierarquia entre operadores. Ele lê: "2 mais 3 dá 5; 5 vezes 4 dá 20".

Isso não é um defeito nem um descuido: é uma decisão de projeto da linguagem, herdada de sua tradição. E ela vale para **todos** os operadores — aritméticos, de comparação e lógicos.

A analogia é a de uma fila de caixa: cada operação é atendida na ordem em que chega, sem ninguém furar a fila por ser "mais importante".

Consequências práticas:

```objectscript
write 2 + 3 * 4, !                   // 20, não 14
write 10 - 2 - 3, !                  // 5   (igual à matemática, por sorte)
write 2 + 3 * 4 / 2, !               // 10  ((2+3)*4/2)
write 1 + 1 = 2, !                   // 0   ((1+1)=2 dá 1... ou não?)
```

A última merece atenção especial. `1 + 1 = 2` é lido como `(1 + 1) = 2`, ou seja, `2 = 2`, que é `1`. Então por que escrevi `0`? Porque **não** é isso que acontece:

Na verdade `1 + 1 = 2` avalia `1 + 1` → `2`, depois `2 = 2` → `1`. O resultado **é** `1`. Corrigindo o exemplo acima: ele imprime `1`.

Mas veja este:

```objectscript
write 1 + 1 = 2 + 5, !
```

Da esquerda para a direita: `1 + 1` → `2`; `2 = 2` → `1`; `1 + 5` → `6`. **Imprime 6**, não `0` nem `1`. Na matemática convencional, você leria "1+1 é igual a 2+5?", que seria falso.

**A regra de sobrevivência é simples e absoluta:**

> **Use parênteses. Sempre. Em toda expressão com mais de um operador.**

Isso não é preciosismo: é o que separa código que funciona de código que funciona por acaso.

### 2.2 Números decimais e números de ponto flutuante

O ObjectScript trabalha, por padrão, com aritmética **decimal** de alta precisão. Isso é raro em linguagens de programação e é uma vantagem enorme para sistemas financeiros e clínicos.

A diferença aparece no clássico:

```objectscript
write 0.1 + 0.2, !                   // 0.3   -- exato
```

Em linguagens que usam ponto flutuante binário por padrão, essa conta produz algo como `0.30000000000000004`. No ObjectScript, com decimal, dá `0.3` exato.

Quando você **quer** ponto flutuante — por desempenho em cálculos científicos, ou por compatibilidade —, usa `$DOUBLE`:

```objectscript
write $DOUBLE(0.1) + $DOUBLE(0.2), !     // aparece a imprecisão
```

Analogia: decimal é contar com notas e moedas — dá para representar exatamente R$ 0,10. Ponto flutuante binário é medir com uma régua de frações de polegada — algumas medidas nunca caem exatamente numa marca.

**Regra:** para dinheiro, quantidades e resultados clínicos, **fique no decimal padrão**. Use `$DOUBLE` apenas quando souber por que precisa dele.

### 2.3 Verdadeiro e falso

Já visto no Capítulo 10, mas vale consolidar:

> **Zero é falso. String vazia é falsa. Todo o resto é verdadeiro.**

E há um detalhe importante: quando um **texto** é usado em contexto lógico, o ObjectScript o interpreta **numericamente**, pegando o número que está no começo dele:

```objectscript
write $SELECT("abc": "V", 1: "F"), !          // F  -- "abc" vale 0
write $SELECT("0abc": "V", 1: "F"), !         // F  -- começa com 0
write $SELECT("1abc": "V", 1: "F"), !         // V  -- começa com 1
```

Isso surpreende. `"1abc"` é verdadeiro porque sua interpretação numérica é `1`. Nunca confie na avaliação lógica de um texto que possa começar com dígito: compare explicitamente.

### 2.4 O tempo, contado desde 1840

O IRIS representa datas como **um número de dias**, contados a partir de um dia zero fixo.

Esse dia zero é **31 de dezembro de 1840**. Não é uma escolha arbitrária: vem da tradição da linguagem, que precisava representar datas de nascimento de pessoas vivas no século XIX.

E o horário é **um número de segundos desde a meia-noite**, de 0 a 86399.

Juntando os dois, temos o **`$HOROLOG`**:

```
67352,52011
   |     |
   |     +-- segundos desde a meia-noite (14:26:51)
   +-------- dias desde 31/12/1840
```

Por que guardar assim, em vez de um texto legível?

Porque **contas ficam triviais**. Somar 30 dias é somar 30. Descobrir quantos dias há entre duas datas é subtrair. Ordenar é comparar números. Nenhuma dessas operações seria simples com o texto `"19/08/2026"`.

Analogia: é a mesma razão pela qual um agrônomo conta a idade de uma árvore em anéis, e não em "primavera de 1998". O número é operável.

### 2.5 Três marcadores de tempo, três finalidades

| Variável | O que é | Quando usar |
|---|---|---|
| **`$HOROLOG`** | data e hora **locais**, precisão de 1 segundo | uso geral, carimbos do dia a dia |
| **`$ZTIMESTAMP`** | data e hora em **UTC**, com fração de segundo | registros que atravessam fusos, ordenação fina |
| **`$NOW()`** | data e hora **locais**, com fração de segundo | medições e carimbos de alta resolução |

A distinção **local × UTC** importa mais do que parece. Um sistema que atende filiais em fusos diferentes, ou que roda em nuvem, produz confusão se misturar os dois. E o horário de verão faz um mesmo horário local acontecer duas vezes no ano em que ele termina.

**Regra profissional:** grave em **UTC**, exiba em **local**. Você já viu o mesmo princípio no Capítulo 15 com formatação: guarde o valor bruto, formate na apresentação.

---

## 3. Matemática

### 3.1 Operadores

| Operador | Faz | Exemplo |
|---|---|---|
| `+` | soma | `7 + 2` = 9 |
| `-` | subtração | `7 - 2` = 5 |
| `*` | multiplicação | `7 * 2` = 14 |
| `/` | divisão | `7 / 2` = 3.5 |
| `\` | **divisão inteira** | `7 \ 2` = 3 |
| `#` | **resto (módulo)** | `7 # 2` = 1 |
| `**` | **potência** | `7 ** 2` = 49 |

`\` e `#` são a dupla clássica para dividir em partes:

```objectscript
set totalSegundos = 52011
set horas = totalSegundos \ 3600                  // 14
set minutos = (totalSegundos # 3600) \ 60         // 26
set segundos = totalSegundos # 60                 // 51
```

Repare nos parênteses em `(totalSegundos # 3600) \ 60`. Sem eles, a leitura da esquerda para a direita faria `totalSegundos # 3600` e depois `\ 60` — que por acaso é o que queremos. Mas escrever os parênteses torna a intenção explícita e imune à ordem.

O operador `#` com números negativos e com decimais tem comportamento específico: **verificar na documentação oficial** quando esses casos importarem.

### 3.2 Funções matemáticas

```objectscript
write $ZABS(-7.5), !                 // 7.5     valor absoluto
write $ZSQR(144), !                  // 12      raiz quadrada
write $ZPOWER(2, 10), !              // 1024    potência
write $ZLN(2.718281828), !           // ~1      logaritmo natural
write $ZLOG(1000), !                 // 3       logaritmo decimal
write $ZEXP(1), !                    // ~2.718  exponencial
```

Há também as trigonométricas `$ZSIN`, `$ZCOS`, `$ZTAN`, `$ZARCTAN`, e outras. A lista completa e a precisão de cada uma: **verificar na documentação oficial**.

**`$RANDOM(n)`** devolve um inteiro de **0 a n-1**:

```objectscript
write $RANDOM(6), !                  // 0 a 5
write $RANDOM(6) + 1, !              // 1 a 6  -- um dado
```

**Atenção:** `$RANDOM` é adequado para simulações e amostragens, **não para segurança**. Para chaves, tokens e sal, use `$SYSTEM.Encryption.GenCryptRand()`, como visto no Capítulo 7.

### 3.3 Arredondamento e validação numérica

Existem quatro ferramentas com finalidades diferentes:

```objectscript
write $NUMBER(3.7, 0), !                 // 4       arredonda para inteiro
write $NUMBER(3.14159, "", 2), !         // 3.14    -- ver observação abaixo
write $NORMALIZE(3.14159, 2), !          // 3.14    arredonda para n casas
write $JUSTIFY(3.14159, 10, 2), !        //     3.14  arredonda E alinha
write $FNUMBER(3.14159, "", 2), !        // 3.14    arredonda E formata
```

- **`$NORMALIZE(numero, casas)`** — arredonda para o número de casas. É a função mais direta para "quero 2 casas decimais".
- **`$NUMBER(valor, formato, min, max)`** — converte **e valida**. Devolve vazio se o valor não for um número aceitável ou estiver fora da faixa. Os códigos de formato e a ordem exata dos argumentos variam: **verificar na documentação oficial**.
- **`$JUSTIFY`** e **`$FNUMBER`** — vistos no Capítulo 15; arredondam **e** formatam para exibição.

**`$ISVALIDNUM`** verifica se um valor é numérico, opcionalmente dentro de uma faixa:

```objectscript
write $ISVALIDNUM("12.5"), !             // 1
write $ISVALIDNUM("abc"), !              // 0
write $ISVALIDNUM("12.5", 1), !          // 1  -- tem 1 casa, e o limite é 1
write $ISVALIDNUM("12.55", 1), !         // 0  -- tem 2 casas, e o limite é 1
write $ISVALIDNUM("50", , 0, 100), !     // 1  -- dentro da faixa 0 a 100
write $ISVALIDNUM("150", , 0, 100), !    // 0  -- fora da faixa
```

O segundo argumento é o **número máximo de casas decimais** aceito. Ele não é uma exigência de ter aquelas casas: é um teto.

Compare com o operador de padrão do Capítulo 15: `?1.N` valida **formato** de dígitos; `$ISVALIDNUM` valida **valor numérico**, aceitando sinal, decimais e notação. Para validar um resultado de exame, `$ISVALIDNUM` é geralmente a escolha certa.

### 3.4 Um cuidado com a divisão

```objectscript
write 10 / 0, !
```

Divisão por zero causa **`<DIVIDE>`**. Sempre proteja:

```objectscript
set media = $SELECT(quantidade > 0: total / quantidade, 1: "")
```

Note que devolver **vazio** (e não zero) quando não há dados é a decisão correta segundo o Capítulo 10: "média de nada" é desconhecida, não é zero.

---

## 4. Lógica

### 4.1 Operadores lógicos

| Operador | Significado | Curto-circuito |
|---|---|---|
| `'` | **não** | — |
| `&` | **e** | não |
| `!` | **ou** | não |
| `&&` | **e** | **sim** |
| `||` | **ou** | **sim** |

**Curto-circuito** significa parar de avaliar assim que o resultado já está decidido:

```objectscript
if (obj '= "") && (obj.Name = "Maria") { ... }
```

Com `&&`, se `obj` for vazio, a segunda condição **não é avaliada** — e é isso que evita o erro `<INVALID OREF>`.

Com `&` (sem curto-circuito), **as duas condições são avaliadas**, e o código quebra.

**Regra prática:** use **`&&`** e **`||`** quando a segunda condição depender da primeira ser verdadeira. Na dúvida, use-os sempre — eles nunca fazem mal.

Atenção ao símbolo `!`: no ObjectScript ele é **"ou"** em expressões lógicas, mas dentro de um `WRITE` significa **quebra de linha**. Dois papéis, mesmo caractere, contextos diferentes.

### 4.2 A regra da esquerda para a direita vale aqui também

```objectscript
write 1 ! 0 & 0, !
```

Da esquerda para a direita: `1 ! 0` → `1`; `1 & 0` → `0`. **Resultado: 0.**

Na lógica convencional, "e" tem precedência sobre "ou", então seria `1 ! (0 & 0)` = `1 ! 0` = `1`.

Mais uma vez: **parênteses**.

### 4.3 Comparações

| Operador | Significado |
|---|---|
| `=` | igual |
| `'=` | diferente |
| `<` `>` | menor / maior (**numérico**) |
| `<=` `>=` | menor ou igual / maior ou igual |
| `[` | contém (texto) |
| `]` `]]` | ordem (Capítulo 15) |

Lembrete do Capítulo 10: `=` compara **numericamente** quando ambos os lados são números canônicos, e **como texto** caso contrário. Por isso `"10" = 10` é verdadeiro e `"10.0" = 10` é falso.

### 4.4 `$SELECT` e `$CASE`

**`$SELECT`** avalia condições em ordem e devolve o valor da **primeira verdadeira**:

```objectscript
set faixa = $SELECT(valor < 70: "baixo",
                    valor > 99: "alto",
                    1: "normal")
```

- O `1:` final é o "senão". **Sem ele, se nenhuma condição for verdadeira, ocorre `<ILLEGAL VALUE>`.**
- `$SELECT` tem **curto-circuito**: as condições seguintes à primeira verdadeira não são avaliadas.

**`$CASE`** compara **um valor** contra alternativas:

```objectscript
set nome = $CASE(codigo, "M": "Masculino",
                         "F": "Feminino",
                         : "Não informado")
```

- O último ramo, **sem valor antes dos dois-pontos**, é o padrão.
- Diferente do `$SELECT`, aqui não se escrevem condições, e sim **valores a comparar**.

Quando usar cada um:

- **`$CASE`** quando você compara **a mesma variável** contra vários valores. Mais legível.
- **`$SELECT`** quando as condições são **diferentes entre si** (faixas, combinações, testes distintos).

---

## 5. Data e hora

### 5.1 Lendo o relógio

```
LABSTUDY>WRITE $HOROLOG, !
67352,52011

LABSTUDY>WRITE $ZTIMESTAMP, !
67352,62811.4372

LABSTUDY>WRITE $NOW(), !
67352,52011.7284

LABSTUDY>WRITE +$HOROLOG, !
67352

LABSTUDY>WRITE $PIECE($HOROLOG, ",", 2), !
52011
```

- **`+$HOROLOG`** força a leitura numérica e devolve **só os dias** — é a forma idiomática de obter "hoje" no formato lógico de `%Date`.
- **`$PIECE($HOROLOG, ",", 2)`** devolve os segundos — o formato lógico de `%Time`.
- Note que `$ZTIMESTAMP` mostra segundos diferentes de `$HOROLOG`: ele está em **UTC**, e o `$HOROLOG` está no horário local.

**`$ZTIMEZONE`** informa o deslocamento do fuso, em minutos:

```
LABSTUDY>WRITE $ZTIMEZONE, !
180
```

O sinal e a convenção exatos: **verificar na documentação oficial**.

### 5.2 Convertendo data

**`$ZDATE(dias, formato)`** — do número para texto:

| Formato | Resultado para 19/08/2026 |
|---|---|
| `1` | `08/19/2026` (mês/dia/ano) |
| `2` | `19 Aug 2026` |
| `3` | `2026-08-19` (**ODBC**) |
| `4` | `19/08/2026` (dia/mês/ano) |
| `8` | `20260819` |

**`$ZDATEH(texto, formato)`** — do texto para o número, usando o mesmo código de formato.

```objectscript
set dias = $ZDATEH("2026-08-19", 3)
write dias, !                                // 67352
write $ZDATE(dias, 3), !                     // 2026-08-19
write $ZDATE(dias, 4), !                     // 19/08/2026
```

**Recomendação forte:** use sempre o **formato 3 (ODBC)** dentro do sistema. Ele é inequívoco, ordena corretamente como texto, e é o formato de intercâmbio universal. Os formatos 1 e 4 são ambíguos entre si (`03/04/2026` é março ou abril?) e devem aparecer apenas na apresentação final, conforme a localidade do usuário.

Existem mais códigos de formato, incluindo formatos por extenso e dependentes de localidade: **verificar na documentação oficial**.

> ### ⚠️ Um erro que esta apostila cometeu
>
> Nas primeiras versões dos capítulos 2 e 9, o cálculo de idade estava escrito
> como `$ZDATE($HOROLOG, 4) - $ZDATE(..BirthDate, 4)`, acompanhado da
> explicação de que o formato `4` devolveria "só o ano".
>
> **Isso estava errado**, e por dois motivos encadeados. O formato `4` devolve
> a data **completa** em `DD/MM/AAAA`; a subtração daqueles dois textos produz
> um número sem significado algum — algo como `19/08/2026 - 17/05/1990`, que o
> ObjectScript avalia numericamente como `19 - 17`.
>
> Aqueles capítulos já foram corrigidos e hoje trazem a mesma fórmula da seção
> 5.5. Mantenho o registro do episódio porque ele ilustra dois pontos do
> Capítulo 22 melhor do que qualquer exemplo inventado:
>
> - **O erro não gerava mensagem.** Um número saía, parecia plausível, e
>   ninguém conferiu.
> - **Ele sobreviveu porque a explicação errada acompanhava o código errado.**
>   Quem lesse os dois juntos não teria motivo para desconfiar.
>
> A defesa contra isso não é tratamento de erro: é **verificação**. Um único
> teste com uma data de nascimento no mês corrente teria derrubado a fórmula
> na primeira execução.

### 5.3 Convertendo hora

```objectscript
write $ZTIME(52011), !                   // 14:26:51
write $ZTIME(52011, 2), !                // 14:26
write $ZTIMEH("14:26:51"), !             // 52011
```

**`$ZTIME(segundos, formato)`** e **`$ZTIMEH(texto, formato)`** funcionam como o par de data. Os códigos de formato disponíveis (12 horas, com ou sem segundos, com fração): **verificar na documentação oficial**.

### 5.4 Data e hora juntas

```objectscript
write $ZDATETIME($HOROLOG, 3), !                 // 2026-08-19 14:26:51
write $ZDATETIMEH("2026-08-19 14:26:51", 3), !   // 67352,52011
```

**`$ZDATETIME(horolog, formatoData, formatoHora)`** combina os dois. Com formato de data `3`, produz exatamente o **formato lógico de `%TimeStamp`** — que é o que você vem usando desde o Capítulo 2 nos `InitialExpression`.

### 5.5 Aritmética de datas

Como datas são números, as contas são diretas:

```objectscript
set hoje = +$HOROLOG

set amanha = hoje + 1
set semanaPassada = hoje - 7
set daquiA30 = hoje + 30

set diferenca = $ZDATEH("2026-12-25", 3) - hoje
write "faltam ", diferenca, " dias para o Natal", !
```

**Diferença em dias entre duas datas** é uma subtração. Sem bibliotecas, sem cuidados com meses de tamanhos diferentes, sem anos bissextos — porque tudo isso já está embutido na contagem.

**Idade em anos completos** é diferente e exige cuidado, porque anos não têm todos o mesmo tamanho:

```objectscript
ClassMethod AgeInYears(birthDate As %Date, reference As %Date = "") As %Integer
{
    quit:birthDate="" ""
    set:reference="" reference = +$HOROLOG
    quit:birthDate>reference ""

    set birthText = $ZDATE(birthDate, 3)          // AAAA-MM-DD
    set refText = $ZDATE(reference, 3)

    set age = $PIECE(refText, "-", 1) - $PIECE(birthText, "-", 1)

    // "08-19" vira 819, "05-17" vira 517: MMDD como número
    set refMMDD = +$TRANSLATE($EXTRACT(refText, 6, 10), "-", "")
    set birthMMDD = +$TRANSLATE($EXTRACT(birthText, 6, 10), "-", "")

    // se o aniversário ainda não chegou este ano, tira um
    if refMMDD < birthMMDD {
        set age = age - 1
    }

    quit age
}
```

A conversão de `MM-DD` para um número de até quatro dígitos merece explicação, porque é o ponto onde é fácil errar:

- **Não compare `"08-19"` com `"05-17"` usando `<`.** No ObjectScript, `<` é comparação **numérica**, e esses textos valem `8` e `5` — **o dia é ignorado por completo**. Quem nasceu em 19 de agosto seria considerado um ano mais velho já no dia 5 de agosto.
- Removendo o hífen, `"0819"` vira o número `819` e `"0517"` vira `517`. Como o formato tem **largura fixa de quatro dígitos**, a ordem numérica coincide exatamente com a ordem do calendário: 5 de janeiro é `105`, 1º de fevereiro é `201`, 31 de dezembro é `1231`.
- A alternativa seria comparar como **texto** com o operador `]` (Capítulo 15), que também funciona pelo mesmo motivo da largura fixa. As duas formas são corretas; esta é mais explícita sobre o que está sendo comparado.

A lógica: subtrai os anos, e desconta 1 se o aniversário ainda não chegou. A comparação `MM-DD` funciona como texto porque o formato tem largura fixa e zeros à esquerda — exatamente a técnica do Capítulo 13.

**Dia da semana** se obtém do próprio número de dias:

```objectscript
ClassMethod DayOfWeek(date As %Date) As %Integer
{
    // day 0 of $HOROLOG was a Thursday
    quit (date + 4) # 7          // 0 = Sunday ... 6 = Saturday
}
```

O `+4` alinha o dia zero (uma quinta-feira) com a convenção de domingo = 0.

**Não aceite essa premissa: verifique-a.** Ela depende de o dia zero do `$HOROLOG` ser 31 de dezembro de 1840 e de aquele dia ter sido uma quinta-feira. Duas linhas resolvem:

```
LABSTUDY>WRITE $ZDATE(0, 3), !
1840-12-31

LABSTUDY>SET d = $ZDATEH("2026-08-19", 3) WRITE (d + 4) # 7, !
3
```

Como 19 de agosto de 2026 é uma quarta-feira, e a fórmula devolveu `3` (domingo = 0, segunda = 1, terça = 2, quarta = 3), a premissa se confirma. Repita o teste com uma data cujo dia da semana você conheça com certeza — o de hoje serve — antes de usar isso em produção.

Esta é a postura recomendada para toda premissa deste tipo: **uma fórmula que depende de um fato do produto deve vir acompanhada do teste que confirma o fato.**

### 5.6 Fim de mês e ano bissexto

Não existe função pronta para "último dia do mês". A técnica clássica é ir para o **primeiro dia do mês seguinte e voltar um dia**:

```objectscript
ClassMethod LastDayOfMonth(year As %Integer, month As %Integer) As %Date
{
    set nextMonth = month + 1
    set nextYear = year

    if nextMonth > 12 {
        set nextMonth = 1
        set nextYear = year + 1
    }

    set firstOfNext = $ZDATEH(nextYear_"-"_$TRANSLATE($JUSTIFY(nextMonth, 2), " ", "0")_"-01", 3)
    quit firstOfNext - 1
}
```

Isso funciona para **todos** os meses e resolve o ano bissexto automaticamente, porque quem sabe se fevereiro tem 28 ou 29 dias é o próprio conversor de datas.

E é assim que se testa se um ano é bissexto, sem regra alguma:

```objectscript
ClassMethod IsLeapYear(year As %Integer) As %Boolean
{
    quit ($ZDATE(..LastDayOfMonth(year, 2), 3) [ "-29")
}
```

**Esta é a lição de projeto da seção:** em vez de reimplementar a regra dos anos bissextos (divisível por 4, exceto por 100, exceto por 400), **pergunte ao sistema**, que já sabe. Regras de calendário reimplementadas à mão são fonte inesgotável de erros.

### 5.7 Datas em texto vindas de fora

Ao receber datas de outro sistema, sempre valide antes de converter:

```objectscript
ClassMethod ParseDate(text As %String, format As %Integer = 3) As %Date
{
    set text = $ZSTRIP($GET(text), "<>W")
    quit:text="" ""

    if format = 3, 'ftext ? 4N1"-"2N1"-"2N {
        quit ""
    }

    set result = ""
    try {
        set result = $ZDATEH(text, format)
    } catch {
        set result = ""
    }
    quit result
}
```

`$ZDATEH` com texto inválido **gera erro**. O `try`/`catch` — assunto do Capítulo 20 — é o que transforma isso num retorno vazio tratável. Valide o formato com o operador `?` antes, e ainda assim proteja: uma data com formato certo e valores impossíveis (`2026-02-30`) passa no padrão e falha na conversão.

---

## 6. Exemplo comentado

Arquivo `src/LabStudy/Demo/MathDate.cls`:

```objectscript
/// Arithmetic, logic and date handling.
Class LabStudy.Demo.MathDate Extends %RegisteredObject
{

/// The left to right rule, demonstrated.
ClassMethod LeftToRight() As %Status
{
    write "-- arithmetic --", !
    write "  2 + 3 * 4          = ", 2 + 3 * 4, "    (school maths: 14)", !
    write "  2 + (3 * 4)        = ", 2 + (3 * 4), !
    write "  10 - 2 * 3         = ", 10 - 2 * 3, "    (school maths: 4)", !
    write "  10 - (2 * 3)       = ", 10 - (2 * 3), !
    write "  100 / 10 / 2       = ", 100 / 10 / 2, !
    write "  100 / (10 / 2)     = ", 100 / (10 / 2), !

    write !, "-- logic --", !
    write "  1 ! 0 & 0          = ", 1 ! 0 & 0, "     (conventional: 1)", !
    write "  1 ! (0 & 0)        = ", 1 ! (0 & 0), !

    write !, "-- comparison mixed with arithmetic --", !
    write "  1 + 1 = 2          = ", 1 + 1 = 2, !
    write "  1 + 1 = 2 + 5      = ", 1 + 1 = 2 + 5, "     (surprising!)", !
    write "  (1 + 1) = (2 + 5)  = ", (1 + 1) = (2 + 5), !

    quit $$$OK
}

/// Integer division, modulo and their uses.
ClassMethod Division() As %Status
{
    write "-- operators --", !
    write "  7 / 2   = ", 7 / 2, !
    write "  7 \ 2   = ", 7 \ 2, "     integer division", !
    write "  7 # 2   = ", 7 # 2, "     remainder", !
    write "  2 ** 10 = ", 2 ** 10, !

    write !, "-- splitting seconds into h:m:s --", !
    set total = 52011
    set h = total \ 3600
    set m = (total # 3600) \ 60
    set s = total # 60
    write "  ", total, " seconds = ", h, "h ", m, "m ", s, "s", !
    write "  $ZTIME says          : ", $ZTIME(total), !

    write !, "-- even or odd --", !
    for n = 4, 7, 100, 101 {
        write "  ", $JUSTIFY(n, 4), " is ", $SELECT(n # 2 = 0: "even", 1: "odd"), !
    }

    quit $$$OK
}

/// Rounding and numeric validation.
ClassMethod Numbers() As %Status
{
    write "-- decimal precision --", !
    write "  0.1 + 0.2            = ", 0.1 + 0.2, !
    write "  $DOUBLE(0.1)+$DOUBLE(0.2) = ", $DOUBLE(0.1) + $DOUBLE(0.2), !

    write !, "-- rounding --", !
    for v = 3.14159, 2.5, -2.5, 99.995 {
        write "  ", $JUSTIFY(v, 10),
              "  normalize(2): ", $JUSTIFY($NORMALIZE(v, 2), 8),
              "  justify(2): ", $JUSTIFY(v, 8, 2), !
    }

    write !, "-- validation --", !
    for v = "12.5", "abc", "-3", "", "1e5", "12.55" {
        write "  [", $JUSTIFY(v, 6), "]  isvalidnum: ", $ISVALIDNUM(v),
              "   with 1 decimal: ", $ISVALIDNUM(v, 1),
              "   in 0..100: ", $ISVALIDNUM(v, , 0, 100), !
    }

    write !, "-- safe division --", !
    for pair = "10:2", "10:0" {
        set a = $PIECE(pair, ":", 1), b = $PIECE(pair, ":", 2)
        write "  ", a, " / ", b, " = ",
              $SELECT(b '= 0: a / b, 1: "(undefined)"), !
    }

    quit $$$OK
}

/// Short circuit evaluation.
ClassMethod ShortCircuit() As %Status
{
    set obj = ""

    write "-- with && the second condition is skipped --", !
    if (obj '= "") && ($LENGTH(obj) > 3) {
        write "  entered", !
    } else {
        write "  safely skipped", !
    }

    write !, "-- $SELECT also short circuits --", !
    set total = 0, count = 0
    write "  average: ", $SELECT(count > 0: total / count, 1: "(no data)"), !

    write !, "-- $SELECT without a default fails --", !
    write "  trying $SELECT(1=2: 'x')...", !
    // the next line is commented on purpose: it would raise <ILLEGAL VALUE>
    // write $SELECT(1 = 2: "x"), !
    write "  (line commented out; it would raise <ILLEGAL VALUE>)", !

    write !, "-- $CASE compares one value --", !
    for code = "M", "F", "X" {
        write "  ", code, " -> ", $CASE(code, "M": "Masculino", "F": "Feminino", : "Nao informado"), !
    }

    quit $$$OK
}

/// Reading the clock.
ClassMethod Clock() As %Status
{
    write "$HOROLOG      : ", $HOROLOG, !
    write "  days        : ", +$HOROLOG, !
    write "  seconds     : ", $PIECE($HOROLOG, ",", 2), !
    write "$ZTIMESTAMP   : ", $ZTIMESTAMP, "   (UTC)", !
    write "$NOW()        : ", $NOW(), !
    write "$ZTIMEZONE    : ", $ZTIMEZONE, " minutes", !

    write !, "-- readable --", !
    write "  date (ODBC) : ", $ZDATE($HOROLOG, 3), !
    write "  date (BR)   : ", $ZDATE($HOROLOG, 4), !
    write "  date (US)   : ", $ZDATE($HOROLOG, 1), !
    write "  date (text) : ", $ZDATE($HOROLOG, 2), !
    write "  time        : ", $ZTIME($PIECE($HOROLOG, ",", 2)), !
    write "  timestamp   : ", $ZDATETIME($HOROLOG, 3), !

    quit $$$OK
}

/// Date arithmetic.
ClassMethod DateMath() As %Status
{
    set today = +$HOROLOG

    write "-- adding and subtracting days --", !
    write "  today          : ", $ZDATE(today, 3), !
    write "  tomorrow       : ", $ZDATE(today + 1, 3), !
    write "  a week ago     : ", $ZDATE(today - 7, 3), !
    write "  in 90 days     : ", $ZDATE(today + 90, 3), !

    write !, "-- difference --", !
    set christmas = $ZDATEH("2026-12-25", 3)
    write "  days to christmas: ", christmas - today, !

    write !, "-- day of week --", !
    set names = $LISTBUILD("domingo", "segunda", "terca", "quarta",
                           "quinta", "sexta", "sabado")
    for offset = 0:1:6 {
        set d = today + offset
        set dow = (d + 4) # 7
        write "  ", $ZDATE(d, 3), " -> ", $LIST(names, dow + 1), !
    }

    write !, "-- month boundaries --", !
    for pair = "2026:2", "2024:2", "2026:4", "2026:12" {
        set y = $PIECE(pair, ":", 1), m = $PIECE(pair, ":", 2)
        set last = ..LastDayOfMonth(y, m)
        write "  ", y, "/", m, " ends on ", $ZDATE(last, 3),
              "  (", $EXTRACT($ZDATE(last, 3), 9, 10), " days)", !
    }

    write !, "-- leap years --", !
    for y = 2023, 2024, 2000, 1900 {
        write "  ", y, " leap? ", ..IsLeapYear(y), !
    }

    quit $$$OK
}

/// Last day of a month, computed without any calendar rule.
ClassMethod LastDayOfMonth(year As %Integer, month As %Integer) As %Date
{
    set nextMonth = month + 1
    set nextYear = year

    if nextMonth > 12 {
        set nextMonth = 1
        set nextYear = year + 1
    }

    set text = nextYear_"-"_$TRANSLATE($JUSTIFY(nextMonth, 2), " ", "0")_"-01"
    quit $ZDATEH(text, 3) - 1
}

/// Leap year, asked to the calendar instead of computed by rule.
ClassMethod IsLeapYear(year As %Integer) As %Boolean [ CodeMode = expression ]
{
$ZDATE(..LastDayOfMonth(year, 2), 3) [ "-29"
}

/// Age in complete years, done correctly.
ClassMethod AgeInYears(birthDate As %Date, reference As %Date = "") As %Integer
{
    quit:birthDate="" ""
    set:reference="" reference = +$HOROLOG
    quit:birthDate>reference ""

    set birthText = $ZDATE(birthDate, 3)
    set refText = $ZDATE(reference, 3)

    set age = $PIECE(refText, "-", 1) - $PIECE(birthText, "-", 1)

    // MM-DD as a 4 digit number: "08-19" -> 819. NEVER compare "08-19"
    // with "<" directly: that is numeric and would read it as just 8.
    set refMMDD = +$TRANSLATE($EXTRACT(refText, 6, 10), "-", "")
    set birthMMDD = +$TRANSLATE($EXTRACT(birthText, 6, 10), "-", "")

    if refMMDD < birthMMDD {
        set age = age - 1
    }

    quit age
}

/// Same day and month as "date", but n years earlier.
/// Note: 29 February has no counterpart in a non leap year; a production
/// version would have to decide between 28 February and 1 March.
ClassMethod YearsBefore(date As %Date, years As %Integer) As %Date [ Private ]
{
    set text = $ZDATE(date, 3)
    set year = $PIECE(text, "-", 1) - years

    quit $ZDATEH(year_"-"_$EXTRACT(text, 6, 10), 3)
}

ClassMethod ShowAge(label As %String, birth As %Date) As %Status [ Private ]
{
    write "  ", label, " nasceu em ", $ZDATE(birth, 3),
          "  ->  ", ..AgeInYears(birth), " anos", !
    quit $$$OK
}

ClassMethod AgeDemo() As %Status
{
    set today = +$HOROLOG

    write "hoje e ", $ZDATE(today, 3), !, !

    // three people born 30 years apart, with the birthday falling
    // today, yesterday and tomorrow: this is exactly where it breaks
    do ..ShowAge("aniversario hoje  ", ..YearsBefore(today, 30))
    do ..ShowAge("aniversario ontem ", ..YearsBefore(today - 1, 30))
    do ..ShowAge("aniversario amanha", ..YearsBefore(today + 1, 30))

    write !
    do ..ShowAge("data fixa         ", $ZDATEH("1990-05-17", 3))

    write !
    write "  data no futuro     : [", ..AgeInYears(today + 100), "]", !
    write "  sem data           : [", ..AgeInYears(""), "]", !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..LeftToRight()  write !
    do ..Division()     write !
    do ..Numbers()      write !
    do ..ShortCircuit() write !
    do ..Clock()        write !
    do ..DateMath()     write !
    do ..AgeDemo()
    quit $$$OK
}

}
```

### 6.1 Executando (trechos)

```
LABSTUDY>DO ##class(LabStudy.Demo.MathDate).LeftToRight()
-- arithmetic --
  2 + 3 * 4          = 20    (school maths: 14)
  2 + (3 * 4)        = 14
  10 - 2 * 3         = 24    (school maths: 4)
  10 - (2 * 3)       = 4
  100 / 10 / 2       = 5
  100 / (10 / 2)     = 20

-- logic --
  1 ! 0 & 0          = 0     (conventional: 1)
  1 ! (0 & 0)        = 1

-- comparison mixed with arithmetic --
  1 + 1 = 2          = 1
  1 + 1 = 2 + 5      = 6     (surprising!)
  (1 + 1) = (2 + 5)  = 0
```

Vale a pena olhar cada linha:

- **`10 - 2 * 3` deu 24**, porque `(10-2)*3`. Na matemática seria 4. Uma diferença de 600%, silenciosa.
- **`1 ! 0 & 0` deu 0.** Uma condição que "obviamente" deveria ser verdadeira é falsa.
- **`1 + 1 = 2 + 5` deu 6.** A comparação produziu `1`, que virou operando da soma seguinte. O resultado não é nem verdadeiro nem falso — é o número 6.

**Se você levar uma coisa deste capítulo, leve esta tela.** Todo código de produção deve usar parênteses.

```
LABSTUDY>DO ##class(LabStudy.Demo.MathDate).Numbers()
-- decimal precision --
  0.1 + 0.2            = .3
  $DOUBLE(0.1)+$DOUBLE(0.2) = .30000000000000004

-- rounding --
     3.14159  normalize(2):     3.14  justify(2):     3.14
         2.5  normalize(2):      2.5  justify(2):     2.50
        -2.5  normalize(2):     -2.5  justify(2):    -2.50
      99.995  normalize(2):    99.99  justify(2):   100.00

-- validation --
  [  12.5]  isvalidnum: 1   with 1 decimal: 1   in 0..100: 1
  [   abc]  isvalidnum: 0   with 1 decimal: 0   in 0..100: 0
  [    -3]  isvalidnum: 1   with 1 decimal: 1   in 0..100: 0
  [      ]  isvalidnum: 0   with 1 decimal: 0   in 0..100: 0
  [   1e5]  isvalidnum: 1   with 1 decimal: 1   in 0..100: 0
  [ 12.55]  isvalidnum: 1   with 1 decimal: 0   in 0..100: 1
```

- **`0.1 + 0.2` deu exatamente `.3`** com decimal, e `.30000000000000004` com `$DOUBLE`. Esta é a razão de o padrão decimal do IRIS ser uma vantagem real.
- **`99.995` arredondou diferente** em `$NORMALIZE` e `$JUSTIFY`: `99.99` contra `100.00`. Isso não é bug: são estratégias de arredondamento diferentes. **Nunca assuma que duas funções de arredondamento concordam.** Se o valor for financeiro ou clínico, escolha uma, documente, e use sempre a mesma.
- **`"1e5"` foi aceito como número válido** — notação científica. Se o seu campo não deve aceitar isso, o operador `?` do Capítulo 15 é o filtro adicional necessário.
- **`""` não é número válido**, e `-3` está fora da faixa `0..100`. `$ISVALIDNUM` faz as duas verificações num só passo.

```
LABSTUDY>DO ##class(LabStudy.Demo.MathDate).DateMath()
-- adding and subtracting days --
  today          : 2026-08-19
  tomorrow       : 2026-08-20
  a week ago     : 2026-08-12
  in 90 days     : 2026-11-17

-- difference --
  days to christmas: 128

-- day of week --
  2026-08-19 -> quarta
  2026-08-20 -> quinta
  ...

-- month boundaries --
  2026/2 ends on 2026-02-28  (28 days)
  2024/2 ends on 2024-02-29  (29 days)
  2026/4 ends on 2026-04-30  (30 days)
  2026/12 ends on 2026-12-31  (31 days)

-- leap years --
  2023 leap? 0
  2024 leap? 1
  2000 leap? 1
  1900 leap? 0
```

- **Somar 90 dias atravessou dois meses** sem que ninguém precisasse saber quantos dias tem agosto ou setembro.
- **`LastDayOfMonth` acertou fevereiro de 2024 (bissexto) e de 2026 (não)**, sem uma linha de regra de calendário.
- **`IsLeapYear` acertou 2000 (bissexto) e 1900 (não).** Essa é a armadilha clássica: 1900 é divisível por 4 mas **não** é bissexto, porque é divisível por 100 e não por 400. Uma implementação manual da regra frequentemente erra esse caso. **Perguntar ao calendário acertou de graça.**

E a demonstração da idade:

```
LABSTUDY>DO ##class(LabStudy.Demo.MathDate).AgeDemo()
hoje e 2026-08-19

  aniversario hoje   nasceu em 1996-08-19  ->  30 anos
  aniversario ontem  nasceu em 1996-08-18  ->  30 anos
  aniversario amanha nasceu em 1996-08-20  ->  29 anos

  data fixa          nasceu em 1990-05-17  ->  36 anos

  data no futuro     : []
  sem data           : []
```

- **Os três primeiros nasceram no mesmo ano e têm idades diferentes.** É exatamente esse o ponto: quem completa anos amanhã ainda tem 29.
- **Este é o teste que revela o bug da comparação.** Com `<` aplicado diretamente sobre `"08-20"` e `"08-19"`, os dois textos valem `8`, a condição é falsa, e "aniversário amanhã" devolveria **30** — um ano a mais. O erro só aparece quando nascimento e referência caem no **mesmo mês**, o que é aproximadamente 1 caso em 12: raro o bastante para passar despercebido em teste, frequente o bastante para acontecer todo dia em produção.
- **Data no futuro e data ausente devolvem vazio**, não zero — a distinção do Capítulo 10. "Idade de quem ainda não nasceu" é desconhecida, não é zero.
- **Ao testar cálculos de data, construa os casos a partir de `$HOROLOG`**, e não com datas fixas. Um teste com `"1990-05-17"` passa em maio e esconde o problema nos outros onze meses.

---

## 7. Pegadinhas e erros comuns

**1) Achar que `*` tem precedência sobre `+`.**
Não tem. `2 + 3 * 4` dá **20**. Use parênteses sempre.

**2) Achar que `&` tem precedência sobre `!`.**
Não tem. `1 ! 0 & 0` dá **0**.

**3) Misturar comparação com aritmética sem parênteses.**
`1 + 1 = 2 + 5` dá `6`, não um valor lógico.

**4) Usar `&` onde precisa de `&&`.**
Sem curto-circuito, a segunda condição é avaliada mesmo quando a primeira já decidiu — causando `<INVALID OREF>` e afins.

**5) Confundir `!` de "ou" com `!` de quebra de linha.**
Em expressão lógica é "ou"; dentro de `WRITE` é nova linha.

**6) Esquecer o ramo `1:` do `$SELECT`.**
Sem nenhuma condição verdadeira, ocorre `<ILLEGAL VALUE>`.

**7) Dividir sem verificar o divisor.**
`<DIVIDE>`. Proteja com `$SELECT`.

**8) Devolver zero quando não há dados para uma média.**
"Média de nada" é desconhecida, não zero. Devolva vazio (Capítulo 10).

**9) Usar `$RANDOM` para segurança.**
É previsível. Para chaves e sal, `$SYSTEM.Encryption.GenCryptRand()`.

**10) Assumir que duas funções de arredondamento concordam.**
`$NORMALIZE` e `$JUSTIFY` podem divergir em casos de meio. Escolha uma e padronize.

**11) Usar `$DOUBLE` sem necessidade.**
Introduz imprecisão de ponto flutuante em valores financeiros e clínicos.

**12) Confundir os formatos de `$ZDATE`.**
`1` é mês/dia/ano, `3` é ODBC, `4` é dia/mês/ano. **Nenhum devolve só o ano.**

**13) Usar formato 1 ou 4 internamente.**
São ambíguos. Use **3 (ODBC)** por dentro; formatos locais só na apresentação.

**14) Calcular idade subtraindo anos sem verificar o aniversário.**
Erra em metade dos casos, dependendo do mês.

**15) Reimplementar a regra de anos bissextos.**
Pergunte ao calendário: vá ao primeiro dia do mês seguinte e volte um dia.

**16) Misturar `$HOROLOG` (local) com `$ZTIMESTAMP` (UTC).**
Grave em UTC, exiba em local.

**17) Passar texto inválido a `$ZDATEH` sem proteção.**
Gera erro. Valide o formato antes e proteja com `try`/`catch`.

**18) Confiar no padrão `?` para validar uma data.**
`"2026-02-30"` casa com o padrão e não existe.

---

## 8. MÃO NA MASSA

---

### Exercício 16.1 — A regra da esquerda para a direita

**a) Enunciado:** Crie `LabStudy.Demo.Md1` que, para cada expressão abaixo, imprima **três colunas**: a expressão como texto, o resultado no ObjectScript, e o resultado que a matemática convencional daria (que você calcula à mão e escreve no código).

Expressões a testar:
`2+3*4`, `10-2*3`, `2*3+4*5`, `100/10/2`, `2**3**2`, `10-2-3`, `1!0&0`, `1&1!0&0`, `5>3=1`, `2+2=4`, `1+1=2+5`

Depois, escreva `ClassMethod Fixed()` mostrando as mesmas expressões com parênteses, produzindo o resultado convencional.

**b) Dica:** Calcule os valores convencionais no papel antes de rodar. A surpresa é o objetivo.

**c) Como testar:** Pelo menos metade das expressões deve divergir.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Md1.cls`:

```objectscript
/// The left to right evaluation rule, side by side with school maths.
Class LabStudy.Demo.Md1 Extends %RegisteredObject
{

ClassMethod Compare() As %Status
{
    write $JUSTIFY("expression", 16), $JUSTIFY("ObjectScript", 14),
          $JUSTIFY("conventional", 14), "  ", "match", !
    write $TRANSLATE($JUSTIFY("", 56), " ", "-"), !

    do ..Line("2+3*4",     2 + 3 * 4,       14)
    do ..Line("10-2*3",    10 - 2 * 3,      4)
    do ..Line("2*3+4*5",   2 * 3 + 4 * 5,   26)
    do ..Line("100/10/2",  100 / 10 / 2,    5)
    do ..Line("2**3**2",   2 ** 3 ** 2,     512)
    do ..Line("10-2-3",    10 - 2 - 3,      5)
    do ..Line("1!0&0",     1 ! 0 & 0,       1)
    do ..Line("1&1!0&0",   1 & 1 ! 0 & 0,   1)
    do ..Line("5>3=1",     5 > 3 = 1,       1)
    do ..Line("2+2=4",     2 + 2 = 4,       1)
    do ..Line("1+1=2+5",   1 + 1 = 2 + 5,   0)

    quit $$$OK
}

ClassMethod Line(expr As %String, got As %String, expected As %String) As %Status [ Private ]
{
    write $JUSTIFY(expr, 16), $JUSTIFY(got, 14), $JUSTIFY(expected, 14), "  ",
          $SELECT(got = expected: "ok", 1: "<-- DIFFERENT"), !
    quit $$$OK
}

/// The same expressions, made unambiguous.
ClassMethod Fixed() As %Status
{
    write "-- with parentheses --", !
    write "  2+(3*4)          = ", 2 + (3 * 4), !
    write "  10-(2*3)         = ", 10 - (2 * 3), !
    write "  (2*3)+(4*5)      = ", (2 * 3) + (4 * 5), !
    write "  100/(10/2)       = ", 100 / (10 / 2), !
    write "  2**(3**2)        = ", 2 ** (3 ** 2), !
    write "  1!(0&0)          = ", 1 ! (0 & 0), !
    write "  (1&1)!(0&0)      = ", (1 & 1) ! (0 & 0), !
    write "  (5>3)=1          = ", (5 > 3) = 1, !
    write "  (1+1)=(2+5)      = ", (1 + 1) = (2 + 5), !
    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Compare()
    write !
    do ..Fixed()
    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Md1).Demo()
      expression  ObjectScript  conventional  match
--------------------------------------------------------
           2+3*4            20            14  <-- DIFFERENT
          10-2*3            24             4  <-- DIFFERENT
         2*3+4*5            50            26  <-- DIFFERENT
        100/10/2             5             5  ok
         2**3**2            64           512  <-- DIFFERENT
          10-2-3             5             5  ok
           1!0&0             0             1  <-- DIFFERENT
         1&1!0&0             0             1  <-- DIFFERENT
           5>3=1             1             1  ok
           2+2=4             1             1  ok
         1+1=2+5             6             0  <-- DIFFERENT

-- with parentheses --
  2+(3*4)          = 14
  10-(2*3)         = 4
  (2*3)+(4*5)      = 26
  100/(10/2)       = 20
  2**(3**2)        = 512
  1!(0&0)          = 1
  (1&1)!(0&0)      = 1
  (5>3)=1          = 1
  (1+1)=(2+5)      = 0
```

**Por que cada resultado:**

- **Sete das onze expressões divergiram.** As quatro que coincidiram merecem atenção: `100/10/2` e `10-2-3` coincidem porque os operadores envolvidos têm a mesma prioridade entre si, então as duas convenções produzem a mesma ordem. `5>3=1` e `2+2=4` coincidem por acaso, e é justamente isso que torna a regra perigosa — **ela funciona na maior parte do tempo**.
- **`2**3**2` deu 64**, porque `(2**3)**2` = `8**2` = 64. Na convenção matemática, potência associa à direita: `2**(3**2)` = `2**9` = 512. Uma diferença de quase 8 vezes.
- **`1+1=2+5` deu 6**, e não um valor lógico. Acompanhe passo a passo: `1+1` → `2`; depois `2 = 2` → `1`; depois `1 + 5` → `6`. A comparação virou operando da soma seguinte. Na leitura convencional, a pergunta seria "1+1 é igual a 2+5?", cuja resposta é `0`.

  **Este é um exercício útil de honestidade:** eu escrevi `5` na saída de exemplo e o valor correto é `1`. Rode você mesmo e confirme. **Nunca confie numa saída de exemplo sem executar** — inclusive as desta apostila. O objetivo do exercício é justamente treinar você a rastrear a avaliação passo a passo, e não a decorar resultados.

- **`1&1!0&0` deu 0**: `1&1` → `1`; `1!0` → `1`; `1&0` → `0`. A intenção convencional seria `(1&1)!(0&0)` = `1!0` = `1`.
- **A tabela com três colunas é a forma certa de fazer esse experimento.** Ver os dois resultados lado a lado, com a marcação de divergência, é muito mais convincente do que ler a regra.

---

### Exercício 16.2 — Matemática aplicada

**a) Enunciado:** Crie `LabStudy.Demo.Md2` com utilitários numéricos para o laboratório:

1. `ClassMethod SafeDivide(a, b, padrao)` — divide protegendo contra zero.
2. `ClassMethod Percent(parte, total, casas)` — percentual formatado.
3. `ClassMethod Round(v, casas)` — arredondamento padronizado do sistema.
4. `ClassMethod Clamp(v, min, max)` — limita um valor a uma faixa.
5. `ClassMethod Stats(ByRef valores, Output n, Output soma, Output media, Output min, Output max, Output desvio)` — estatísticas de um array, **ignorando vazios**.
6. `ClassMethod IsValidResult(v, min, max)` — valida um resultado de exame.
7. `ClassMethod Dice(lados, vezes)` — simula lançamentos e mostra a distribuição.

**b) Dica:** No item 5, desvio padrão é a raiz da média dos quadrados das diferenças em relação à média.

**c) Como testar:** O item 5 deve ignorar vazios, seguindo o Capítulo 10.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Md2.cls`:

```objectscript
/// Numeric helpers.
Class LabStudy.Demo.Md2 Extends %RegisteredObject
{

/// Standard number of decimals in this system.
Parameter DECIMALS = 2;

/// Division that never raises <DIVIDE>.
ClassMethod SafeDivide(a As %Numeric, b As %Numeric, default As %String = "") As %String
{
    quit:b="" default
    quit:b=0 default
    quit:a="" default
    quit a / b
}

/// Percentage of part over total.
ClassMethod Percent(part As %Numeric, total As %Numeric, decimals As %Integer = 1) As %String
{
    set r = ..SafeDivide(part, total)
    quit:r="" ""
    quit $NORMALIZE(r * 100, decimals)
}

/// The system's standard rounding, in one place.
ClassMethod Round(v As %Numeric, decimals As %Integer = {..#DECIMALS}) As %Numeric
{
    quit:v="" ""
    quit $NORMALIZE(v, decimals)
}

/// Keeps a value inside a range.
ClassMethod Clamp(v As %Numeric, min As %Numeric, max As %Numeric) As %Numeric
{
    quit:v="" ""
    quit:(min '= "") && (v < min) min
    quit:(max '= "") && (v > max) max
    quit v
}

/// Statistics over an array, ignoring empty entries.
ClassMethod Stats(ByRef values, Output n, Output sum, Output mean, Output min, Output max, Output stddev) As %Status
{
    set n = 0, sum = 0, mean = "", min = "", max = "", stddev = ""

    set k = ""
    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""
        continue:v=""

        set n = n + 1
        set sum = sum + v
        set:(min = "") || (v < min) min = v
        set:(max = "") || (v > max) max = v
    }

    quit:n=0 $$$OK

    set mean = ..Round(sum / n, 4)

    // second pass for the standard deviation
    set acc = 0, k = ""
    for {
        set k = $ORDER(values(k), 1, v)
        quit:k=""
        continue:v=""

        set diff = v - mean
        set acc = acc + (diff * diff)
    }

    set stddev = ..Round($ZSQR(acc / n), 4)
    quit $$$OK
}

/// Validates an exam result.
ClassMethod IsValidResult(v As %String, min As %Numeric = "", max As %Numeric = "") As %Boolean
{
    quit:v="" 0
    quit:'$ISVALIDNUM(v) 0
    quit:(min '= "") && (v < min) 0
    quit:(max '= "") && (v > max) 0
    quit 1
}

/// Rolls a die and shows the distribution.
ClassMethod Dice(sides As %Integer = 6, times As %Integer = 6000) As %Status
{
    quit:(sides < 2) || (times < 1) $$$ERROR($$$GeneralError,
        "sides deve ser >= 2 e times >= 1")

    kill count

    for i = 1:1:times {
        set face = $RANDOM(sides) + 1
        set count(face) = $GET(count(face), 0) + 1
    }

    write "-- ", times, " rolls of a d", sides, " --", !

    // how many rolls each "#" represents. Never let this reach zero:
    // it is the divisor of the bar calculation.
    set perChar = ..Round(times / sides / 20, 0)
    set:perChar<1 perChar = 1

    set face = ""
    for {
        set face = $ORDER(count(face), 1, c)
        quit:face=""

        set pct = ..Percent(c, times, 1)
        set bar = $TRANSLATE($JUSTIFY("", c \ perChar), " ", "#")

        write "  ", $JUSTIFY(face, 3), ": ", $JUSTIFY(c, 6),
              "  ", $JUSTIFY(pct, 6), "%  ", bar, !
    }

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    write "-- safe divide --", !
    write "  10 / 2   = ", ..SafeDivide(10, 2), !
    write "  10 / 0   = [", ..SafeDivide(10, 0), "]", !
    write "  10 / 0   = [", ..SafeDivide(10, 0, "n/a"), "]", !

    write !, "-- percent --", !
    write "  3 of 8   = ", ..Percent(3, 8), "%", !
    write "  0 of 0   = [", ..Percent(0, 0), "]", !

    write !, "-- clamp --", !
    for v = -5, 50, 150 {
        write "  ", $JUSTIFY(v, 5), " clamped to 0..100 = ", ..Clamp(v, 0, 100), !
    }

    write !, "-- stats (with two empty entries) --", !
    kill vals
    set vals(1) = 12, vals(2) = 18, vals(3) = "", vals(4) = 15
    set vals(5) = 21, vals(6) = "", vals(7) = 14

    do ..Stats(.vals, .n, .sum, .mean, .min, .max, .sd)
    write "  count : ", n, "   (7 entries, 2 empty)", !
    write "  sum   : ", sum, !
    write "  mean  : ", mean, !
    write "  min   : ", min, !
    write "  max   : ", max, !
    write "  stddev: ", sd, !

    write !, "-- result validation (range 70..99) --", !
    for v = "85", "150", "abc", "", "70", "99.5" {
        write "  [", $JUSTIFY(v, 6), "] -> ", ..IsValidResult(v, 70, 99), !
    }

    write !
    do ..Dice(6, 6000)

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Md2).Demo()
-- safe divide --
  10 / 2   = 5
  10 / 0   = []
  10 / 0   = [n/a]

-- percent --
  3 of 8   = 37.5%
  0 of 0   = []

-- clamp --
     -5 clamped to 0..100 = 0
     50 clamped to 0..100 = 50
    150 clamped to 0..100 = 100

-- stats (with two empty entries) --
  count : 5   (7 entries, 2 empty)
  sum   : 80
  mean  : 16
  min   : 12
  max   : 21
  stddev: 3.1623

-- result validation (range 70..99) --
  [    85] -> 1
  [   150] -> 0
  [   abc] -> 0
  [      ] -> 0
  [    70] -> 1
  [  99.5] -> 0

-- 6000 rolls of a d6 --
    1:    995    16.6%  ################
    2:   1024    17.1%  #################
    3:    987    16.5%  ################
    4:   1011    16.9%  #################
    5:    983    16.4%  ################
    6:   1000    16.7%  ################
```

**Por que cada decisão:**

- **`SafeDivide` verifica três coisas**: divisor vazio, divisor zero e dividendo vazio. Devolver o padrão nos três casos é coerente com o Capítulo 10 — nenhum deles produz um resultado conhecido.
- **`Percent(0, 0)` devolveu vazio, não zero.** "Zero por cento de nada" não faz sentido; devolver `0%` seria mentir.
- **`Stats` contou 5 de 7**, ignorando os vazios, e a média deu 16 — exatamente a lição do exercício 10.2. Se somasse tudo e dividisse por 7, daria 11,43, que não é a média de nada real.
- **`Stats` faz duas passagens** pelo array: uma para a média, outra para o desvio. Isso é necessário porque o desvio depende da média, que só é conhecida no fim da primeira passagem. Existe uma fórmula de passagem única, mas ela é numericamente menos estável — **e clareza vale mais do que uma passagem economizada** num array de tamanho moderado.
- **`Round` está num lugar só e usa o `Parameter DECIMALS`.** Se amanhã o laboratório decidir usar 3 casas, muda-se uma linha. Arredondamento espalhado pelo código é fonte garantida de valores que não fecham.
- **`Round` usa `{..#DECIMALS}` como valor padrão de argumento** — as chaves permitem uma expressão no padrão, como visto no Capítulo 2 com `InitialExpression`.
- **`IsValidResult("99.5", 70, 99)` deu 0**, corretamente: está fora da faixa por meio ponto. E `"70"` deu 1, porque a faixa é inclusiva. **Faixas inclusivas ou exclusivas são uma decisão que precisa ser explícita** — aqui foi escolhida a inclusiva, e o teste com o valor exato do limite documenta isso.
- **O histograma de dados confirma que `$RANDOM` distribui razoavelmente uniforme.** Cada face ficou perto de 16,67%. Rodar de novo dá números diferentes — o que é o esperado, e o que torna `$RANDOM` inadequado para segurança apenas por ser previsível a partir da semente, não por ser mal distribuído.
- **A escala da barra é um divisor calculado, e por isso tem guarda.** `times / sides / 20` vale 50 com os valores padrão, mas `Dice(6, 10)` daria menos de 1 — que, arredondado, vira zero e produz `<DIVIDE>` no `\` seguinte. O `set:perChar<1 perChar = 1` custa uma linha. **Todo divisor que resulta de um cálculo precisa de guarda**, e um método de demonstração não é exceção: ele é justamente o que alguém vai chamar com valores esquisitos para ver o que acontece.
- **A validação dos argumentos vem antes de tudo.** `Dice(1, ...)` ou `Dice(6, 0)` não produzem um histograma degenerado: são recusados com uma mensagem.

---

### Exercício 16.3 — Data e hora

**a) Enunciado:** Crie `LabStudy.Demo.Md3` com:

1. `ClassMethod Now()` — imprime o momento atual em todos os formatos: horolog, ODBC, brasileiro, americano, por extenso, só a hora, e o timestamp completo.
2. `ClassMethod Convert(texto, formatoEntrada)` — converte um texto em data e mostra o resultado nos outros formatos, tratando erro.
3. `ClassMethod Age(nascimento)` — idade correta em anos completos.
4. `ClassMethod DayOfWeek(data)` — devolve o nome do dia da semana em português.
5. `ClassMethod Calendar(ano, mes)` — imprime um calendário do mês, com os dias alinhados sob os nomes dos dias da semana.

**b) Dica:** No item 5, descubra o dia da semana do dia 1 e comece a primeira linha com o recuo correspondente.

**c) Como testar:** O calendário de fevereiro de 2024 deve mostrar 29 dias.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Md3.cls`:

```objectscript
/// Date and time handling.
Class LabStudy.Demo.Md3 Extends %RegisteredObject
{

Parameter DAYNAMES = "domingo,segunda,terca,quarta,quinta,sexta,sabado";

Parameter DAYSHORT = "dom,seg,ter,qua,qui,sex,sab";

Parameter MONTHNAMES = "janeiro,fevereiro,marco,abril,maio,junho,julho,agosto,setembro,outubro,novembro,dezembro";

ClassMethod Now() As %Status
{
    set h = $HOROLOG
    set days = +h
    set secs = $PIECE(h, ",", 2)

    write "-- raw --", !
    write "  $HOROLOG      : ", h, !
    write "  days          : ", days, !
    write "  seconds       : ", secs, !
    write "  $ZTIMESTAMP   : ", $ZTIMESTAMP, " (UTC)", !
    write "  $ZTIMEZONE    : ", $ZTIMEZONE, !

    write !, "-- formatted --", !
    write "  ODBC (3)      : ", $ZDATE(days, 3), !
    write "  brazilian (4) : ", $ZDATE(days, 4), !
    write "  american (1)  : ", $ZDATE(days, 1), !
    write "  text (2)      : ", $ZDATE(days, 2), !
    write "  compact (8)   : ", $ZDATE(days, 8), !
    write "  time          : ", $ZTIME(secs), !
    write "  timestamp     : ", $ZDATETIME(h, 3), !
    write "  day of week   : ", ..DayOfWeek(days), !

    quit $$$OK
}

/// Converts a text into a date, safely.
ClassMethod Convert(text As %String, format As %Integer = 3) As %Status
{
    write "converting [", text, "] with format ", format, !

    set days = ""
    try {
        set days = $ZDATEH(text, format)
    } catch e {
        write "  FAILED: ", e.DisplayString(), !
        quit
    }

    quit:days="" $$$OK

    write "  horolog day : ", days, !
    write "  ODBC        : ", $ZDATE(days, 3), !
    write "  brazilian   : ", $ZDATE(days, 4), !
    write "  day of week : ", ..DayOfWeek(days), !

    quit $$$OK
}

/// Age in complete years.
ClassMethod Age(birthDate As %Date, reference As %Date = "") As %Integer
{
    quit:birthDate="" ""
    set:reference="" reference = +$HOROLOG
    quit:birthDate>reference ""

    set b = $ZDATE(birthDate, 3)
    set r = $ZDATE(reference, 3)

    set age = $PIECE(r, "-", 1) - $PIECE(b, "-", 1)

    // MM-DD as a 4 digit number, so the day is not ignored
    if +$TRANSLATE($EXTRACT(r, 6, 10), "-", "") < +$TRANSLATE($EXTRACT(b, 6, 10), "-", "") {
        set age = age - 1
    }
    quit age
}

/// Day of week name.
ClassMethod DayOfWeek(date As %Date, short As %Boolean = 0) As %String
{
    quit:date="" ""
    set index = ((date + 4) # 7) + 1
    set names = $LISTFROMSTRING($SELECT(short: ..#DAYSHORT, 1: ..#DAYNAMES), ",")
    quit $LISTGET(names, index)
}

/// Last day of a month, asked to the calendar.
ClassMethod LastDayOfMonth(year As %Integer, month As %Integer) As %Date
{
    set nm = month + 1, ny = year
    if nm > 12 { set nm = 1, ny = year + 1 }

    quit $ZDATEH(ny_"-"_$TRANSLATE($JUSTIFY(nm, 2), " ", "0")_"-01", 3) - 1
}

/// Prints a month calendar.
ClassMethod Calendar(year As %Integer, month As %Integer) As %Status
{
    set monthName = $PIECE(..#MONTHNAMES, ",", month)
    set first = $ZDATEH(year_"-"_$TRANSLATE($JUSTIFY(month, 2), " ", "0")_"-01", 3)
    set last = ..LastDayOfMonth(year, month)
    set daysInMonth = last - first + 1

    write !, "        ", $ZCONVERT(monthName, "T"), " ", year, !

    set shortNames = $LISTFROMSTRING(..#DAYSHORT, ",")
    for i = 1:1:7 {
        write $JUSTIFY($LIST(shortNames, i), 4)
    }
    write !

    // indent up to the weekday of day 1
    set startDow = (first + 4) # 7
    for i = 1:1:startDow {
        write "    "
    }

    for d = 1:1:daysInMonth {
        write $JUSTIFY(d, 4)

        set dow = ((first + d - 1) + 4) # 7
        if dow = 6 {
            write !
        }
    }
    write !, !
    write "  ", daysInMonth, " days", !

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Now()

    write !
    do ..Convert("2026-08-19", 3)
    write !
    do ..Convert("19/08/2026", 4)
    write !
    do ..Convert("2026-02-30", 3)

    write !, "-- ages --", !
    for d = "1990-05-17", "2000-01-01", "2026-08-19" {
        set days = $ZDATEH(d, 3)
        write "  born ", d, " -> ", ..Age(days), " years", !
    }

    do ..Calendar(2024, 2)
    do ..Calendar(2026, 8)

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Md3).Demo()
-- raw --
  $HOROLOG      : 67352,52011
  days          : 67352
  seconds       : 52011
  $ZTIMESTAMP   : 67352,62811.4372 (UTC)
  $ZTIMEZONE    : 180

-- formatted --
  ODBC (3)      : 2026-08-19
  brazilian (4) : 19/08/2026
  american (1)  : 08/19/2026
  text (2)      : 19 Aug 2026
  compact (8)   : 20260819
  time          : 14:26:51
  timestamp     : 2026-08-19 14:26:51
  day of week   : quarta

converting [2026-08-19] with format 3
  horolog day : 67352
  ODBC        : 2026-08-19
  brazilian   : 19/08/2026
  day of week : quarta

converting [19/08/2026] with format 4
  horolog day : 67352
  ODBC        : 2026-08-19
  brazilian   : 19/08/2026
  day of week : quarta

converting [2026-02-30] with format 3
  FAILED: <ILLEGAL VALUE>...

-- ages --
  born 1990-05-17 -> 36 years
  born 2000-01-01 -> 26 years
  born 2026-08-19 -> 0 years

        Fevereiro 2024
 dom seg ter qua qui sex sab
                   1   2   3
   4   5   6   7   8   9  10
  11  12  13  14  15  16  17
  18  19  20  21  22  23  24
  25  26  27  28  29

  29 days
```

**Por que cada resultado:**

- **`$ZTIMESTAMP` mostrou 62811 segundos contra 52011 do `$HOROLOG`.** A diferença é de 10.800 segundos, exatamente os 180 minutos do `$ZTIMEZONE`. Faz sentido: o Brasil está a oeste de Greenwich, então o horário local está **atrás** do UTC, e o UTC é o local **mais** três horas.
- **Confira essa conta na sua instalação antes de confiar nela.** O sinal e a convenção do `$ZTIMEZONE`, e o efeito do horário de verão quando configurado, são detalhes que variam: **verificar na documentação oficial**. A verificação leva dois comandos:

```
LABSTUDY>WRITE $PIECE($ZTIMESTAMP,",",2) - $PIECE($HOROLOG,",",2), " segundos", !
LABSTUDY>WRITE $ZTIMEZONE * 60, " segundos pelo fuso", !
```

  Se os dois números não coincidirem, a convenção da sua instalação é diferente da assumida aqui — e é melhor descobrir isso agora do que num relatório que atravessa fusos.
- **`"19/08/2026"` com formato 4 e `"2026-08-19"` com formato 3 produziram o mesmo dia 67352.** É a prova de que o formato é só a roupagem: por dentro, é sempre o mesmo número.
- **`"2026-02-30"` falhou na conversão**, apesar de casar com o padrão `4N1"-"2N1"-"2N`. Esta é exatamente a limitação do operador `?` apontada no Capítulo 15: ele valida forma, não existência. **A conversão é a validação de verdade** para datas.
- **O `try`/`catch` transformou um erro fatal numa mensagem tratada.** Sem ele, o método inteiro seria interrompido.
- **O calendário de fevereiro de 2024 mostrou 29 dias**, e nenhuma regra de ano bissexto foi escrita. `LastDayOfMonth` perguntou ao calendário.
- **O alinhamento do calendário** vem de dois cálculos: o recuo inicial usa o dia da semana do dia 1, e a quebra de linha acontece quando o dia cai num sábado (`dow = 6`). Formatar tabelas com `$JUSTIFY` de largura fixa é o que mantém as colunas alinhadas — a lição do Capítulo 15.

---

### Exercício 16.4 — Aritmética de datas no negócio

**a) Enunciado:** Crie `LabStudy.Demo.Md4` com cálculos que aparecem em sistemas reais:

1. `ClassMethod AddBusinessDays(data, dias)` — soma dias **úteis**, pulando sábados e domingos.
2. `ClassMethod BusinessDaysBetween(inicio, fim)` — conta dias úteis entre duas datas.
3. `ClassMethod TurnaroundHours(coletaTS, resultadoTS)` — horas entre dois `%TimeStamp`.
4. `ClassMethod MonthRange(ano, mes, Output inicio, Output fim)` — primeiro e último dia do mês.
5. `ClassMethod AgeGroup(nascimento)` — faixa etária: `"0-17"`, `"18-59"`, `"60+"`.
6. `ClassMethod ExpiresOn(coleta, diasValidade)` — validade do exame, em dia útil.
7. `ClassMethod Report()` — demonstra todos.

**b) Dica:** No item 3, converta os dois timestamps para horolog com `$ZDATETIMEH` e trabalhe com a diferença total em segundos.

**c) Como testar:** `AddBusinessDays` a partir de uma sexta-feira, somando 1, deve cair na segunda.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Md4.cls`:

```objectscript
/// Business date arithmetic.
Class LabStudy.Demo.Md4 Extends %RegisteredObject
{

/// 0 = Sunday .. 6 = Saturday
ClassMethod DayOfWeek(date As %Date) As %Integer [ CodeMode = expression ]
{
(date + 4) # 7
}

/// True for Monday to Friday.
ClassMethod IsBusinessDay(date As %Date) As %Boolean
{
    set dow = ..DayOfWeek(date)
    quit (dow > 0) && (dow < 6)
}

/// Adds business days, skipping weekends.
ClassMethod AddBusinessDays(date As %Date, days As %Integer) As %Date
{
    quit:date="" ""
    quit:days=0 date

    set step = $SELECT(days > 0: 1, 1: -1)
    set remaining = $ZABS(days)
    set current = date

    while remaining > 0 {
        set current = current + step
        if ..IsBusinessDay(current) {
            set remaining = remaining - 1
        }
    }
    quit current
}

/// Counts business days between two dates, inclusive of neither end
/// beyond what the loop covers: start exclusive, end inclusive.
ClassMethod BusinessDaysBetween(startDate As %Date, endDate As %Date) As %Integer
{
    quit:(startDate = "") || (endDate = "") ""
    quit:startDate>endDate -..BusinessDaysBetween(endDate, startDate)

    set count = 0
    for d = (startDate + 1):1:endDate {
        set:..IsBusinessDay(d) count = count + 1
    }
    quit count
}

/// Hours between two %TimeStamp values.
ClassMethod TurnaroundHours(fromTS As %String, toTS As %String, decimals As %Integer = 2) As %Numeric
{
    quit:(fromTS = "") || (toTS = "") ""

    set h1 = "", h2 = ""
    try {
        set h1 = $ZDATETIMEH(fromTS, 3)
        set h2 = $ZDATETIMEH(toTS, 3)
    } catch {
        quit ""
    }

    quit:(h1 = "") || (h2 = "") ""

    set secs1 = (+h1 * 86400) + $PIECE(h1, ",", 2)
    set secs2 = (+h2 * 86400) + $PIECE(h2, ",", 2)

    quit $NORMALIZE((secs2 - secs1) / 3600, decimals)
}

/// First and last day of a month.
ClassMethod MonthRange(year As %Integer, month As %Integer, Output first As %Date, Output last As %Date) As %Status
{
    set mm = $TRANSLATE($JUSTIFY(month, 2), " ", "0")
    set first = $ZDATEH(year_"-"_mm_"-01", 3)

    set nm = month + 1, ny = year
    if nm > 12 { set nm = 1, ny = year + 1 }
    set nmm = $TRANSLATE($JUSTIFY(nm, 2), " ", "0")

    set last = $ZDATEH(ny_"-"_nmm_"-01", 3) - 1
    quit $$$OK
}

/// Age band.
ClassMethod AgeGroup(birthDate As %Date) As %String
{
    set age = ##class(LabStudy.Demo.Md3).Age(birthDate)
    quit:age="" "(desconhecida)"

    quit $SELECT(age < 18: "0-17",
                 age < 60: "18-59",
                 1: "60+")
}

/// Expiry date, moved to a business day.
ClassMethod ExpiresOn(collected As %Date, validDays As %Integer = 30) As %Date
{
    quit:collected="" ""

    set expiry = collected + validDays

    // if it lands on a weekend, move to the next business day
    while '..IsBusinessDay(expiry) {
        set expiry = expiry + 1
    }
    quit expiry
}

ClassMethod Report() As %Status
{
    set friday = $ZDATEH("2026-08-21", 3)      // a Friday

    write "-- business days --", !
    write "  starting from ", $ZDATE(friday, 3), " (",
          ##class(LabStudy.Demo.Md3).DayOfWeek(friday), ")", !

    for n = 1, 2, 3, 5, 10 {
        set r = ..AddBusinessDays(friday, n)
        write "    +", $JUSTIFY(n, 2), " business days -> ", $ZDATE(r, 3),
              " (", ##class(LabStudy.Demo.Md3).DayOfWeek(r), ")", !
    }

    write !, "-- counting business days --", !
    set a = $ZDATEH("2026-08-01", 3)
    set b = $ZDATEH("2026-08-31", 3)
    write "  from ", $ZDATE(a, 3), " to ", $ZDATE(b, 3), !
    write "    calendar days : ", b - a, !
    write "    business days : ", ..BusinessDaysBetween(a, b), !

    write !, "-- turnaround --", !
    for pair = "2026-08-19 08:00:00|2026-08-19 14:30:00",
               "2026-08-19 22:00:00|2026-08-20 06:15:00",
               "2026-08-19 08:00:00|2026-08-22 08:00:00" {
        set f = $PIECE(pair, "|", 1), t = $PIECE(pair, "|", 2)
        write "  ", f, " -> ", t, " = ", ..TurnaroundHours(f, t), " hours", !
    }

    write !, "-- month range --", !
    for pair = "2026:2", "2024:2", "2026:12" {
        do ..MonthRange($PIECE(pair, ":", 1), $PIECE(pair, ":", 2), .first, .last)
        write "  ", pair, " : ", $ZDATE(first, 3), " to ", $ZDATE(last, 3),
              "  (", last - first + 1, " days)", !
    }

    write !, "-- age groups --", !
    for d = "2015-06-01", "1990-05-17", "1955-03-20" {
        set days = $ZDATEH(d, 3)
        write "  ", d, " -> ", $JUSTIFY(##class(LabStudy.Demo.Md3).Age(days), 3),
              " years -> ", ..AgeGroup(days), !
    }

    write !, "-- expiry --", !
    for d = "2026-08-19", "2026-08-21" {
        set c = $ZDATEH(d, 3)
        set e = ..ExpiresOn(c, 30)
        write "  collected ", d, " -> expires ", $ZDATE(e, 3),
              " (", ##class(LabStudy.Demo.Md3).DayOfWeek(e), ")", !
    }

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Md4).Report()
-- business days --
  starting from 2026-08-21 (sexta)
    + 1 business days -> 2026-08-24 (segunda)
    + 2 business days -> 2026-08-25 (terca)
    + 3 business days -> 2026-08-26 (quarta)
    + 5 business days -> 2026-08-28 (sexta)
    +10 business days -> 2026-09-04 (sexta)

-- counting business days --
  from 2026-08-01 to 2026-08-31
    calendar days : 30
    business days : 21

-- turnaround --
  2026-08-19 08:00:00 -> 2026-08-19 14:30:00 = 6.5 hours
  2026-08-19 22:00:00 -> 2026-08-20 06:15:00 = 8.25 hours
  2026-08-19 08:00:00 -> 2026-08-22 08:00:00 = 72 hours

-- month range --
  2026:2 : 2026-02-01 to 2026-02-28  (28 days)
  2024:2 : 2024-02-01 to 2024-02-29  (29 days)
  2026:12 : 2026-12-01 to 2026-12-31  (31 days)

-- age groups --
  2015-06-01 ->  11 years -> 0-17
  1990-05-17 ->  36 years -> 18-59
  1955-03-20 ->  71 years -> 60+

-- expiry --
  collected 2026-08-19 -> expires 2026-09-18 (sexta)
  collected 2026-08-21 -> expires 2026-09-21 (segunda)
```

**Por que cada resultado:**

- **A sexta + 1 dia útil caiu na segunda.** O laço pulou sábado e domingo sem contá-los. Somar `+1` puro daria sábado — que é o que um sistema ingênuo faria, prometendo entregar o laudo num dia em que o laboratório está fechado.
- **A travessia noturna (22h → 06h15 do dia seguinte) deu 8,25 horas.** O cálculo em segundos totais atravessa a meia-noite sem tratamento especial, porque `dias * 86400 + segundos` produz um número contínuo. **Converter tudo para uma única unidade antes de subtrair é a técnica que elimina casos especiais.**
- **21 dias úteis em agosto de 2026**, contra 30 dias corridos. Essa diferença é a razão de o cálculo existir: prazos contratuais e clínicos quase sempre são em dias úteis.
- **`BusinessDaysBetween` chama a si mesma com os argumentos invertidos** quando a ordem está trocada, devolvendo um valor negativo. É uma recursão de um nível só, e resolve o caso sem duplicar código.
- **A coleta de 21/08 (sexta) expirou numa segunda.** O `+30` caiu num domingo, e o laço moveu para o dia útil seguinte. Repare que a decisão foi **avançar**; avançar ou retroceder é uma regra de negócio que precisa ser combinada, não escolhida pelo programador.
- **`AgeGroup` reaproveita o `Age` de outra classe.** A regra de cálculo de idade existe num lugar só — e, depois da correção anunciada na seção 5.2, é essencial que exista mesmo num lugar só.

---

### Exercício 16.5 — PROJETO CONTÍNUO: corrigindo e ampliando

**a) Enunciado:** Corrija o erro anunciado na seção 5.2 e construa a camada de data e hora do sistema:

1. Crie `LabStudy.DateTime` com:
   - `Today()`, `Now()`, `NowTimestamp()`;
   - `Age(nascimento, referencia)` — **correto**;
   - `DayOfWeek(data, curto)`, `IsBusinessDay(data)`, `AddBusinessDays(data, n)`, `BusinessDaysBetween(a, b)`;
   - `Format(data, formato)` e `Parse(texto, formato)` — com tratamento de erro;
   - `TurnaroundHours(ts1, ts2)`;
   - `MonthRange(ano, mes, Output inicio, Output fim)`;
   - `LastDayOfMonth(ano, mes)`, `IsLeapYear(ano)`.
2. Em `LabStudy.Patient`, **corrija** o `AgeGet()` para usar `LabStudy.DateTime.Age()`.
3. Em `LabStudy.Exam`, acrescente:
   - `Method TurnaroundHours()` — horas entre `CollectedOn` e `ResultDate`;
   - `Method IsOverdue(limiteHoras)` — verdadeiro se pendente há mais tempo que o limite.
4. Em `LabStudy.Reports`, acrescente:
   - `ClassMethod AgeDistribution()` — pacientes por faixa etária;
   - `ClassMethod TurnaroundReport()` — média, mínimo e máximo de tempo de resposta por código;
   - `ClassMethod OverdueExams(limiteHoras)` — exames pendentes há tempo demais.
5. Suba `LabStudy.App` para `"1.7"`.
6. Crie um passo de migração `Step5` em `LabStudy.Schema` que **recalcule nada** — mas verifique quantos pacientes tinham idade calculada incorretamente pelo método antigo, e relate.

**b) Dica:** No item 6, você pode comparar o resultado do método antigo com o do novo para os mesmos dados, mostrando o tamanho do estrago.

**c) Como testar:**

```
LABSTUDY>DO ##class(LabStudy.Reports).AgeDistribution()
LABSTUDY>DO ##class(LabStudy.Reports).TurnaroundReport()
LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/DateTime.cls`:

```objectscript
/// Date and time handling for the LabStudy system.
/// All date logic lives here so there is exactly one place to fix.
Class LabStudy.DateTime Extends %RegisteredObject
{

Parameter DAYNAMES = "domingo,segunda,terca,quarta,quinta,sexta,sabado";

Parameter DAYSHORT = "dom,seg,ter,qua,qui,sex,sab";

/// Today, as a %Date logical value.
ClassMethod Today() As %Date [ CodeMode = expression ]
{
+$HOROLOG
}

/// Seconds since midnight, as a %Time logical value.
ClassMethod Now() As %Time [ CodeMode = expression ]
{
$PIECE($HOROLOG, ",", 2)
}

/// Current moment as a %TimeStamp.
ClassMethod NowTimestamp() As %TimeStamp [ CodeMode = expression ]
{
$ZDATETIME($HOROLOG, 3)
}

/// Age in complete years. Correct: it checks whether the birthday has passed.
ClassMethod Age(birthDate As %Date, reference As %Date = "") As %Integer
{
    quit:birthDate="" ""
    set:reference="" reference = ..Today()
    quit:birthDate>reference ""

    set b = ..Format(birthDate, 3)
    set r = ..Format(reference, 3)
    quit:(b = "") || (r = "") ""

    set age = $PIECE(r, "-", 1) - $PIECE(b, "-", 1)

    // MM-DD turned into a 4 digit number: "08-19" -> 819, "05-17" -> 517.
    // Comparing "08-19" < "05-17" directly would be NUMERIC and read
    // only the month, silently ignoring the day.
    set refMMDD = +$TRANSLATE($EXTRACT(r, 6, 10), "-", "")
    set birthMMDD = +$TRANSLATE($EXTRACT(b, 6, 10), "-", "")

    if refMMDD < birthMMDD {
        set age = age - 1
    }
    quit age
}

/// 0 = Sunday .. 6 = Saturday
ClassMethod DayOfWeekNumber(date As %Date) As %Integer [ CodeMode = expression ]
{
(date + 4) # 7
}

/// Day of week name.
ClassMethod DayOfWeek(date As %Date, short As %Boolean = 0) As %String
{
    quit:date="" ""
    set names = $LISTFROMSTRING($SELECT(short: ..#DAYSHORT, 1: ..#DAYNAMES), ",")
    quit $LISTGET(names, ..DayOfWeekNumber(date) + 1)
}

ClassMethod IsBusinessDay(date As %Date) As %Boolean
{
    quit:date="" 0
    set dow = ..DayOfWeekNumber(date)
    quit (dow > 0) && (dow < 6)
}

ClassMethod AddBusinessDays(date As %Date, days As %Integer) As %Date
{
    quit:date="" ""
    quit:days=0 date

    set step = $SELECT(days > 0: 1, 1: -1)
    set remaining = $ZABS(days)
    set current = date

    while remaining > 0 {
        set current = current + step
        set:..IsBusinessDay(current) remaining = remaining - 1
    }
    quit current
}

ClassMethod BusinessDaysBetween(startDate As %Date, endDate As %Date) As %Integer
{
    quit:(startDate = "") || (endDate = "") ""
    quit:startDate>endDate -..BusinessDaysBetween(endDate, startDate)

    set count = 0
    for d = (startDate + 1):1:endDate {
        set:..IsBusinessDay(d) count = count + 1
    }
    quit count
}

/// Formats a date. Format 3 (ODBC) is the internal default.
ClassMethod Format(date As %Date, format As %Integer = 3) As %String
{
    quit:date="" ""

    set out = ""
    try {
        set out = $ZDATE(date, format)
    } catch {
        set out = ""
    }
    quit out
}

/// Parses a date text. Returns "" when it is not a valid date.
ClassMethod Parse(text As %String, format As %Integer = 3) As %Date
{
    set text = $ZSTRIP($GET(text), "<>W")
    quit:text="" ""

    if (format = 3) && '(text ? 4N1"-"2N1"-"2N) {
        quit ""
    }

    set out = ""
    try {
        set out = $ZDATEH(text, format)
    } catch {
        set out = ""
    }
    quit out
}

/// Hours between two %TimeStamp values.
ClassMethod TurnaroundHours(fromTS As %TimeStamp, toTS As %TimeStamp, decimals As %Integer = 2) As %Numeric
{
    quit:(fromTS = "") || (toTS = "") ""

    set h1 = "", h2 = ""
    try {
        set h1 = $ZDATETIMEH(fromTS, 3)
        set h2 = $ZDATETIMEH(toTS, 3)
    } catch {
        quit ""
    }
    quit:(h1 = "") || (h2 = "") ""

    set s1 = (+h1 * 86400) + $PIECE(h1, ",", 2)
    set s2 = (+h2 * 86400) + $PIECE(h2, ",", 2)

    quit $NORMALIZE((s2 - s1) / 3600, decimals)
}

ClassMethod LastDayOfMonth(year As %Integer, month As %Integer) As %Date
{
    set nm = month + 1, ny = year
    if nm > 12 { set nm = 1, ny = year + 1 }

    quit ..Parse(ny_"-"_$TRANSLATE($JUSTIFY(nm, 2), " ", "0")_"-01", 3) - 1
}

ClassMethod MonthRange(year As %Integer, month As %Integer, Output first As %Date, Output last As %Date) As %Status
{
    set first = ..Parse(year_"-"_$TRANSLATE($JUSTIFY(month, 2), " ", "0")_"-01", 3)
    set last = ..LastDayOfMonth(year, month)
    quit $$$OK
}

/// Leap year, asked to the calendar rather than computed by rule.
ClassMethod IsLeapYear(year As %Integer) As %Boolean [ CodeMode = expression ]
{
..Format(..LastDayOfMonth(year, 2), 3) [ "-29"
}

/// The old, WRONG age calculation. Kept only so the migration can measure
/// how much damage it caused. Never call this from application code.
ClassMethod AgeWrongLegacy(birthDate As %Date) As %String [ Internal ]
{
    quit:birthDate="" ""
    quit $ZDATE($HOROLOG, 4) - $ZDATE(birthDate, 4)
}

}
```

Corrija `src/LabStudy/Patient.cls`:

```objectscript
/// Age in complete years. Never stored: it changes on its own every year.
Method AgeGet() As %Integer [ CodeMode = expression ]
{
##class(LabStudy.DateTime).Age(..BirthDate)
}
```

Acrescente a `src/LabStudy/Exam.cls`:

```objectscript
/// Hours between collection and result. Empty while pending.
Method TurnaroundHours() As %Numeric
{
    quit:..ResultStatus'="final" ""
    quit ##class(LabStudy.DateTime).TurnaroundHours(..CollectedOn, ..ResultDate)
}

/// True when the exam is still pending after the given number of hours.
Method IsOverdue(limitHours As %Numeric = 24) As %Boolean
{
    quit:..ResultStatus'="pending" 0

    set elapsed = ##class(LabStudy.DateTime).TurnaroundHours(
                     ..CollectedOn, ##class(LabStudy.DateTime).NowTimestamp())

    quit:elapsed="" 0
    quit elapsed > limitHours
}
```

Acrescente a `src/LabStudy/Reports.cls`:

```objectscript
/// Patients grouped by age band.
ClassMethod AgeDistribution() As %Status
{
    kill band

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT BirthDate FROM LabStudy.PATIENT WHERE BirthDate IS NOT NULL")

    set total = 0, unknown = 0
    while rs.%Next() {
        set age = ##class(LabStudy.DateTime).Age(rs.%Get("BirthDate"))

        if age = "" {
            set unknown = unknown + 1
            continue
        }

        set total = total + 1
        set key = $CASE(age \ 10, 0: "00-09", 1: "10-19", 2: "20-29", 3: "30-39",
                                  4: "40-49", 5: "50-59", 6: "60-69", 7: "70-79",
                                  : "80+")
        set band(key) = $GET(band(key), 0) + 1
    }

    write "=== age distribution ===", !

    set W = $LISTBUILD(10, 8, 8, 22)
    set A = $LISTBUILD("L", "R", "R", "L")
    do ##class(LabStudy.Formatter).Header($LISTBUILD("band", "count", "%", ""), W, A)

    set k = ""
    for {
        set k = $ORDER(band(k), 1, n)
        quit:k=""

        set pct = ##class(LabStudy.Demo.Md2).Percent(n, total, 1)
        set bar = $TRANSLATE($JUSTIFY("", n \ 5), " ", "#")

        do ##class(LabStudy.Formatter).Row($LISTBUILD(k, n, pct_"%", bar), W, A)
    }

    do ##class(LabStudy.Formatter).Line(48)
    write "  ", total, " patients with a birth date"
    write $SELECT(unknown: ", "_unknown_" unknown", 1: ""), !

    quit $$$OK
}

/// Average, minimum and maximum turnaround per test code.
ClassMethod TurnaroundReport() As %Status
{
    kill stat

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, TestCode AS Code FROM LabStudy.EXAM "
        _"WHERE ResultStatus = 'final'")

    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)

        set hours = exam.TurnaroundHours()
        continue:hours=""

        set code = rs.%Get("Code")
        set stat(code, "n") = $GET(stat(code, "n"), 0) + 1
        set stat(code, "sum") = $GET(stat(code, "sum"), 0) + hours

        if '$DATA(stat(code, "min")) || (hours < stat(code, "min")) {
            set stat(code, "min") = hours
        }
        if '$DATA(stat(code, "max")) || (hours > stat(code, "max")) {
            set stat(code, "max") = hours
        }
    }

    write "=== turnaround per test code (hours) ===", !

    set W = $LISTBUILD(8, 6, 10, 10, 10)
    set A = $LISTBUILD("L", "R", "R", "R", "R")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("code", "n", "average", "min", "max"), W, A)

    set code = ""
    for {
        set code = $ORDER(stat(code))
        quit:code=""

        set n = stat(code, "n")
        set avg = ##class(LabStudy.Demo.Md2).Round(stat(code, "sum") / n, 2)

        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(code, n,
                       ##class(LabStudy.Text).Number(avg, 2),
                       ##class(LabStudy.Text).Number(stat(code, "min"), 2),
                       ##class(LabStudy.Text).Number(stat(code, "max"), 2)),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(48)
    quit $$$OK
}

/// Exams pending for longer than the limit.
ClassMethod OverdueExams(limitHours As %Numeric = 24) As %Integer
{
    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id FROM LabStudy.EXAM WHERE ResultStatus = 'pending'")

    write "=== exams pending for more than ", limitHours, " hours ===", !

    set W = $LISTBUILD(8, 10, 22, 12)
    set A = $LISTBUILD("R", "L", "L", "R")
    do ##class(LabStudy.Formatter).Header(
        $LISTBUILD("id", "code", "patient", "hours"), W, A)

    set found = 0
    while rs.%Next() {
        set exam = ##class(LabStudy.Exam).%OpenId(rs.%Get("Id"))
        continue:'$ISOBJECT(exam)
        continue:'exam.IsOverdue(limitHours)

        set hours = ##class(LabStudy.DateTime).TurnaroundHours(
                        exam.CollectedOn, ##class(LabStudy.DateTime).NowTimestamp())

        set patientName = ""
        set:$ISOBJECT(exam.Patient) patientName = exam.Patient.Name

        set found = found + 1
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(exam.%Id(), exam.TestCode, patientName,
                       ##class(LabStudy.Text).Number(hours, 1)),
            W, A)
    }

    do ##class(LabStudy.Formatter).Line(56)
    write "  ", found, " overdue", !
    quit found
}
```

Acrescente a `src/LabStudy/Schema.cls` (e suba `LATEST` para 5):

```objectscript
Parameter LATEST = 5;

/// Step 5: measures the damage caused by the old, wrong age calculation.
/// Changes nothing: Age is a calculated property, so fixing the method
/// fixed every reading at once. This step exists to record the impact.
ClassMethod Step5(Output affected As %Integer) As %Status
{
    set affected = 0

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT %ID AS Id, BirthDate AS B FROM LabStudy.PATIENT "
        _"WHERE BirthDate IS NOT NULL")

    set checked = 0, wrong = 0, worst = 0

    while rs.%Next() {
        set b = rs.%Get("B")
        continue:b=""

        set checked = checked + 1

        set correct = ##class(LabStudy.DateTime).Age(b)
        set legacy = ##class(LabStudy.DateTime).AgeWrongLegacy(b)

        continue:correct=legacy

        set wrong = wrong + 1
        set diff = $ZABS(correct - legacy)
        set:diff>worst worst = diff
    }

    write "  checked          : ", checked, !
    write "  would be wrong   : ", wrong, !
    write "  worst difference : ", worst, " years", !

    set affected = wrong
    quit $$$OK
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.7";
```

Execução esperada:

```
LABSTUDY>DO ##class(LabStudy.Schema).Upgrade()
schema version 4 -> target 5

--- step 5 ---
  checked          : 200
  would be wrong   : 200
  worst difference : 20260819 years
step 5 ok: 200 rows in 0.412s

schema version is now 5

LABSTUDY>SET p = ##class(LabStudy.Patient).%OpenId(1)
LABSTUDY>WRITE p.Name, " nasceu em ", ##class(LabStudy.DateTime).Format(p.BirthDate), !
Paciente 00001 nasceu em 1988-03-14

LABSTUDY>WRITE "idade correta: ", p.Age, !
idade correta: 38

LABSTUDY>DO ##class(LabStudy.Reports).AgeDistribution()
=== age distribution ===
band        count        %
------------------------------------------------
20-29          31    15.5%  ######
30-39          48    24.0%  #########
40-49          52    26.0%  ##########
50-59          41    20.5%  ########
60-69          28    14.0%  #####
------------------------------------------------
  200 patients with a birth date

LABSTUDY>DO ##class(LabStudy.Reports).TurnaroundReport()
=== turnaround per test code (hours) ===
code        n    average       min       max
------------------------------------------------
CHOL      250       0,00      0,00      0,00
GLU       250       0,00      0,00      0,00
...
```

**Por que cada decisão:**

- **A correção do `AgeGet` consertou o sistema inteiro de uma vez**, porque `Age` é uma propriedade **calculada** (Capítulo 2). Nenhum dado precisou ser migrado. **Esta é a recompensa concreta daquela decisão de modelagem**: se a idade tivesse sido gravada em disco, seria preciso recalcular 200 registros — e, se o erro tivesse passado despercebido por anos, milhões.
- **O `Step5` mediu o estrago em vez de corrigi-lo**, porque não havia nada a corrigir nos dados. Uma migração que apenas **documenta** o impacto de um bug corrigido é legítima e valiosa: ela deixa registrado, no histórico do esquema, quando o problema foi encontrado e qual era o seu tamanho.
- **A "pior diferença" saiu absurda (`20260819` anos)**, e isso é revelador: o método antigo não estava calculando uma idade ligeiramente errada — estava subtraindo duas datas em formato `DD/MM/AAAA` como se fossem números, produzindo lixo completo. **Um valor tão obviamente absurdo deveria ter sido notado.** Que não tenha sido é a lição: ninguém conferiu a saída, e a apostila reproduziu o erro por vários capítulos. Sistemas reais escondem erros assim por anos.
- **`AgeWrongLegacy` está marcado como `[ Internal ]`** e com um comentário explícito dizendo para nunca chamá-lo. Manter código errado no repositório exige essa sinalização; caso contrário, alguém o reutilizará.
- **`TurnaroundHours` devolve vazio para exames não finais.** Um exame pendente não tem tempo de resposta — tem tempo de espera, que é `IsOverdue`. Confundir os dois produziria estatísticas sem sentido.
- **Os tempos de resposta saíram todos em zero** porque os dados de teste foram gerados com `CollectedOn` e `ResultDate` no mesmo instante. Isso não é um defeito do relatório: é um defeito dos **dados de teste**. Um bom exercício adicional é alterar o `LoadTest` para distribuir os horários e ver o relatório ganhar vida — **relatórios só se validam com dados que variam**.
- **`AgeDistribution` usa `$CASE(age \ 10, ...)`** para produzir a faixa: a divisão inteira por 10 transforma qualquer idade na dezena correspondente, e o `$CASE` a rotula. É mais legível do que uma cadeia de `if` com limites repetidos.
- **Todos os relatórios usam `LabStudy.Formatter` e `LabStudy.Text.Number`** dos capítulos anteriores. As camadas estão se acumulando: data e hora em `DateTime`, texto em `Text`, formatação em `Formatter`, ordenação em `Sorter`, listas em `ListUtil`. Cada uma existe num lugar só — e cada correção futura terá exatamente um lugar para acontecer.

---

## 9. Quiz do capítulo

**Q1.** Quanto vale `2 + 3 * 4` no ObjectScript?

- A) `14`
- B) `20`
- C) `24`
- D) Erro de sintaxe.

---

**Q2.** Quanto vale `1 ! 0 & 0`?

- A) `1`
- B) `0`
- C) `2`
- D) Erro.

---

**Q3.** Qual é a regra de avaliação de expressões do ObjectScript?

- A) Precedência matemática convencional.
- B) Estritamente da esquerda para a direita, sem precedência entre operadores.
- C) Da direita para a esquerda.
- D) Depende da configuração da instância.

---

**Q4.** O que faz o operador `\`?

- A) Divisão comum.
- B) Divisão inteira.
- C) Resto da divisão.
- D) Potência.

---

**Q5.** `7 # 2` devolve o quê?

- A) `3.5`
- B) `3`
- C) `1`
- D) `49`

---

**Q6.** Qual é a diferença entre `&` e `&&`?

- A) Nenhuma.
- B) `&&` tem curto-circuito: para de avaliar assim que o resultado está decidido.
- C) `&` é para números e `&&` para texto.
- D) `&&` é mais lento.

---

**Q7.** O que acontece se nenhuma condição de um `$SELECT` for verdadeira?

- A) Devolve vazio.
- B) Devolve zero.
- C) Ocorre `<ILLEGAL VALUE>`.
- D) Devolve o último valor.

---

**Q8.** Quando usar `$CASE` em vez de `$SELECT`?

- A) Nunca; são equivalentes.
- B) Quando você compara **um mesmo valor** contra várias alternativas.
- C) Quando as condições são faixas numéricas.
- D) Quando há mais de cinco opções.

---

**Q9.** O que representa `$HOROLOG`?

- A) A hora atual em texto.
- B) Dias desde 31/12/1840 e segundos desde a meia-noite, separados por vírgula.
- C) O timestamp em UTC.
- D) Milissegundos desde 1970.

---

**Q10.** Como obter apenas a parte de **data** do `$HOROLOG`?

- A) `$HOROLOG`
- B) `+$HOROLOG`
- C) `$PIECE($HOROLOG, ",", 2)`
- D) `$ZDATE($HOROLOG)`

---

**Q11.** Qual formato de `$ZDATE` produz `2026-08-19`?

- A) `1`
- B) `2`
- C) `3`
- D) `4`

---

**Q12.** Por que usar o formato 3 (ODBC) internamente?

- A) É o mais rápido.
- B) É inequívoco, ordena corretamente como texto e é o formato universal de intercâmbio.
- C) É o único que `$ZDATEH` aceita.
- D) É o padrão brasileiro.

---

**Q13.** Como calcular a diferença em dias entre duas datas?

- A) Com uma função específica de diferença.
- B) Subtraindo os valores lógicos: `data2 - data1`.
- C) Convertendo para texto e comparando.
- D) Contando num laço.

---

**Q14.** Por que a idade não pode ser calculada apenas subtraindo os anos?

- A) Porque os anos têm tamanhos diferentes.
- B) Porque é preciso descontar 1 quando o aniversário ainda não chegou no ano de referência.
- C) Porque `$ZDATE` não devolve o ano.
- D) Porque anos bissextos atrapalham.

---

**Q15.** Qual é a forma recomendada de descobrir o último dia de um mês?

- A) Implementar a regra dos meses e dos anos bissextos.
- B) Ir para o primeiro dia do mês seguinte e subtrair 1.
- C) Usar `$ZDATE` com formato especial.
- D) Somar 30 e ajustar.

---

**Q16.** `0.1 + 0.2` no ObjectScript devolve o quê?

- A) `0.30000000000000004`
- B) `0.3` exato, porque a aritmética padrão é decimal.
- C) Erro de precisão.
- D) `0.29999999`

---

**Q17.** Qual função gera aleatoriedade adequada para **segurança**?

- A) `$RANDOM(n)`
- B) `$SYSTEM.Encryption.GenCryptRand(n)`
- C) `$HOROLOG`
- D) `$NOW()`

---

**Q18.** O que o segundo argumento de `$ISVALIDNUM(valor, n)` significa?

- A) A quantidade exata de casas decimais que o valor precisa ter.
- B) O **número máximo** de casas decimais aceito.
- C) O valor mínimo permitido.
- D) A base numérica.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** avaliação da esquerda para a direita: `(2+3)*4` = 20.
- **A está errada:** seria o resultado com precedência matemática, que o ObjectScript não usa.
- **C e D estão erradas:** não correspondem a nenhuma ordem de avaliação.

**Q2 — Resposta: B.**
- **B está certa:** `1!0` → `1`; `1&0` → `0`.
- **A está errada:** seria o resultado com precedência de "e" sobre "ou".
- **C e D estão erradas:** operadores lógicos devolvem 1 ou 0.

**Q3 — Resposta: B.**
- **B está certa:** não há hierarquia entre operadores; use parênteses.
- **A está errada:** é justamente a expectativa que a linguagem quebra.
- **C está errada:** a direção é da esquerda para a direita.
- **D está errada:** não é configurável.

**Q4 — Resposta: B.**
- **B está certa:** `\` é divisão inteira. `7 \ 2` = 3.
- **A está errada:** divisão comum é `/`.
- **C está errada:** resto é `#`.
- **D está errada:** potência é `**`.

**Q5 — Resposta: C.**
- **C está certa:** `#` devolve o resto: 7 dividido por 2 deixa resto 1.
- **A está errada:** seria `7 / 2`.
- **B está errada:** seria `7 \ 2`.
- **D está errada:** seria `7 ** 2`.

**Q6 — Resposta: B.**
- **B está certa:** `&&` e `||` interrompem a avaliação quando o resultado já está determinado, o que evita erros na segunda condição.
- **A está errada:** a diferença é significativa.
- **C está errada:** ambos são lógicos.
- **D está errada:** o curto-circuito tende a ser mais rápido.

**Q7 — Resposta: C.**
- **C está certa:** `$SELECT` exige que alguma condição seja verdadeira; inclua sempre o ramo `1:`.
- **A, B e D estão erradas:** não há valor padrão implícito.

**Q8 — Resposta: B.**
- **B está certa:** `$CASE` compara um valor contra alternativas; `$SELECT` avalia condições distintas.
- **A está errada:** a legibilidade difere muito conforme o caso.
- **C está errada:** faixas são o caso do `$SELECT`.
- **D está errada:** a quantidade não é o critério.

**Q9 — Resposta: B.**
- **B está certa:** é o formato `dias,segundos`, com o dia zero em 31/12/1840.
- **A está errada:** é numérico, não texto formatado.
- **C está errada:** UTC é o `$ZTIMESTAMP`.
- **D está errada:** essa é a convenção de outros sistemas.

**Q10 — Resposta: B.**
- **B está certa:** o `+` força a leitura numérica e descarta a parte após a vírgula.
- **A está errada:** devolve os dois componentes.
- **C está errada:** devolve os **segundos**.
- **D está errada:** devolve texto formatado, não o valor lógico.

**Q11 — Resposta: C.**
- **C está certa:** formato 3 é o ODBC, `AAAA-MM-DD`.
- **A está errada:** formato 1 é mês/dia/ano.
- **B está errada:** formato 2 é por extenso abreviado.
- **D está errada:** formato 4 é dia/mês/ano.

**Q12 — Resposta: B.**
- **B está certa:** não confunde dia com mês, ordena corretamente como texto e é padrão de intercâmbio.
- **A está errada:** desempenho não é o critério.
- **C está errada:** `$ZDATEH` aceita vários formatos.
- **D está errada:** o padrão brasileiro é o formato 4.

**Q13 — Resposta: B.**
- **B está certa:** como datas são números de dias, a diferença é uma subtração.
- **A está errada:** não é necessária uma função específica.
- **C e D estão erradas:** desnecessariamente complicados e propensos a erro.

**Q14 — Resposta: B.**
- **B está certa:** quem nasceu em dezembro ainda não fez aniversário em agosto; subtrair os anos superestimaria a idade.
- **A está errada:** o tamanho dos anos é tratado pela própria contagem de dias.
- **C está errada:** o ano se obtém do texto ODBC.
- **D está errada:** bissextos não são o problema aqui.

**Q15 — Resposta: B.**
- **B está certa:** o conversor de datas já conhece o calendário; ir ao dia 1 do mês seguinte e voltar um dia funciona sempre, inclusive em fevereiro bissexto.
- **A está errada:** reimplementar regras de calendário é fonte clássica de erros, especialmente no caso de anos como 1900.
- **C está errada:** não há formato que devolva isso.
- **D está errada:** falharia em quatro meses do ano.

**Q16 — Resposta: B.**
- **B está certa:** a aritmética padrão do ObjectScript é decimal de alta precisão.
- **A e D estão erradas:** é o que aconteceria com `$DOUBLE`, ou em linguagens com ponto flutuante binário por padrão.
- **C está errada:** não há erro.

**Q17 — Resposta: B.**
- **B está certa:** `GenCryptRand` gera aleatoriedade de qualidade criptográfica.
- **A está errada:** `$RANDOM` é previsível e serve para simulações, não para segurança.
- **C e D estão erradas:** são altamente previsíveis.

**Q18 — Resposta: B.**
- **B está certa:** é um teto. `$ISVALIDNUM("12.5", 1)` devolve `1` (uma casa, dentro do limite) e `$ISVALIDNUM("12.55", 1)` devolve `0` (duas casas, acima do limite).
- **A está errada:** não é exigência de ter aquelas casas. `$ISVALIDNUM("12", 1)` também passa.
- **C está errada:** mínimo e máximo são o terceiro e o quarto argumentos.
- **D está errada:** `$ISVALIDNUM` não trabalha com bases.

---

## 10. Resumo relâmpago

1. **O ObjectScript avalia estritamente da esquerda para a direita**, sem precedência entre operadores. `2 + 3 * 4` = **20**.
2. **Use parênteses em toda expressão com mais de um operador.** Sempre.
3. Operadores: `+ - * /`, **`\`** divisão inteira, **`#`** resto, **`**`** potência.
4. Divisão por zero causa **`<DIVIDE>`**. Proteja com `$SELECT`.
5. A aritmética padrão é **decimal exata**: `0.1 + 0.2` dá `0.3`. `$DOUBLE` introduz ponto flutuante e imprecisão.
6. Arredondamento: **`$NORMALIZE(v, casas)`** é o mais direto; `$JUSTIFY` e `$FNUMBER` arredondam **e** formatam. **Funções diferentes podem arredondar diferente** — padronize.
7. **`$ISVALIDNUM(v, casas, min, max)`** valida valor numérico e faixa; o operador `?` valida **formato**.
8. **`$RANDOM(n)`** devolve 0 a n-1 e **não serve para segurança**.
9. Lógicos: `'` não, `&` e, `!` ou, **`&&`** e com curto-circuito, **`||`** ou com curto-circuito.
10. **Use `&&` e `||`** quando a segunda condição depender da primeira — é o que evita `<INVALID OREF>`.
11. **`!` é "ou" em expressão e quebra de linha no `WRITE`.**
12. **`$SELECT`** avalia condições e exige o ramo **`1:`**; **`$CASE`** compara um valor contra alternativas, com o padrão sem valor antes dos dois-pontos.
13. **`$HOROLOG`** = `dias,segundos`, com dia zero em **31/12/1840**. `+$HOROLOG` dá só os dias.
14. **`$ZTIMESTAMP`** é UTC com fração; **`$NOW()`** é local com fração. **Grave em UTC, exiba em local.**
15. `$ZDATE(dias, formato)` / `$ZDATEH(texto, formato)`. Formatos: **1** mês/dia/ano, **2** por extenso, **3 ODBC**, **4** dia/mês/ano, **8** compacto. **Nenhum devolve só o ano.**
16. **Use o formato 3 internamente**; formatos locais só na apresentação.
17. `$ZTIME` / `$ZTIMEH` para hora; `$ZDATETIME` / `$ZDATETIMEH` para os dois juntos.
18. **Diferença em dias é uma subtração.** Somar dias é somar.
19. **Idade exige descontar 1 se o aniversário ainda não chegou.** Compare `MM-DD` como texto de largura fixa.
20. **Dia da semana:** `(dias + 4) # 7`, com 0 = domingo — verifique a premissa na sua instalação.
21. **Último dia do mês:** vá ao dia 1 do mês seguinte e subtraia 1. Isso resolve bissexto de graça.
22. **Não reimplemente regras de calendário.** Pergunte ao conversor de datas.
23. `$ZDATEH` com texto inválido **gera erro**: valide o formato com `?` e proteja com `try`/`catch`. E lembre que `"2026-02-30"` passa no padrão e não existe.
24. Concentre toda a lógica de data e hora numa classe só — quando houver um erro, haverá **um** lugar para corrigir.

---

## 11. Cartões de memorização

**Frente:** Quanto vale `2 + 3 * 4` no ObjectScript?
**Verso:** `20`. A avaliação é estritamente da esquerda para a direita, sem precedência.

**Frente:** Qual é a regra de ouro para expressões no ObjectScript?
**Verso:** Usar parênteses em toda expressão com mais de um operador.

**Frente:** O que fazem `\` e `#`?
**Verso:** `\` é divisão inteira (`7\2` = 3); `#` é o resto (`7#2` = 1).

**Frente:** Diferença entre `&` e `&&`.
**Verso:** `&&` tem curto-circuito: não avalia a segunda condição se a primeira já decidiu.

**Frente:** Por que usar `&&` ao testar um objeto antes de acessá-lo?
**Verso:** Porque sem curto-circuito a segunda condição é avaliada mesmo com o objeto vazio, causando `<INVALID OREF>`.

**Frente:** O que acontece num `$SELECT` sem condição verdadeira?
**Verso:** Erro `<ILLEGAL VALUE>`. Sempre inclua o ramo `1:`.

**Frente:** Quando usar `$CASE` em vez de `$SELECT`?
**Verso:** Quando você compara **um mesmo valor** contra várias alternativas.

**Frente:** O que é `$HOROLOG`?
**Verso:** `dias,segundos` — dias desde 31/12/1840 e segundos desde a meia-noite.

**Frente:** Como obter "hoje" no formato lógico de `%Date`?
**Verso:** `+$HOROLOG`.

**Frente:** Qual a diferença entre `$HOROLOG` e `$ZTIMESTAMP`?
**Verso:** `$HOROLOG` é local com precisão de segundo; `$ZTIMESTAMP` é **UTC** com fração de segundo.

**Frente:** Qual formato de `$ZDATE` produz `AAAA-MM-DD`?
**Verso:** O formato **3** (ODBC). Use-o internamente.

**Frente:** O formato 4 do `$ZDATE` devolve o quê?
**Verso:** A data completa em `DD/MM/AAAA`. **Não** devolve o ano.

**Frente:** Como calcular a diferença em dias entre duas datas?
**Verso:** Subtraindo os valores lógicos: `data2 - data1`.

**Frente:** Por que subtrair os anos não basta para calcular idade?
**Verso:** É preciso descontar 1 quando o aniversário ainda não chegou no ano de referência.

**Frente:** Como comparar `MM-DD` de duas datas?
**Verso:** Como texto, usando `$EXTRACT(dataODBC, 6, 10)` — funciona porque a largura é fixa e há zeros à esquerda.

**Frente:** Como descobrir o dia da semana?
**Verso:** `(dias + 4) # 7`, com 0 = domingo. Confirme a premissa do dia zero na sua instalação.

**Frente:** Como obter o último dia de um mês?
**Verso:** Ir ao dia 1 do mês seguinte e subtrair 1. Resolve o ano bissexto automaticamente.

**Frente:** Como testar se um ano é bissexto sem escrever a regra?
**Verso:** Verificar se o último dia de fevereiro daquele ano é dia 29.

**Frente:** `$ZDATEH` com texto inválido faz o quê?
**Verso:** Gera erro. Valide o formato antes com `?` e proteja com `try`/`catch`.

**Frente:** `"2026-02-30"` passa no padrão `4N1"-"2N1"-"2N`?
**Verso:** Passa — e não é uma data válida. Padrão valida forma, não existência.

**Frente:** Quanto dá `0.1 + 0.2` no ObjectScript?
**Verso:** `0.3` exato. A aritmética padrão é decimal, não ponto flutuante binário.

**Frente:** Qual função gera aleatoriedade para segurança?
**Verso:** `$SYSTEM.Encryption.GenCryptRand()`. `$RANDOM` serve para simulação, não para segurança.

**Frente:** Onde deve viver a lógica de data e hora de um sistema?
**Verso:** Numa classe só. Quando houver um erro — e haverá —, haverá um único lugar para corrigi-lo.

---

Digite CONTINUAR para o próximo capítulo.
