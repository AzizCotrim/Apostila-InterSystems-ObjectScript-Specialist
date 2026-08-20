# Apostila InterSystems ObjectScript Specialist
## Capítulo 15 — T4.3 Manipulates Strings (Manipulando textos)

> Ainda em **T4 — Functions & APIs**, o domínio de maior peso da prova. Este é o capítulo com mais funções para conhecer — e a boa notícia é que quase todas seguem os mesmos padrões de argumentos. Aprendendo o padrão, você aprende o conjunto.

---

## 1. O que você vai saber fazer ao terminar

1. Medir textos com **`$LENGTH`**, nas duas formas: caracteres e pedaços.
2. Recortar com **`$EXTRACT`**, inclusive contando a partir do fim com `*`.
3. Trabalhar com campos separados usando **`$PIECE`**, e alterá-los no lugar.
4. Localizar com **`$FIND`** e com o operador **`[`**, sabendo a diferença.
5. Substituir com **`$REPLACE`** e mapear caracteres com **`$TRANSLATE`**.
6. Limpar com **`$ZSTRIP`** e converter maiúsculas/minúsculas com **`$ZCONVERT`**.
7. Alinhar e formatar com **`$JUSTIFY`** e **`$FNUMBER`**.
8. Converter entre caractere e código com **`$CHAR`** e **`$ASCII`**.
9. Inverter com **`$REVERSE`** e concatenar com **`_`**.
10. Comparar textos com `=`, `<`, `>`, **`[`**, **`]`** e **`]]`**.
11. Validar formatos com o operador de **casamento de padrão `?`**.
12. Entender **maiúsculas na ordenação**, **Unicode** e o limite **`<MAXSTRING>`**.
13. Escolher entre `$PIECE`, `$LIST` e `$EXTRACT` conscientemente.
14. Levar o projeto à versão **1.6**, com formatação, validação e mascaramento de dados.

---

## 2. O conceito em linguagem de gente

### 2.1 Três formas de olhar para um texto

O ObjectScript oferece três "óculos" diferentes para o mesmo texto, e escolher o certo resolve metade do problema.

Pegue o texto `"Maria;Silva;36"`.

**Óculos 1 — o texto como sequência de caracteres.**
Você vê 15 posições. O caractere 1 é `M`, o 6 é `;`. A ferramenta é **`$EXTRACT`**.

Use quando a posição importa: pegar as duas primeiras letras, os últimos 4 dígitos, o caractere do meio.

**Óculos 2 — o texto como campos separados.**
Você vê 3 pedaços delimitados por `;`. O pedaço 1 é `Maria`, o 3 é `36`. A ferramenta é **`$PIECE`**.

Use quando o texto carrega campos: CSV, mensagens de protocolo, chaves compostas.

**Óculos 3 — o texto como conteúdo a procurar.**
Você quer saber se `"Silva"` aparece, e onde. As ferramentas são **`$FIND`** e o operador **`[`**.

Use quando busca, e não posição, é a questão.

Grande parte dos erros de manipulação de texto vem de usar os óculos errados — por exemplo, contar caracteres para achar o terceiro campo, quando `$PIECE` faria isso sozinho e continuaria funcionando se os campos mudassem de tamanho.

### 2.2 A convenção do asterisco

Muitas funções de texto do IRIS aceitam **`*`** significando **"o último"**, e expressões como **`*-2`** significando "o terceiro a contar do fim".

```objectscript
$EXTRACT(texto, *)          // último caractere
$EXTRACT(texto, *-3, *)     // os quatro últimos
$EXTRACT(texto, 2, *)       // do segundo até o fim
```

Isso poupa você de escrever `$LENGTH(texto)` a toda hora, e — mais importante — **evita o erro de calcular a posição errada**. Sempre que quiser algo relativo ao fim, procure o `*`.

### 2.3 Modificar no lugar

Duas funções aceitam ficar do **lado esquerdo** de um `SET`:

```objectscript
set $EXTRACT(texto, 1) = "X"           // troca o primeiro caractere
set $PIECE(texto, ";", 2) = "Souza"    // troca o segundo campo
```

Isso é muito mais legível do que remontar o texto por concatenação, e é idiomático no ObjectScript. Você já viu o mesmo com `$LIST` no capítulo anterior.

Há um detalhe elegante: se você atribuir a um pedaço que **não existe**, o IRIS **cria** os separadores necessários:

```objectscript
set t = "a"
set $PIECE(t, ";", 4) = "d"
write t, !                              // a;;;d
```

Isso é útil para montar registros posicionais sem contar separadores na mão.

### 2.4 Substituir texto ou mapear caracteres

Duas funções parecidas que fazem coisas diferentes, e a prova adora:

**`$REPLACE(texto, procurado, novo)`** — troca **um pedaço de texto** por outro.

```objectscript
write $REPLACE("banana", "an", "X"), !       // bXXa
```

**`$TRANSLATE(texto, de, para)`** — mapeia **caractere por caractere**.

```objectscript
write $TRANSLATE("banana", "an", "XY"), !    // bXYXYX
```

Leia o `$TRANSLATE` assim: *"todo `a` vira `X`, todo `n` vira `Y`"*. Ele não procura a sequência `"an"`; ele trata cada caractere isoladamente.

E há um comportamento do `$TRANSLATE` que é a razão principal de ele existir: **se o segundo texto for mais curto que o primeiro, os caracteres sobrando são REMOVIDOS**.

```objectscript
write $TRANSLATE("a-b.c,d", "-.,", ""), !    // abcd  -- removeu os três
```

Essa é a forma mais rápida de eliminar um conjunto de caracteres de uma vez.

Resumo da decisão:

| Preciso... | Uso |
|---|---|
| trocar a palavra `"Rua"` por `"R."` | `$REPLACE` |
| trocar todo `a` por `4` e todo `e` por `3` | `$TRANSLATE` |
| remover todos os pontos, hífens e barras | `$TRANSLATE(t, "-./", "")` |
| remover espaços das pontas | `$ZSTRIP(t, "<>W")` |

### 2.5 Comparar textos: mais operadores do que parece

Além de `=` e `'=`, o ObjectScript tem operadores próprios para texto:

| Operador | Significado | Exemplo verdadeiro |
|---|---|---|
| `[` | **contém** | `"Maria Silva" [ "Silva"` |
| `]` | **vem depois** na ordenação de texto | `"b" ] "a"` |
| `]]` | **vem depois** na ordenação padrão (números antes de texto) | `"a" ]] 10` |
| `<` `>` | comparação **numérica** | `10 > 9` |

O par `[` e `]` merece atenção porque são visualmente parecidos e fazem coisas completamente diferentes: um pergunta "está dentro?", o outro pergunta "vem depois?".

E `<` e `>` são **numéricos**: `"10" < "9"` é verdadeiro, porque compara 10 com 9. Para comparar texto alfabeticamente, use `]`.

### 2.6 Casamento de padrão: validar formato

O ObjectScript tem um operador embutido para verificar se um texto **tem o formato esperado**: o `?`.

```objectscript
if cpf ? 3N1"."3N1"."3N1"-"2N {
    write "formato válido", !
}
```

Leia da esquerda para a direita: *"3 dígitos, um ponto literal, 3 dígitos, um ponto, 3 dígitos, um hífen, 2 dígitos"*.

A analogia é a de um gabarito de papel com recortes: você encosta o gabarito no texto e vê se ele encaixa. Não interessa **o que** está escrito, interessa **o formato**.

Isso é imensamente útil para validar códigos, telefones, datas em texto, e para o problema do Capítulo 10: distinguir `"0"` (número) de `"abc"` (texto) sem depender de conversão numérica silenciosa.

---

## 3. As funções, uma a uma

### 3.1 `$LENGTH` — duas funções num nome só

```objectscript
write $LENGTH("Maria Silva"), !                // 11  -- caracteres
write $LENGTH("a;b;c", ";"), !                 // 3   -- pedaços
```

- **Com um argumento**: conta **caracteres**.
- **Com dois argumentos**: conta **pedaços** delimitados.

O segundo caso tem uma sutileza que cai na prova: **um texto sem o separador tem 1 pedaço**, e um texto **vazio** também tem... 1 pedaço? Não:

```objectscript
write $LENGTH("abc", ";"), !                   // 1
write $LENGTH("", ";"), !                      // 0
write $LENGTH(";", ";"), !                     // 2  -- dois pedaços vazios
write $LENGTH("a;", ";"), !                    // 2
```

Guarde: **o número de pedaços é o número de separadores mais um** — exceto no texto vazio, que tem zero.

### 3.2 `$EXTRACT` — recortar por posição

```objectscript
set t = "Maria Silva"

write $EXTRACT(t), !                // M      -- sem posição = primeiro
write $EXTRACT(t, 3), !             // r
write $EXTRACT(t, 1, 5), !          // Maria
write $EXTRACT(t, 7, *), !          // Silva
write $EXTRACT(t, *), !             // a      -- último
write $EXTRACT(t, *-4, *), !        // Silva  -- os cinco últimos
write $EXTRACT(t, 50), !            // ""     -- fora de faixa devolve vazio
```

Note: **`$EXTRACT` fora de faixa devolve vazio, não gera erro.** Isso o diferencia de `$LIST`, que gera erro. É uma inconsistência do produto que vale memorizar.

Atribuição no lugar:

```objectscript
set t = "Maria"
set $EXTRACT(t, 1) = "D"
write t, !                          // Daria

set $EXTRACT(t, 6, *) = " Silva"
write t, !                          // Daria Silva
```

### 3.3 `$PIECE` — trabalhar com campos

```objectscript
set t = "HGB;13.5;g/dL;final"

write $PIECE(t, ";", 1), !          // HGB
write $PIECE(t, ";", 3), !          // g/dL
write $PIECE(t, ";", 2, 3), !       // 13.5;g/dL  -- faixa, COM os separadores
write $PIECE(t, ";", 9), !          // ""  -- fora de faixa devolve vazio
```

Pontos importantes:

- **Com faixa (`de, ate`), o resultado inclui os separadores internos.** É um recorte do texto original, não uma lista.
- **Fora de faixa devolve vazio**, sem erro.
- Omitindo o número do pedaço, `$PIECE(t, ";")` devolve o **primeiro**.
- O separador pode ter **mais de um caractere**: `$PIECE(t, "->", 2)`.

Atribuição:

```objectscript
set $PIECE(t, ";", 2) = "14.2"
set $PIECE(t, ";", 6) = "extra"      // cria os separadores que faltam
```

E o padrão de laço sobre pedaços:

```objectscript
for i = 1:1:$LENGTH(t, ";") {
    write i, ": ", $PIECE(t, ";", i), !
}
```

### 3.4 `$FIND` e o operador `[`

```objectscript
set t = "Maria Silva Souza"

write t [ "Silva", !                // 1  -- contém?
write t [ "Pereira", !              // 0

write $FIND(t, "Silva"), !          // 12
write $FIND(t, "Pereira"), !        // 0
write $FIND(t, "S", 12), !          // 14  -- procura a partir da posição 12
```

**A pegadinha clássica:** `$FIND` devolve a posição **imediatamente após** o texto encontrado, não onde ele começa.

`"Silva"` começa na posição 7 e tem 5 caracteres; `$FIND` devolveu **12**, que é `7 + 5`.

Por que assim? Porque esse valor é exatamente onde você deve continuar procurando, o que torna o laço de busca repetida natural:

```objectscript
set pos = 0
for {
    set pos = $FIND(t, "a", pos)
    quit:pos=0
    write "achado terminando em ", pos, !
}
```

Para saber onde o texto **começa**, subtraia o comprimento:

```objectscript
set inicio = $FIND(t, "Silva") - $LENGTH("Silva")
```

Resumindo a escolha: **`[` quando você só quer saber se contém; `$FIND` quando precisa da posição.**

### 3.5 `$REPLACE` e `$TRANSLATE`

```objectscript
write $REPLACE("Rua das Flores", "Rua", "R."), !         // R. das Flores
write $REPLACE("a.b.c", ".", "-"), !                     // a-b-c

write $TRANSLATE("a.b,c;d", ".,;", "---"), !             // a-b-c-d
write $TRANSLATE("a.b,c;d", ".,;", ""), !                // abcd  -- remove
write $TRANSLATE("hello", "el", "ip"), !                 // hippo
```

Acompanhe a última letra a letra, porque ela mostra bem o mecanismo: `h` não está na lista de origem e passa intacto; `e` vira `i`; os dois `l` viram `p` cada um; `o` passa intacto. Resultado: `hippo`. **Cada caractere é decidido isoladamente** — a função nunca olha para a sequência `"el"` como uma unidade.

`$REPLACE` aceita argumentos adicionais para controlar a partir de onde começar, quantas substituições fazer e se a busca diferencia maiúsculas. Os argumentos exatos variam por versão: **verificar na documentação oficial**.

### 3.6 `$ZSTRIP` — limpeza

```objectscript
write "[", $ZSTRIP("  Maria  ", "<>W"), "]", !      // [Maria]
write "[", $ZSTRIP("  Maria  ", "<W"), "]", !       // [Maria  ]
write "[", $ZSTRIP("  Maria  ", ">W"), "]", !       // [  Maria]
write "[", $ZSTRIP("a1b2c3", "*N"), "]", !          // [abc]  -- remove dígitos
write "[", $ZSTRIP("Olá, mundo!", "*P"), "]", !     // [Olá mundo]  -- remove pontuação
```

A ação tem duas partes:

**Onde agir:**

| Código | Significado |
|---|---|
| `<` | só no **início** |
| `>` | só no **fim** |
| `<>` | nas **duas pontas** |
| `*` | em **todo** o texto |

**O que remover:**

| Código | Classe |
|---|---|
| `W` | espaços em branco (*whitespace*) |
| `P` | pontuação |
| `C` | caracteres de controle |
| `N` | dígitos |
| `A` | alfanuméricos |
| `E` | tudo (*everything*) |

E há um terceiro argumento para listar caracteres específicos a remover. As combinações completas: **verificar na documentação oficial**.

O uso mais comum, de longe, é **`$ZSTRIP(texto, "<>W")`** — limpar as pontas. Você já o viu em vários capítulos.

### 3.7 `$ZCONVERT` — maiúsculas, minúsculas e codificação

```objectscript
write $ZCONVERT("maria silva", "U"), !      // MARIA SILVA
write $ZCONVERT("MARIA SILVA", "L"), !      // maria silva
write $ZCONVERT("maria silva", "T"), !      // Maria Silva  -- título
write $ZCONVERT("maria silva", "W"), !      // Maria silva  -- primeira palavra
```

- **`"U"`** — tudo maiúsculo.
- **`"L"`** — tudo minúsculo.
- **`"T"`** — inicial maiúscula em cada palavra (*title case*).
- **`"W"`** — inicial maiúscula na primeira palavra.

Com **três argumentos**, `$ZCONVERT` faz conversão de **codificação de caracteres**:

```objectscript
set utf8 = $ZCONVERT(texto, "O", "UTF8")     // do formato interno para UTF-8
set nativo = $ZCONVERT(bytes, "I", "UTF8")   // de UTF-8 para o formato interno
```

`"O"` é *output* (saindo) e `"I"` é *input* (entrando). Isso é essencial ao ler ou gravar arquivos e ao trocar dados com sistemas externos. Os nomes de codificação suportados: **verificar na documentação oficial**.

### 3.8 `$JUSTIFY` e `$FNUMBER` — formatar

```objectscript
write "[", $JUSTIFY("abc", 10), "]", !              // [       abc]
write "[", $JUSTIFY(3.14159, 10, 2), "]", !         // [      3.14]
write "[", $JUSTIFY(7, 5, 2), "]", !                // [ 7.00]
```

- **`$JUSTIFY(valor, largura)`** — alinha à **direita**, preenchendo com espaços.
- **`$JUSTIFY(numero, largura, decimais)`** — **arredonda** para o número de casas e alinha.

Um truque que você já viu nesta apostila: combinando com `$TRANSLATE`, obtém-se preenchimento com zeros:

```objectscript
write $TRANSLATE($JUSTIFY(42, 6), " ", "0"), !      // 000042
```

Para alinhar à **esquerda**, não há função direta: concatene e recorte.

```objectscript
write "[", $EXTRACT("abc"_$JUSTIFY("", 10), 1, 10), "]", !   // [abc       ]
```

**`$FNUMBER`** formata números:

```objectscript
write $FNUMBER(1234567.891, "", 2), !        // 1234567.89
write $FNUMBER(1234567.891, ",", 2), !       // 1,234,567.89
write $FNUMBER(-42, "P"), !                  // (42)   -- negativo entre parênteses
write $FNUMBER(42, "+"), !                   // +42
```

Os códigos de formato mais usados são `","` (separador de milhar), `"+"` (força o sinal), `"-"` (suprime o sinal) e `"P"` (negativos entre parênteses). Podem ser combinados. A lista completa: **verificar na documentação oficial**.

**Atenção:** `$FNUMBER` produz o formato **norte-americano** por padrão — ponto decimal e vírgula de milhar. Para o formato brasileiro, é preciso trocar depois:

```objectscript
set n = $FNUMBER(1234567.891, ",", 2)        // 1,234,567.89
set br = $TRANSLATE(n, ".,", ",.")           // 1.234.567,89
```

O `$TRANSLATE` faz a troca cruzada num passo só: todo `.` vira `,` e todo `,` vira `.`. Este é um uso elegante e típico da função.

### 3.9 `$CHAR`, `$ASCII` e `$REVERSE`

```objectscript
write $CHAR(65), !                     // A
write $CHAR(72, 73), !                 // HI  -- vários códigos
write $CHAR(9), !                      // tabulação
write $CHAR(13, 10), !                 // quebra de linha do Windows

write $ASCII("A"), !                   // 65
write $ASCII("Maria", 3), !            // 114  -- código do 'r'
write $ASCII(""), !                    // -1   -- vazio devolve -1

write $REVERSE("Maria"), !             // airaM
```

`$CHAR` e `$ASCII` são as pontes entre texto e código numérico. Usos típicos: montar caracteres de controle, comparar códigos, detectar caracteres não imprimíveis.

`$REVERSE` é mais útil do que parece: ele permite fazer operações "a partir do fim" com funções que só olham para o começo.

### 3.10 O operador de padrão `?`

```
texto ? padrão
```

Devolve `1` se o texto **inteiro** casa com o padrão, `0` se não.

**Códigos de classe:**

| Código | Casa com |
|---|---|
| `A` | letras (alfabéticas) |
| `N` | dígitos numéricos |
| `U` | letras maiúsculas |
| `L` | letras minúsculas |
| `P` | pontuação |
| `C` | caracteres de controle |
| `E` | qualquer caractere |
| `"texto"` | o texto literal, entre aspas |

**Contadores, antes da classe:**

| Forma | Significado |
|---|---|
| `3N` | exatamente 3 dígitos |
| `.N` | zero ou mais dígitos |
| `1.N` | um ou mais dígitos |
| `2.5N` | de 2 a 5 dígitos |
| `.E` | qualquer coisa, inclusive nada |

**Exemplos:**

```objectscript
write "12345" ? 5N, !                          // 1
write "1234" ? 5N, !                           // 0
write "ABC-123" ? 3U1"-"3N, !                  // 1
write "abc" ? 1.A, !                           // 1
write "" ? .A, !                               // 1  -- zero ou mais
write "" ? 1.A, !                              // 0  -- um ou mais
write "2026-08-19" ? 4N1"-"2N1"-"2N, !         // 1
write "Maria123" ? 1.A.N, !                    // 1  -- letras depois dígitos
```

**Alternativas** são escritas entre parênteses, separadas por vírgula:

```objectscript
write "M" ? 1(1"M",1"F"), !                    // 1
write "X" ? 1(1"M",1"F"), !                    // 0
```

Duas armadilhas importantes:

1. **O padrão precisa cobrir o texto INTEIRO.** `"abc123" ? 1.A` é falso, porque sobram os dígitos. Para "começa com letras", use `"abc123" ? 1.A.E`.
2. **Vazio casa com padrões que permitem zero repetições.** `"" ? .N` é verdadeiro. Se vazio não for aceitável, use `1.N`.

### 3.11 Textos longos e Unicode

- Uma string tem um **limite máximo** na casa dos milhões de caracteres. Ultrapassá-lo causa **`<MAXSTRING>`**. Conteúdo grande ou indeterminado → **stream**, como visto no Capítulo 4.
- `$LENGTH` conta **caracteres**, não bytes. Um caractere acentuado conta como 1.
- Ao gravar em arquivo ou enviar para fora, a conversão de codificação com `$ZCONVERT` de três argumentos é o que garante que os acentos cheguem certos do outro lado.
- Comparações e ordenação de acentos dependem da configuração de localidade da instância: **verificar na documentação oficial**.

---

## 4. Exemplo comentado

Arquivo `src/LabStudy/Demo/Strings.cls`:

```objectscript
/// A tour of the string functions.
Class LabStudy.Demo.Strings Extends %RegisteredObject
{

/// Length, extraction and the star convention.
ClassMethod Extracting() As %Status
{
    set t = "Maria Silva Souza"

    write "text            : [", t, "]", !
    write "$LENGTH          : ", $LENGTH(t), !
    write "$LENGTH(t,' ')   : ", $LENGTH(t, " "), !

    write !, "-- $EXTRACT --", !
    write "  first          : ", $EXTRACT(t), !
    write "  1 to 5         : ", $EXTRACT(t, 1, 5), !
    write "  7 to end       : ", $EXTRACT(t, 7, *), !
    write "  last           : ", $EXTRACT(t, *), !
    write "  last 5         : ", $EXTRACT(t, *-4, *), !
    write "  out of range   : [", $EXTRACT(t, 99), "]", !

    write !, "-- in place --", !
    set copy = t
    set $EXTRACT(copy, 1) = "D"
    write "  after set      : ", copy, !

    write !, "-- $REVERSE --", !
    write "  reversed       : ", $REVERSE(t), !

    quit $$$OK
}

/// Fields and delimiters.
ClassMethod Pieces() As %Status
{
    set row = "HGB;13.5;g/dL;final"

    write "row              : ", row, !
    write "pieces           : ", $LENGTH(row, ";"), !

    for i = 1:1:$LENGTH(row, ";") {
        write "  ", i, ": [", $PIECE(row, ";", i), "]", !
    }

    write !, "  range 2 to 3   : [", $PIECE(row, ";", 2, 3), "]   <-- keeps separators", !
    write "  out of range   : [", $PIECE(row, ";", 9), "]", !

    write !, "-- in place --", !
    set copy = row
    set $PIECE(copy, ";", 2) = "14.2"
    write "  replaced       : ", copy, !

    set $PIECE(copy, ";", 6) = "extra"
    write "  beyond the end : ", copy, "   <-- separators created", !

    write !, "-- edge cases of $LENGTH with delimiter --", !
    write "  'abc'          : ", $LENGTH("abc", ";"), !
    write "  ''             : ", $LENGTH("", ";"), !
    write "  ';'            : ", $LENGTH(";", ";"), !
    write "  'a;'           : ", $LENGTH("a;", ";"), !

    quit $$$OK
}

/// Searching.
ClassMethod Searching() As %Status
{
    set t = "Maria Silva Souza"

    write "-- contains --", !
    write "  t [ 'Silva'    : ", t [ "Silva", !
    write "  t [ 'Pereira'  : ", t [ "Pereira", !

    write !, "-- $FIND returns the position AFTER the match --", !
    write "  $FIND(t,'Silva'): ", $FIND(t, "Silva"), !
    write "  where it starts : ", $FIND(t, "Silva") - $LENGTH("Silva"), !
    write "  not found       : ", $FIND(t, "Pereira"), !

    write !, "-- all occurrences of 'a' --", !
    set pos = 0
    for {
        set pos = $FIND(t, "a", pos)
        quit:pos=0
        write "    ends at ", pos, " (starts at ", pos - 1, ")", !
    }

    quit $$$OK
}

/// Replacing, translating and stripping.
ClassMethod Changing() As %Status
{
    write "-- $REPLACE works on substrings --", !
    write "  banana / an -> X : ", $REPLACE("banana", "an", "X"), !

    write !, "-- $TRANSLATE works character by character --", !
    write "  banana / an -> XY: ", $TRANSLATE("banana", "an", "XY"), !

    write !, "-- $TRANSLATE with a shorter target REMOVES --", !
    write "  123.456,78 clean : ", $TRANSLATE("123.456,78", ".,", ""), !
    write "  phone clean      : ", $TRANSLATE("(17) 99999-8888", "() -", ""), !

    write !, "-- $TRANSLATE for a crossed swap --", !
    write "  US to BR number  : ", $TRANSLATE("1,234,567.89", ".,", ",."), !

    write !, "-- $ZSTRIP --", !
    write "  trim both        : [", $ZSTRIP("   Maria   ", "<>W"), "]", !
    write "  trim left        : [", $ZSTRIP("   Maria   ", "<W"), "]", !
    write "  remove digits    : [", $ZSTRIP("a1b2c3", "*N"), "]", !
    write "  remove punct     : [", $ZSTRIP("Ola, mundo!", "*P"), "]", !

    write !, "-- $ZCONVERT --", !
    write "  upper            : ", $ZCONVERT("maria silva", "U"), !
    write "  lower            : ", $ZCONVERT("MARIA SILVA", "L"), !
    write "  title            : ", $ZCONVERT("maria silva", "T"), !

    quit $$$OK
}

/// Formatting for output.
ClassMethod Formatting() As %Status
{
    write "-- $JUSTIFY --", !
    write "  [", $JUSTIFY("abc", 10), "]", !
    write "  [", $JUSTIFY(3.14159, 10, 2), "]", !
    write "  [", $JUSTIFY(7, 8, 2), "]", !

    write !, "-- zero padding trick --", !
    write "  [", $TRANSLATE($JUSTIFY(42, 6), " ", "0"), "]", !

    write !, "-- $FNUMBER --", !
    write "  plain            : ", $FNUMBER(1234567.891, "", 2), !
    write "  thousands        : ", $FNUMBER(1234567.891, ",", 2), !
    write "  brazilian        : ", $TRANSLATE($FNUMBER(1234567.891, ",", 2), ".,", ",."), !
    write "  forced sign      : ", $FNUMBER(42, "+"), !
    write "  negative in ()   : ", $FNUMBER(-42, "P"), !

    write !, "-- a formatted table --", !
    write $JUSTIFY("code", 8), $JUSTIFY("value", 12), $JUSTIFY("unit", 8), !
    write $TRANSLATE($JUSTIFY("", 28), " ", "-"), !

    for row = "HGB:13.5:g/dL", "GLU:92:mg/dL", "CHOL:190.456:mg/dL" {
        write $JUSTIFY($PIECE(row, ":", 1), 8),
              $JUSTIFY($PIECE(row, ":", 2), 12, 2),
              $JUSTIFY($PIECE(row, ":", 3), 8), !
    }

    quit $$$OK
}

/// Character codes.
ClassMethod Codes() As %Status
{
    write "$CHAR(65)        : ", $CHAR(65), !
    write "$CHAR(72,73)     : ", $CHAR(72, 73), !
    write "$ASCII('A')      : ", $ASCII("A"), !
    write "$ASCII('Maria',3): ", $ASCII("Maria", 3), !
    write "$ASCII('')       : ", $ASCII(""), !

    write !, "-- detecting non printable characters --", !
    set dirty = "abc"_$CHAR(9)_"def"_$CHAR(1)_"ghi"

    write "  length         : ", $LENGTH(dirty), !
    for i = 1:1:$LENGTH(dirty) {
        set code = $ASCII(dirty, i)
        if code < 32 {
            write "    position ", i, " has control char code ", code, !
        }
    }
    write "  cleaned        : [", $ZSTRIP(dirty, "*C"), "]", !

    quit $$$OK
}

/// Pattern matching.
ClassMethod Patterns() As %Status
{
    write "-- basic --", !
    write "  '12345' ? 5N          : ", "12345" ? 5N, !
    write "  '1234'  ? 5N          : ", "1234" ? 5N, !
    write "  'abc'   ? 1.A         : ", "abc" ? 1.A, !
    write "  'abc123'? 1.A         : ", "abc123" ? 1.A, "   <-- pattern must cover it all", !
    write "  'abc123'? 1.A.N       : ", "abc123" ? 1.A.N, !

    write !, "-- empty string --", !
    write "  ''      ? .N          : ", "" ? .N, "   <-- zero or more matches empty", !
    write "  ''      ? 1.N         : ", "" ? 1.N, "   <-- one or more does not", !

    write !, "-- real formats --", !
    write "  '2026-08-19' as date  : ", "2026-08-19" ? 4N1"-"2N1"-"2N, !
    write "  'REG-000001' as record: ", "REG-000001" ? 3U1"-"6N, !
    write "  '(17) 99999-8888'     : ", "(17) 99999-8888" ? 1"("2N1") "5N1"-"4N, !

    write !, "-- alternatives --", !
    for v = "M", "F", "X" {
        write "  '", v, "' ? 1(1""M"",1""F"") : ", v ? 1(1"M",1"F"), !
    }

    write !, "-- telling numbers from text (chapter 10 revisited) --", !
    for v = "0", "007", "abc", "12.5", "-3", "" {
        write "  [", $JUSTIFY(v, 5), "] pure digits: ", (v ? 1.N),
              "   numeric-ish: ", (v ? .1"-".N.1"."(.N)), !
    }

    quit $$$OK
}

ClassMethod Demo() As %Status
{
    do ..Extracting()  write !
    do ..Pieces()      write !
    do ..Searching()   write !
    do ..Changing()    write !
    do ..Formatting()  write !
    do ..Codes()       write !
    do ..Patterns()
    quit $$$OK
}

}
```

### 4.1 Executando (trechos comentados)

```
LABSTUDY>DO ##class(LabStudy.Demo.Strings).Extracting()
text            : [Maria Silva Souza]
$LENGTH          : 17
$LENGTH(t,' ')   : 3

-- $EXTRACT --
  first          : M
  1 to 5         : Maria
  7 to end       : Silva Souza
  last           : a
  last 5         : Souza
  out of range   : []

-- in place --
  after set      : Daria Silva Souza

-- $REVERSE --
  reversed       : azuoS avliS airaM
```

- **`$LENGTH(t, " ")` devolveu 3**: três palavras separadas por dois espaços. Dois separadores, três pedaços.
- **`$EXTRACT(t, 99)` devolveu vazio, sem erro.** Compare com `$LIST`, que geraria erro. Duas funções análogas, comportamentos diferentes — memorize.
- **`$EXTRACT(t, *-4, *)` pegou os cinco últimos** sem que eu precisasse saber o comprimento.

```
LABSTUDY>DO ##class(LabStudy.Demo.Strings).Pieces()
...
  range 2 to 3   : [13.5;g/dL]   <-- keeps separators
  out of range   : []

-- in place --
  replaced       : HGB;14.2;g/dL;final
  beyond the end : HGB;14.2;g/dL;final;;extra   <-- separators created

-- edge cases of $LENGTH with delimiter --
  'abc'          : 1
  ''             : 0
  ';'            : 2
  'a;'           : 2
```

- **A faixa do `$PIECE` devolveu `"13.5;g/dL"`, com o separador no meio.** Ela recorta o texto original — não é uma lista, e não é um conjunto de campos.
- **Atribuir ao pedaço 6 criou os separadores faltantes**, deixando o pedaço 5 vazio. Comportamento muito útil para montar registros.
- **Os quatro casos de borda do `$LENGTH` com separador** merecem ser decorados. O texto vazio tem **zero** pedaços; qualquer outro tem separadores + 1.

```
LABSTUDY>DO ##class(LabStudy.Demo.Strings).Searching()
-- contains --
  t [ 'Silva'    : 1
  t [ 'Pereira'  : 0

-- $FIND returns the position AFTER the match --
  $FIND(t,'Silva'): 12
  where it starts : 7
  not found       : 0

-- all occurrences of 'a' --
    ends at 3 (starts at 2)
    ends at 6 (starts at 5)
    ends at 11 (starts at 10)
    ends at 18 (starts at 17)
```

- **`$FIND` devolveu 12 para algo que começa em 7.** Esta é a pegadinha mais cobrada do capítulo. O valor devolvido é onde **continuar** a busca.
- **O laço de todas as ocorrências funciona sem ajuste** justamente por causa disso: passar o resultado de volta como ponto de partida já pula o que foi achado.

```
LABSTUDY>DO ##class(LabStudy.Demo.Strings).Changing()
-- $REPLACE works on substrings --
  banana / an -> X : bXXa

-- $TRANSLATE works character by character --
  banana / an -> XY: bXYXYX

-- $TRANSLATE with a shorter target REMOVES --
  123.456,78 clean : 12345678
  phone clean      : 17999998888

-- $TRANSLATE for a crossed swap --
  US to BR number  : 1.234.567,89
...
```

- **`$REPLACE` achou duas vezes a sequência `"an"`** e trocou cada uma por um `X`, sobrando o `a` final: `bXXa`.
- **`$TRANSLATE` trocou cada `a` por `X` e cada `n` por `Y`**, independentemente: `bXYXYX`.
- **A limpeza do telefone em uma linha** é o uso prático mais comum do `$TRANSLATE`: remover um conjunto de caracteres de uma vez, sem laço.
- **A troca cruzada `".,"` → `",."`** converte número americano em brasileiro num passo. Elegante e impossível com `$REPLACE`, que faria duas passagens e se atrapalharia na segunda.

```
LABSTUDY>DO ##class(LabStudy.Demo.Strings).Patterns()
...
  'abc123'? 1.A         : 0   <-- pattern must cover it all
  'abc123'? 1.A.N       : 1

-- empty string --
  ''      ? .N          : 1   <-- zero or more matches empty
  ''      ? 1.N         : 0   <-- one or more does not
...
-- telling numbers from text (chapter 10 revisited) --
  [    0] pure digits: 1   numeric-ish: 1
  [  007] pure digits: 1   numeric-ish: 1
  [  abc] pure digits: 0   numeric-ish: 0
  [ 12.5] pure digits: 0   numeric-ish: 1
  [   -3] pure digits: 0   numeric-ish: 1
  [     ] pure digits: 0   numeric-ish: 1
```

- **O padrão precisa cobrir o texto inteiro.** `"abc123" ? 1.A` falhou porque sobraram dígitos.
- **Vazio casa com `.N` e não casa com `1.N`.** Se você usa padrão para validar campo obrigatório, use sempre `1.` (um ou mais).
- **A última tabela resolve o problema do Capítulo 10** de distinguir número de texto sem depender de conversão silenciosa. Note que o padrão "numeric-ish" aceita o vazio — se isso não for desejado, ajuste o padrão. É um bom exemplo de que **padrões precisam ser testados com os casos de borda**, não só com o caso feliz.

---

## 5. Variações e detalhes

### 5.1 `$PIECE`, `$LIST` ou `$EXTRACT`?

| Situação | Ferramenta |
|---|---|
| dado interno com vários campos | **`$LIST`** (Capítulo 14) |
| dado externo com separadores (CSV, protocolo) | **`$PIECE`** |
| posição fixa de caracteres (largura fixa, código com máscara) | **`$EXTRACT`** |
| buscar conteúdo | **`[`** ou **`$FIND`** |
| validar formato | **`?`** |

E a regra de arquitetura do capítulo anterior continua valendo: **`$PIECE` na fronteira, `$LIST` por dentro.**

### 5.2 Comparação de texto e maiúsculas

```objectscript
write ("Maria" = "maria"), !          // 0
write ("Maria" ] "maria"), !          // 0  -- 'M'(77) vem ANTES de 'm'(109)
write ("maria" ] "Maria"), !          // 1
```

Para comparar ignorando maiúsculas, normalize os dois lados:

```objectscript
if $ZCONVERT(a, "U") = $ZCONVERT(b, "U") { ... }
```

E lembre do Capítulo 13: essa mesma normalização é o que se usa como **chave** para ordenar sem diferenciar maiúsculas.

### 5.3 O operador `]]` e a ordenação padrão

```objectscript
write (10 ]] 2), !                    // 1  -- 10 vem depois de 2 numericamente
write ("a" ]] 10), !                  // 1  -- texto vem depois de número
write ("10" ] "2"), !                 // 0  -- como TEXTO, "10" vem antes de "2"
```

- **`]`** compara como **texto puro**, caractere a caractere.
- **`]]`** compara na **ordem padrão de subscritos** do Capítulo 13: números canônicos primeiro, em ordem numérica, depois texto.

Ou seja, `]]` responde à pergunta *"se estes dois fossem subscritos, qual viria depois?"*. É a ferramenta certa quando você precisa ordenar valores que podem ser números ou texto.

### 5.4 Construindo textos grandes com eficiência

Concatenar em laço tem custo, porque cada operação cria uma string nova:

```objectscript
// Aceitável para dezenas de itens
set out = ""
for i = 1:1:100 {
    set out = out_"linha "_i_$CHAR(13,10)
}
```

Para milhares de linhas, prefira um **stream** (Capítulo 4) ou acumule em array e junte no fim. E lembre: passar do limite de string causa `<MAXSTRING>`.

### 5.5 Máscaras e mascaramento

Duas tarefas parecidas com nomes parecidos:

**Aplicar máscara** — transformar `"12345678901"` em `"123.456.789-01"`:

```objectscript
ClassMethod ApplyMask(digits As %String, mask As %String) As %String
{
    set out = "", d = 1

    for i = 1:1:$LENGTH(mask) {
        set ch = $EXTRACT(mask, i)
        if ch = "#" {
            set out = out_$EXTRACT(digits, d)
            set d = d + 1
        } else {
            set out = out_ch
        }
    }
    quit out
}
```

**Mascarar dado sensível** — transformar `"12345678901"` em `"***.***.789-01"`, para exibir sem expor.

O segundo é um requisito de segurança (Capítulo 7) e será usado no projeto.

### 5.6 Textos vindos de arquivo

Ao ler linhas de um arquivo, três limpezas quase sempre são necessárias:

```objectscript
set line = $ZSTRIP(line, "<>W")                  // pontas
set line = $ZSTRIP(line, "*C")                   // caracteres de controle
set line = $ZCONVERT(line, "I", "UTF8")          // codificação, se aplicável
```

A ordem importa: converter a codificação **antes** de contar caracteres, senão `$LENGTH` conta bytes de uma sequência multibyte como se fossem caracteres separados.

---

## 6. Pegadinhas e erros comuns

**1) Achar que `$FIND` devolve onde o texto começa.**
Devolve a posição **imediatamente após**. Para o início, subtraia o comprimento.

**2) Confundir `$REPLACE` com `$TRANSLATE`.**
`$REPLACE` troca **sequências**; `$TRANSLATE` mapeia **caractere a caractere**.

**3) Esquecer que `$TRANSLATE` com destino mais curto REMOVE.**
Isso é recurso, não bug — e é a forma mais rápida de eliminar caracteres.

**4) Confundir `[` com `]`.**
`[` é "contém"; `]` é "vem depois na ordenação de texto".

**5) Usar `<` e `>` para comparar texto.**
Eles são **numéricos**. Para ordem alfabética, use `]`.

**6) Esperar que o padrão `?` case com parte do texto.**
Ele precisa cobrir o texto **inteiro**. Acrescente `.E` para permitir sobra.

**7) Usar `.N` num campo obrigatório.**
`.N` aceita vazio. Use `1.N` para exigir pelo menos um.

**8) Achar que `$LENGTH("", ";")` devolve 1.**
Devolve **0**. Texto vazio tem zero pedaços.

**9) Achar que `$EXTRACT` fora de faixa gera erro.**
Devolve vazio. É `$LIST` que gera erro — as duas se comportam diferente.

**10) Achar que `$PIECE(t, ";", 2, 3)` devolve dois valores.**
Devolve **um texto** com os dois pedaços e o separador entre eles.

**11) Usar `$FNUMBER` esperando formato brasileiro.**
Ele produz ponto decimal e vírgula de milhar. Converta com `$TRANSLATE(n, ".,", ",.")`.

**12) Comparar textos sem normalizar maiúsculas.**
`"Maria" = "maria"` é falso. Normalize os dois lados com `$ZCONVERT`.

**13) Contar caracteres para achar um campo.**
Use `$PIECE`. Contar posições quebra quando o tamanho do campo muda.

**14) Concatenar milhares de linhas numa string.**
Risco de `<MAXSTRING>` e custo alto. Use stream.

**15) Contar `$LENGTH` antes de converter a codificação.**
Uma sequência multibyte pode ser contada como vários caracteres.

**16) Usar `$ZSTRIP(t, "W")` sem indicar onde.**
Falta o `<`, `>` ou `*`. A ação tem duas partes: onde e o quê.

---

## 7. MÃO NA MASSA

---

### Exercício 15.1 — Recortando

**a) Enunciado:** Crie `LabStudy.Demo.Str1` com:

1. `ClassMethod Anatomy(texto)` — imprime comprimento, primeiro, último, primeiros 3, últimos 3, o do meio e o texto invertido.
2. `ClassMethod Initials(nomeCompleto)` — devolve as iniciais: `"Maria Silva Souza"` → `"M.S.S."`.
3. `ClassMethod Abbreviate(texto, limite)` — se o texto passar do limite, corta e acrescenta `"..."`, mantendo o total exatamente no limite.
4. `ClassMethod MaskTail(texto, visiveis)` — mostra só os últimos N caracteres, substituindo o resto por `*`: `"12345678901"` com 4 → `"*******8901"`.
5. `ClassMethod Center(texto, largura)` — centraliza o texto numa largura fixa.

**b) Dica:** Use `*` para o fim. No item 3, o corte precisa considerar o tamanho das reticências.

**c) Como testar:** `Abbreviate("Maria Silva Souza", 10)` deve devolver exatamente 10 caracteres.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Str1.cls`:

```objectscript
/// Character level string handling.
Class LabStudy.Demo.Str1 Extends %RegisteredObject
{

ClassMethod Anatomy(text As %String) As %Status
{
    write "text        : [", text, "]", !
    write "  length    : ", $LENGTH(text), !
    write "  first     : [", $EXTRACT(text), "]", !
    write "  last      : [", $EXTRACT(text, *), "]", !
    write "  first 3   : [", $EXTRACT(text, 1, 3), "]", !
    write "  last 3    : [", $EXTRACT(text, *-2, *), "]", !
    write "  middle    : [", $EXTRACT(text, ($LENGTH(text) + 1) \ 2), "]", !
    write "  reversed  : [", $REVERSE(text), "]", !
    quit $$$OK
}

/// "Maria Silva Souza" -> "M.S.S."
ClassMethod Initials(fullName As %String) As %String
{
    set fullName = $ZSTRIP(fullName, "<>W")
    quit:fullName="" ""

    set out = ""
    for i = 1:1:$LENGTH(fullName, " ") {
        set word = $PIECE(fullName, " ", i)
        continue:word=""
        set out = out_$ZCONVERT($EXTRACT(word), "U")_"."
    }
    quit out
}

/// Cuts to exactly "limit" characters, ending with "..." when it had to cut.
ClassMethod Abbreviate(text As %String, limit As %Integer = 20) As %String
{
    quit:$LENGTH(text)<=limit text
    quit:limit<=3 $EXTRACT(text, 1, limit)

    quit $EXTRACT(text, 1, limit - 3)_"..."
}

/// Shows only the last "visible" characters.
ClassMethod MaskTail(text As %String, visible As %Integer = 4) As %String
{
    set len = $LENGTH(text)
    quit:len<=visible text

    set stars = $TRANSLATE($JUSTIFY("", len - visible), " ", "*")
    quit stars_$EXTRACT(text, len - visible + 1, *)
}

/// Centres the text in a fixed width.
ClassMethod Center(text As %String, width As %Integer = 20) As %String
{
    set len = $LENGTH(text)
    quit:len>=width $EXTRACT(text, 1, width)

    set left = (width - len) \ 2
    set right = width - len - left

    quit $JUSTIFY("", left)_text_$JUSTIFY("", right)
}

ClassMethod Demo() As %Status
{
    do ..Anatomy("Maria Silva Souza")

    write !, "-- initials --", !
    for n = "Maria Silva Souza", "Bruno Lima", "  Ana  ", "" {
        write "  [", n, "] -> [", ..Initials(n), "]", !
    }

    write !, "-- abbreviate --", !
    for lim = 30, 17, 10, 5, 3, 2 {
        set r = ..Abbreviate("Maria Silva Souza", lim)
        write "  limit ", $JUSTIFY(lim, 2), " -> [", r, "]  (", $LENGTH(r), " chars)", !
    }

    write !, "-- mask tail --", !
    write "  [", ..MaskTail("12345678901", 4), "]", !
    write "  [", ..MaskTail("123", 4), "]", !

    write !, "-- center --", !
    write "  [", ..Center("titulo", 20), "]", !
    write "  [", ..Center("um titulo bem longo demais", 20), "]", !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Str1).Demo()
text        : [Maria Silva Souza]
  length    : 17
  first     : [M]
  last      : [a]
  first 3   : [Mar]
  last 3    : [uza]
  middle    : [S]
  reversed  : [azuoS avliS airaM]

-- initials --
  [Maria Silva Souza] -> [M.S.S.]
  [Bruno Lima] -> [B.L.]
  [  Ana  ] -> [A.]
  [] -> []

-- abbreviate --
  limit 30 -> [Maria Silva Souza]  (17 chars)
  limit 17 -> [Maria Silva Souza]  (17 chars)
  limit 10 -> [Maria S...]  (10 chars)
  limit  5 -> [Ma...]  (5 chars)
  limit  3 -> [Mar]  (3 chars)
  limit  2 -> [Ma]  (2 chars)

-- mask tail --
  [*******8901]
  [123]

-- center --
  [       titulo       ]
  [um titulo bem longo ]
```

**Por que cada decisão:**

- **`$EXTRACT(text, *-2, *)` pega os três últimos** sem calcular nada. Se eu tivesse escrito `$EXTRACT(text, $LENGTH(text)-2, $LENGTH(text))`, funcionaria — mas com duas chamadas a mais e uma chance a mais de errar o `-2`.
- **`Initials` usa `$LENGTH(nome, " ")` para contar palavras** e `$PIECE` para pegar cada uma. Note o `continue:word=""`, que trata espaços duplos — `"Maria  Silva"` produziria um pedaço vazio no meio.
- **`Initials` limpa as pontas primeiro.** Sem isso, `"  Ana  "` produziria pedaços vazios no começo e no fim, e a saída teria pontos sobrando.
- **`Abbreviate` trata o caso `limit <= 3`.** Sem ele, `Abbreviate(t, 2)` calcularia `$EXTRACT(t, 1, -1)`, devolvendo vazio, e depois acrescentaria `"..."` — resultando em 3 caracteres onde o limite era 2. **A borda quebra a promessa da função**, e a função foi escrita para nunca ultrapassar o limite.
- **`MaskTail` usa o truque `$TRANSLATE($JUSTIFY("", n), " ", "*")`** para gerar N asteriscos. É o mesmo truque do preenchimento com zeros, com outro caractere.
- **`MaskTail` devolve o texto inteiro quando ele é curto demais.** Mascarar `"123"` deixando 4 visíveis não faz sentido; devolver o original é mais honesto do que devolver algo estranho. Numa aplicação de segurança real, porém, isso seria discutível — talvez fosse melhor mascarar tudo. **Decisões de mascaramento pertencem ao negócio, não ao utilitário.**
- **`Center` distribui o espaço extra à direita** quando a diferença é ímpar, e trunca quando o texto não cabe. As duas decisões precisam ser tomadas; deixá-las implícitas produziria larguras inconsistentes na tabela.

---

### Exercício 15.2 — Campos e busca

**a) Enunciado:** Crie `LabStudy.Demo.Str2` para trabalhar com uma linha de protocolo no formato `"campo1|campo2|campo3|..."`:

1. `ClassMethod ShowFields(linha, sep)` — lista todos os campos numerados.
2. `ClassMethod GetField(linha, sep, n, padrao)` — devolve o campo n, ou o padrão se estiver vazio ou não existir.
3. `ClassMethod SetField(linha, sep, n, valor)` — devolve a linha com o campo n alterado, criando separadores se necessário.
4. `ClassMethod CountOccurrences(texto, procurado)` — conta quantas vezes um texto aparece.
5. `ClassMethod PositionsOf(texto, procurado) As %List` — devolve as posições **iniciais** de todas as ocorrências.
6. `ClassMethod ReplaceNth(texto, procurado, novo, n)` — substitui apenas a n-ésima ocorrência.

**b) Dica:** Nos itens 4 a 6, use `$FIND` em laço, lembrando que ele devolve a posição **após** o achado.

**c) Como testar:** `PositionsOf("banana", "an")` deve devolver as posições 2 e 4.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Str2.cls`:

```objectscript
/// Field and search handling.
Class LabStudy.Demo.Str2 Extends %RegisteredObject
{

ClassMethod ShowFields(line As %String, sep As %String = "|") As %Integer
{
    set n = $LENGTH(line, sep)
    write "line   : [", line, "]", !
    write "fields : ", n, !

    for i = 1:1:n {
        write "  ", $JUSTIFY(i, 2), ": [", $PIECE(line, sep, i), "]", !
    }
    quit n
}

/// Field n, or the default when missing or empty.
ClassMethod GetField(line As %String, sep As %String, n As %Integer, default As %String = "") As %String
{
    set v = $PIECE(line, sep, n)
    quit $SELECT(v = "": default, 1: v)
}

/// Returns the line with field n replaced.
ClassMethod SetField(line As %String, sep As %String, n As %Integer, value As %String) As %String
{
    set $PIECE(line, sep, n) = value
    quit line
}

/// How many times "needle" appears in "text".
ClassMethod CountOccurrences(text As %String, needle As %String) As %Integer
{
    quit:needle="" 0

    set count = 0, pos = 0
    for {
        set pos = $FIND(text, needle, pos)
        quit:pos=0
        set count = count + 1
    }
    quit count
}

/// Starting positions of every occurrence.
ClassMethod PositionsOf(text As %String, needle As %String) As %List
{
    quit:needle="" ""

    set result = "", pos = 0, len = $LENGTH(needle)
    for {
        set pos = $FIND(text, needle, pos)
        quit:pos=0
        set result = result_$LISTBUILD(pos - len)
    }
    quit result
}

/// Replaces only the nth occurrence.
ClassMethod ReplaceNth(text As %String, needle As %String, new As %String, n As %Integer = 1) As %String
{
    set positions = ..PositionsOf(text, needle)
    quit:$LISTLENGTH(positions)<n text

    set start = $LIST(positions, n)
    set len = $LENGTH(needle)

    quit $EXTRACT(text, 1, start - 1)_new_$EXTRACT(text, start + len, *)
}

ClassMethod Demo() As %Status
{
    set line = "HGB|13.5||g/dL|final"

    do ..ShowFields(line)

    write !, "-- GetField with default --", !
    write "  field 3         : [", ..GetField(line, "|", 3, "(vazio)"), "]", !
    write "  field 9         : [", ..GetField(line, "|", 9, "(ausente)"), "]", !

    write !, "-- SetField --", !
    write "  set 3           : [", ..SetField(line, "|", 3, "OK"), "]", !
    write "  set 8           : [", ..SetField(line, "|", 8, "novo"), "]", !

    write !, "-- searching in 'banana' --", !
    write "  occurrences of 'an' : ", ..CountOccurrences("banana", "an"), !

    set p = ..PositionsOf("banana", "an")
    write "  positions           : ", $LISTTOSTRING(p, ", "), !

    write !, "-- replace nth --", !
    for n = 1, 2, 3 {
        write "  replace #", n, " : [", ..ReplaceNth("banana", "an", "<X>", n), "]", !
    }

    write !, "-- a real case: fixing one date in a text --", !
    set t = "coleta 2026-01-01, resultado 2026-01-01, laudo 2026-01-01"
    write "  before : ", t, !
    write "  after  : ", ..ReplaceNth(t, "2026-01-01", "2026-01-05", 2), !

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Str2).Demo()
line   : [HGB|13.5||g/dL|final]
fields : 5
   1: [HGB]
   2: [13.5]
   3: []
   4: [g/dL]
   5: [final]

-- GetField with default --
  field 3         : [(vazio)]
  field 9         : [(ausente)]

-- SetField --
  set 3           : [HGB|13.5|OK|g/dL|final]
  set 8           : [HGB|13.5||g/dL|final|||novo]

-- searching in 'banana' --
  occurrences of 'an' : 2
  positions           : 2, 4

-- replace nth --
  replace #1 : [b<X>ana]
  replace #2 : [ban<X>a]
  replace #3 : [banana]

-- a real case: fixing one date in a text --
  before : coleta 2026-01-01, resultado 2026-01-01, laudo 2026-01-01
  after  : coleta 2026-01-01, resultado 2026-01-05, laudo 2026-01-01
```

**Por que cada resultado:**

- **O campo 3 vazio e o campo 9 inexistente foram tratados igualmente pelo `GetField`.** Isso é uma escolha: `$PIECE` devolve vazio nos dois casos e não os distingue. Se a distinção importasse — e no Capítulo 10 você viu quando ela importa — seria preciso comparar `n` com `$LENGTH(line, sep)` antes. **`$PIECE` não distingue "vazio" de "inexistente".** Este é um limite real da ferramenta.
- **`SetField(line, "|", 8, ...)` criou três separadores** para chegar à posição 8. Comportamento documentado e útil.
- **As ocorrências de `"an"` em `"banana"` foram 2, não 3.** Depois de achar `"an"` na posição 2, a busca continua da posição 4 — pulando a sobreposição. Se você esperava 3, estava contando ocorrências **sobrepostas**, que exigiriam avançar apenas uma posição por vez. **A definição de "quantas vezes aparece" não é única**, e o código precisa escolher.
- **`ReplaceNth` com n=3 devolveu o texto original**, porque só existem 2 ocorrências. Devolver a entrada inalterada quando a operação não se aplica é a decisão mais previsível para um utilitário.
- **O caso da data é o motivo de `ReplaceNth` existir.** `$REPLACE` trocaria as três datas de uma vez; aqui era preciso trocar apenas a segunda. É uma necessidade real em correção de laudos e mensagens.
- **`PositionsOf` devolve uma lista**, não um texto separado — a lição do Capítulo 14 aplicada.

---

### Exercício 15.3 — Limpeza, conversão e formatação

**a) Enunciado:** Crie `LabStudy.Demo.Str3`, um conjunto de normalizadores realistas:

1. `ClassMethod OnlyDigits(texto)` — mantém apenas dígitos.
2. `ClassMethod NormalizeSpaces(texto)` — remove pontas e reduz espaços múltiplos internos a um só.
3. `ClassMethod ProperName(texto)` — normaliza um nome: espaços, capitalização de cada palavra, mas mantendo em minúsculas as preposições `de`, `da`, `do`, `dos`, `das` e `e`.
4. `ClassMethod FormatBrazilian(numero, decimais)` — formata no padrão brasileiro.
5. `ClassMethod ApplyMask(digitos, mascara)` — aplica uma máscara com `#`.
6. `ClassMethod Slug(texto)` — transforma um título em identificador: minúsculas, sem pontuação, espaços viram hífen.

**b) Dica:** No item 2, reduzir espaços múltiplos exige um laço com `$REPLACE` até não haver mais espaços duplos.

**c) Como testar:** `ProperName("  MARIA   DA silva SOUZA ")` deve devolver `"Maria da Silva Souza"`.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Str3.cls`:

```objectscript
/// Realistic text normalisers.
Class LabStudy.Demo.Str3 Extends %RegisteredObject
{

/// Words that stay lowercase inside a name.
Parameter LOWERWORDS = "de,da,do,dos,das,e";

/// Keeps digits only.
ClassMethod OnlyDigits(text As %String) As %String [ CodeMode = expression ]
{
$ZSTRIP(text, "*E", , "0123456789")
}

/// Trims and collapses inner runs of spaces.
ClassMethod NormalizeSpaces(text As %String) As %String
{
    set text = $ZSTRIP($GET(text), "<>W")

    while text [ "  " {
        set text = $REPLACE(text, "  ", " ")
    }
    quit text
}

/// Proper capitalisation of a person name.
ClassMethod ProperName(text As %String) As %String
{
    set text = ..NormalizeSpaces(text)
    quit:text="" ""

    set out = ""
    set words = $LENGTH(text, " ")

    for i = 1:1:words {
        set word = $ZCONVERT($PIECE(text, " ", i), "L")
        continue:word=""

        // small words stay lowercase, unless they are the first word
        if (i > 1) && ($LISTFIND($LISTFROMSTRING(..#LOWERWORDS, ","), word) > 0) {
            set piece = word
        } else {
            set piece = $ZCONVERT($EXTRACT(word), "U")_$EXTRACT(word, 2, *)
        }

        set out = out_$SELECT(out = "": "", 1: " ")_piece
    }
    quit out
}

/// Brazilian number format: 1.234.567,89
ClassMethod FormatBrazilian(number As %Numeric, decimals As %Integer = 2) As %String
{
    quit:number="" ""
    quit $TRANSLATE($FNUMBER(number, ",", decimals), ".,", ",.")
}

/// Applies a mask where # is replaced by the next digit.
ClassMethod ApplyMask(digits As %String, mask As %String) As %String
{
    set digits = ..OnlyDigits(digits)
    set out = "", d = 1

    for i = 1:1:$LENGTH(mask) {
        set ch = $EXTRACT(mask, i)

        if ch = "#" {
            quit:d>$LENGTH(digits)
            set out = out_$EXTRACT(digits, d)
            set d = d + 1
        } else {
            set out = out_ch
        }
    }
    quit out
}

/// Turns a title into an identifier.
ClassMethod Slug(text As %String) As %String
{
    set text = $ZCONVERT(..NormalizeSpaces(text), "L")
    set text = $ZSTRIP(text, "*P")           // drop punctuation
    set text = ..NormalizeSpaces(text)       // punctuation removal may leave gaps
    quit $TRANSLATE(text, " ", "-")
}

ClassMethod Demo() As %Status
{
    write "-- only digits --", !
    for t = "(17) 99999-8888", "123.456.789-01", "abc123def" {
        write "  [", $JUSTIFY(t, 18), "] -> [", ..OnlyDigits(t), "]", !
    }

    write !, "-- normalize spaces --", !
    write "  [", ..NormalizeSpaces("  Maria    Silva   Souza  "), "]", !

    write !, "-- proper name --", !
    for n = "  MARIA   DA silva SOUZA ", "joao dos santos e silva", "ANA", "de paula" {
        write "  [", $JUSTIFY(n, 26), "] -> [", ..ProperName(n), "]", !
    }

    write !, "-- brazilian numbers --", !
    for v = 1234567.891, 0.5, -42, 1000 {
        write "  ", $JUSTIFY(v, 14), " -> ", ..FormatBrazilian(v), !
    }

    write !, "-- masks --", !
    write "  [", ..ApplyMask("12345678901", "###.###.###-##"), "]", !
    write "  [", ..ApplyMask("1799999888", "(##) #####-####"), "]", !
    write "  [", ..ApplyMask("123", "###.###.###-##"), "]", !

    write !, "-- slug --", !
    for t = "Relatorio Mensal de Exames!", "  HGB, GLU e CHOL  " {
        write "  [", $JUSTIFY(t, 28), "] -> [", ..Slug(t), "]", !
    }

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Str3).Demo()
-- only digits --
  [   (17) 99999-8888] -> [17999998888]
  [    123.456.789-01] -> [12345678901]
  [         abc123def] -> [123]

-- normalize spaces --
  [Maria Silva Souza]

-- proper name --
  [   MARIA   DA silva SOUZA ] -> [Maria da Silva Souza]
  [   joao dos santos e silva] -> [Joao dos Santos e Silva]
  [                       ANA] -> [Ana]
  [                  de paula] -> [De Paula]

-- brazilian numbers --
     1234567.891 -> 1.234.567,89
             0.5 -> 0,50
             -42 -> -42,00
            1000 -> 1.000,00

-- masks --
  [123.456.789-01]
  [(17) 99999-8888]
  [123.]

-- slug --
  [ Relatorio Mensal de Exames!] -> [relatorio-mensal-de-exames]
  [         HGB, GLU e CHOL    ] -> [hgb-glu-e-chol]
```

**Por que cada decisão:**

- **`OnlyDigits` usa `$ZSTRIP` com o argumento de caracteres a preservar** — removendo tudo (`*E`) **exceto** os dígitos. A alternativa com `$TRANSLATE` exigiria listar todos os caracteres possíveis a remover, o que é impraticável.
- **Repare na vírgula dupla:** `$ZSTRIP(text, "*E", , "0123456789")`. O argumento vazio no meio é o de caracteres a **remover**, que aqui não é usado; a lista de dígitos vai na posição seguinte, a de caracteres a **preservar**. Trocar as duas posições produz um método que devolve sempre vazio, sem erro nenhum — um erro silencioso clássico. **Confirme a ordem na sua versão antes de confiar:**

```
LABSTUDY>WRITE "[", $ZSTRIP("(17) 99999-8888", "*E", , "0123456789"), "]", !
[17999998888]
```

  Se a saída vier vazia ou inalterada, a assinatura da sua versão é diferente: **verificar na documentação oficial** ou usar `DO ##class(%SYSTEM.Util).Help()` para conferir.
- **`NormalizeSpaces` usa um `while` com `$REPLACE`.** Uma passagem só não basta: `"a    b"` com quatro espaços vira `"a  b"` numa passagem e `"a b"` na segunda. Este é um caso em que a solução ingênua de uma passagem **parece** funcionar nos testes com dois espaços e falha com quatro.
- **`ProperName` mantém as preposições em minúsculas, exceto na primeira posição.** Veja `"de paula"` virando `"De Paula"`: a primeira palavra é capitalizada mesmo sendo preposição, porque um nome não começa com minúscula. Esse é o tipo de regra que só aparece quando se testa com dados reais.
- **`ProperName` usa `$LISTFIND` sobre uma lista construída de um parâmetro.** A lista de palavras fica num `Parameter`, num lugar só — se amanhã incluírem `"del"` ou `"van"`, muda-se uma linha.
- **`FormatBrazilian` é a troca cruzada num `$TRANSLATE`.** Repare que `-42` virou `-42,00` corretamente: o sinal não é afetado, porque não é nem `.` nem `,`.
- **`ApplyMask` para quando os dígitos acabam**, produzindo `"123."` para uma entrada curta. Isso é honesto — mostra que faltam dígitos — mas em produção você provavelmente validaria o comprimento antes e recusaria. **Utilitário que aceita entrada inválida e produz saída parcial precisa de um validador na frente.**
- **`Slug` chama `NormalizeSpaces` duas vezes**, e isso é necessário: remover a pontuação de `"HGB, GLU"` deixa `"HGB  GLU"` com dois espaços, que só a segunda passagem colapsa. A ordem das operações de limpeza importa, e testar com dados reais revela essas interações.

---

### Exercício 15.4 — Casamento de padrão

**a) Enunciado:** Crie `LabStudy.Demo.Str4` com validadores baseados no operador `?`:

1. `ClassMethod IsInteger(v)` — inteiro positivo, sem sinal.
2. `ClassMethod IsNumber(v)` — número com sinal opcional e parte decimal opcional.
3. `ClassMethod IsIsoDate(v)` — formato `AAAA-MM-DD`, **e** com valores plausíveis (mês de 01 a 12).
4. `ClassMethod IsRecordNumber(v)` — o formato do projeto: `REG-` seguido de 6 dígitos.
5. `ClassMethod IsEmail(v)` — validação simples e honesta sobre suas limitações.
6. `ClassMethod Classify(v)` — devolve uma etiqueta descrevendo o que o valor parece ser.
7. `ClassMethod TestAll()` — roda todos os validadores contra uma bateria de valores, incluindo bordas.

**b) Dica:** No item 3, o padrão sozinho não valida o mês; combine padrão com verificação de faixa.

**c) Como testar:** `IsIsoDate("2026-13-01")` deve ser falso, apesar de o formato estar certo.

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Demo/Str4.cls`:

```objectscript
/// Format validation with the pattern match operator.
Class LabStudy.Demo.Str4 Extends %RegisteredObject
{

/// One or more digits, nothing else.
ClassMethod IsInteger(v As %String) As %Boolean [ CodeMode = expression ]
{
v ? 1.N
}

/// Optional sign, digits, optional decimal part.
ClassMethod IsNumber(v As %String) As %Boolean [ CodeMode = expression ]
{
v ? .1"-" 1.N .1(1"." 1.N)
}

/// ISO date, format AND plausible values.
ClassMethod IsIsoDate(v As %String) As %Boolean
{
    quit:'(v ? 4N1"-"2N1"-"2N) 0

    set month = +$PIECE(v, "-", 2)
    set day = +$PIECE(v, "-", 3)

    quit:(month < 1) || (month > 12) 0
    quit:(day < 1) || (day > 31) 0

    quit 1
}

/// REG- followed by exactly 6 digits.
ClassMethod IsRecordNumber(v As %String) As %Boolean [ CodeMode = expression ]
{
v ? 1"REG-" 6N
}

/// Deliberately simple. Real e-mail validation is far more complex;
/// this only rejects the obviously wrong.
ClassMethod IsEmail(v As %String) As %Boolean
{
    quit:'(v ? 1.E 1"@" 1.E 1"." 1.E) 0
    quit:v [ " " 0
    quit:$LENGTH(v, "@") '= 2 0
    quit:$PIECE(v, "@", 1) = "" 0
    quit:$PIECE(v, "@", 2) = "" 0
    quit 1
}

/// Describes what the value looks like.
ClassMethod Classify(v As %String) As %String
{
    quit:v="" "vazio"
    quit:..IsIsoDate(v) "data ISO"
    quit:..IsRecordNumber(v) "numero de registro"
    quit:..IsEmail(v) "email"
    quit:..IsInteger(v) "inteiro"
    quit:..IsNumber(v) "numero"
    quit:v?1.A "somente letras"
    quit:v?1.AN "alfanumerico"
    quit "texto"
}

ClassMethod TestAll() As %Status
{
    set values = $LISTBUILD("123", "007", "-3", "12.5", "12.", ".5", "abc",
                            "abc123", "2026-08-19", "2026-13-01", "2026-8-19",
                            "REG-000001", "REG-1", "a@b.com", "a b@c.com",
                            "@x.com", "", "0")

    write $JUSTIFY("valor", 14), $JUSTIFY("int", 5), $JUSTIFY("num", 5),
          $JUSTIFY("data", 6), $JUSTIFY("reg", 5), $JUSTIFY("mail", 6),
          "   classificacao", !
    write $TRANSLATE($JUSTIFY("", 70), " ", "-"), !

    set ptr = 0
    while $LISTNEXT(values, ptr, v) {
        set v = $GET(v)
        write $JUSTIFY("["_v_"]", 14),
              $JUSTIFY(..IsInteger(v), 5),
              $JUSTIFY(..IsNumber(v), 5),
              $JUSTIFY(..IsIsoDate(v), 6),
              $JUSTIFY(..IsRecordNumber(v), 5),
              $JUSTIFY(..IsEmail(v), 6),
              "   ", ..Classify(v), !
    }

    quit $$$OK
}

}
```

```
LABSTUDY>DO ##class(LabStudy.Demo.Str4).TestAll()
         valor  int  num  data  reg  mail   classificacao
----------------------------------------------------------------------
         [123]    1    1     0    0     0   inteiro
         [007]    1    1     0    0     0   inteiro
          [-3]    0    1     0    0     0   numero
        [12.5]    0    1     0    0     0   numero
         [12.]    0    0     0    0     0   texto
          [.5]    0    0     0    0     0   texto
         [abc]    0    0     0    0     0   somente letras
      [abc123]    0    0     0    0     0   alfanumerico
  [2026-08-19]    0    0     1    0     0   data ISO
  [2026-13-01]    0    0     0    0     0   texto
   [2026-8-19]    0    0     0    0     0   texto
  [REG-000001]    0    0     0    1     0   numero de registro
       [REG-1]    0    0     0    0     0   texto
     [a@b.com]    0    0     0    0     1   email
   [a b@c.com]    0    0     0    0     0   texto
      [@x.com]    0    0     0    0     0   texto
            []    0    0     0    0     0   vazio
           [0]    1    1     0    0     0   inteiro
```

**Por que cada resultado:**

- **`"007"` foi classificado como inteiro.** Correto: o padrão valida **formato**, não canonicidade. Se você precisa distinguir número canônico (Capítulo 13), essa é outra verificação.
- **`"12."` e `".5"` foram rejeitados pelo `IsNumber`.** O padrão exige dígitos antes do ponto e depois dele. Isso é uma escolha: alguns sistemas aceitam `.5`. O padrão **documenta a decisão** de forma legível, o que é uma vantagem sobre uma cadeia de `if`.
- **`"2026-13-01"` passou no formato e falhou na faixa.** É a demonstração de que **padrão valida forma, não conteúdo**. Um validador de data honesto precisa dos dois — e, para validar o dia corretamente considerando o mês e anos bissextos, precisaria de mais ainda (Capítulo 16).
- **`"2026-8-19"` falhou** porque o padrão exige exatamente 2 dígitos no mês. Formatos ISO são rígidos de propósito.
- **`"REG-1"` falhou** porque o padrão exige 6 dígitos. Isso protege contra números de registro mal formados entrando no sistema.
- **`IsEmail` combina padrão com quatro verificações extras**, e o comentário admite explicitamente que é uma validação simples. Validação completa de e-mail é notoriamente difícil, e um método que **finge** validar completamente é pior do que um que declara o próprio limite. Note que `"a b@c.com"` foi rejeitado pelo teste de espaço, não pelo padrão.
- **`Classify` testa do mais específico para o mais genérico.** Se testasse `IsInteger` antes de `IsIsoDate`, nada mudaria aqui — mas se testasse "texto" primeiro, tudo cairia lá. **Ordem de especificidade é a regra em cadeias de classificação.**
- **`"0"` foi classificado como inteiro**, e não como vazio. Este é o Capítulo 10 outra vez: zero é um valor.

---

### Exercício 15.5 — PROJETO CONTÍNUO: texto no laboratório

**a) Enunciado:** Crie a camada de texto do sistema:

1. `LabStudy.Text` com:
   - `Clean(texto)` — limpeza padrão de entrada: pontas, controles, espaços múltiplos;
   - `ProperName(nome)`, `OnlyDigits(texto)`, `Slug(texto)` (reaproveite o exercício 15.3);
   - `Number(valor, decimais)` — formatação brasileira;
   - `Mask(digitos, mascara)` e `MaskSensitive(texto, visiveis)`;
   - `Abbreviate(texto, limite)` e `Pad(texto, largura, alinhamento)`;
   - `IsRecordNumber(v)`, `IsIsoDate(v)`, `IsEmail(v)`.
2. Em `LabStudy.Patient`:
   - use `LabStudy.Text.Clean` e `ProperName` no `%OnBeforeSave`;
   - valide `RecordNumber` e `Email` no `%OnValidateObject`, usando os padrões;
   - acrescente `Method DisplayName()` e `Method MaskedEmail()`.
3. Crie `LabStudy.Formatter` com:
   - `ClassMethod Table(colunas, larguras, alinhamentos)` — imprime cabeçalho e separador;
   - `ClassMethod Row(valores, larguras, alinhamentos)` — imprime uma linha alinhada;
   - `ClassMethod Line(largura, caractere)`.
4. Reescreva `LabStudy.Reports.Dashboard()` usando o `Formatter`.
5. Suba `LabStudy.App` para `"1.6"`.

**b) Dica:** No item 3, receba as colunas, larguras e alinhamentos como **listas** — a lição do Capítulo 14.

**c) Como testar:**

```
LABSTUDY>SET p = ##class(LabStudy.Patient).%New()
LABSTUDY>SET p.Name = "  MARIA   DA silva  ", p.RecordNumber = "reg-000099"
LABSTUDY>SET p.BirthDate = $ZDATEH("1990-05-17",3), p.Email = "maria@exemplo.com"
LABSTUDY>WRITE $$$ISOK(p.%Save()), " -> [", p.Name, "] [", p.RecordNumber, "]", !
LABSTUDY>WRITE p.MaskedEmail(), !
LABSTUDY>DO ##class(LabStudy.Reports).Dashboard()
```

```
=== SOLUÇÃO (só olhe depois de tentar) ===
```

`src/LabStudy/Text.cls`:

```objectscript
/// Text handling for the LabStudy system.
/// Everything that touches strings goes through here, so the rules live in one place.
Class LabStudy.Text Extends %RegisteredObject
{

/// Words that stay lowercase inside a person name.
Parameter LOWERWORDS = "de,da,do,dos,das,e";

/// Standard input cleaning: control chars, outer spaces, inner runs.
ClassMethod Clean(text As %String = "") As %String
{
    set text = $ZSTRIP($GET(text), "*C")
    set text = $ZSTRIP(text, "<>W")

    while text [ "  " {
        set text = $REPLACE(text, "  ", " ")
    }
    quit text
}

/// Digits only.
ClassMethod OnlyDigits(text As %String = "") As %String [ CodeMode = expression ]
{
$ZSTRIP($GET(text), "*E", , "0123456789")
}

/// Person name with proper capitalisation.
ClassMethod ProperName(text As %String = "") As %String
{
    set text = ..Clean(text)
    quit:text="" ""

    set small = $LISTFROMSTRING(..#LOWERWORDS, ",")
    set out = ""

    for i = 1:1:$LENGTH(text, " ") {
        set word = $ZCONVERT($PIECE(text, " ", i), "L")
        continue:word=""

        if (i > 1) && ($LISTFIND(small, word) > 0) {
            set piece = word
        } else {
            set piece = $ZCONVERT($EXTRACT(word), "U")_$EXTRACT(word, 2, *)
        }

        set out = out_$SELECT(out = "": "", 1: " ")_piece
    }
    quit out
}

/// Brazilian number format.
ClassMethod Number(value As %Numeric = "", decimals As %Integer = 2) As %String
{
    quit:value="" ""
    quit $TRANSLATE($FNUMBER(value, ",", decimals), ".,", ",.")
}

/// Applies a mask, # standing for the next digit.
ClassMethod Mask(digits As %String, mask As %String) As %String
{
    set digits = ..OnlyDigits(digits)
    set out = "", d = 1

    for i = 1:1:$LENGTH(mask) {
        set ch = $EXTRACT(mask, i)
        if ch = "#" {
            quit:d>$LENGTH(digits)
            set out = out_$EXTRACT(digits, d)
            set d = d + 1
        } else {
            set out = out_ch
        }
    }
    quit out
}

/// Hides all but the last characters. For display of sensitive data.
ClassMethod MaskSensitive(text As %String = "", visible As %Integer = 4) As %String
{
    set len = $LENGTH(text)
    quit:len=0 ""
    quit:len<=visible $TRANSLATE($JUSTIFY("", len), " ", "*")

    quit $TRANSLATE($JUSTIFY("", len - visible), " ", "*")_$EXTRACT(text, len - visible + 1, *)
}

/// Hides the local part of an e-mail: ma***@exemplo.com
ClassMethod MaskEmail(email As %String = "") As %String
{
    quit:email="" ""
    quit:'(email [ "@") ..MaskSensitive(email, 0)

    set local = $PIECE(email, "@", 1)
    set domain = $PIECE(email, "@", 2)

    set keep = $SELECT($LENGTH(local) > 2: 2, 1: 1)
    set hidden = $TRANSLATE($JUSTIFY("", $LENGTH(local) - keep), " ", "*")

    quit $EXTRACT(local, 1, keep)_hidden_"@"_domain
}

/// Cuts to exactly "limit" characters, with an ellipsis when cut.
ClassMethod Abbreviate(text As %String = "", limit As %Integer = 20) As %String
{
    quit:$LENGTH(text)<=limit text
    quit:limit<=3 $EXTRACT(text, 1, limit)
    quit $EXTRACT(text, 1, limit - 3)_"..."
}

/// Pads to a fixed width. align: "L", "R" or "C".
ClassMethod Pad(text As %String = "", width As %Integer = 10, align As %String = "L") As %String
{
    set text = ..Abbreviate(text, width)
    set len = $LENGTH(text)
    set gap = width - len
    quit:gap<=0 text

    if align = "R" {
        quit $JUSTIFY("", gap)_text
    }
    if align = "C" {
        set left = gap \ 2
        quit $JUSTIFY("", left)_text_$JUSTIFY("", gap - left)
    }
    quit text_$JUSTIFY("", gap)
}

ClassMethod IsRecordNumber(v As %String = "") As %Boolean [ CodeMode = expression ]
{
v ? 1"REG-" 6N
}

ClassMethod IsIsoDate(v As %String = "") As %Boolean
{
    quit:'(v ? 4N1"-"2N1"-"2N) 0
    set m = +$PIECE(v, "-", 2), d = +$PIECE(v, "-", 3)
    quit:(m < 1) || (m > 12) 0
    quit:(d < 1) || (d > 31) 0
    quit 1
}

ClassMethod IsEmail(v As %String = "") As %Boolean
{
    quit:v="" 0
    quit:'(v ? 1.E 1"@" 1.E 1"." 1.E) 0
    quit:v [ " " 0
    quit:$LENGTH(v, "@") '= 2 0
    quit:$PIECE(v, "@", 1) = "" 0
    quit:$PIECE(v, "@", 2) = "" 0
    quit 1
}

}
```

`src/LabStudy/Formatter.cls`:

```objectscript
/// Aligned console output for reports.
/// Columns, widths and alignments travel as lists.
Class LabStudy.Formatter Extends %RegisteredObject
{

/// A horizontal line.
ClassMethod Line(width As %Integer = 60, char As %String = "-") As %Status
{
    write $TRANSLATE($JUSTIFY("", width), " ", char), !
    quit $$$OK
}

/// Total width implied by a list of column widths, plus the gaps.
ClassMethod TotalWidth(widths As %List, gap As %Integer = 1) As %Integer
{
    set total = 0, ptr = 0, n = 0
    while $LISTNEXT(widths, ptr, w) {
        set n = n + 1
        set total = total + $GET(w)
    }
    quit total + ((n - 1) * gap)
}

/// Prints one row.
ClassMethod Row(values As %List, widths As %List, aligns As %List = "", gap As %Integer = 1) As %Status
{
    set spacer = $JUSTIFY("", gap)

    for i = 1:1:$LISTLENGTH(widths) {
        set value = $LISTGET(values, i)
        set width = $LISTGET(widths, i, 10)
        set align = $LISTGET(aligns, i, "L")

        write $SELECT(i = 1: "", 1: spacer)
        write ##class(LabStudy.Text).Pad(value, width, align)
    }
    write !
    quit $$$OK
}

/// Prints a header plus a separator line.
ClassMethod Header(titles As %List, widths As %List, aligns As %List = "", gap As %Integer = 1) As %Status
{
    do ..Row(titles, widths, aligns, gap)
    do ..Line(..TotalWidth(widths, gap))
    quit $$$OK
}

}
```

Acrescente a `src/LabStudy/Patient.cls`:

```objectscript
/// Normalises every incoming value, in one place, on every save.
Method %OnBeforeSave(insert As %Boolean) As %Status [ Private, ServerOnly = 1 ]
{
    set ..Name = ##class(LabStudy.Text).ProperName(..Name)
    set ..RecordNumber = $ZCONVERT(##class(LabStudy.Text).Clean(..RecordNumber), "U")
    set ..Email = $ZCONVERT(##class(LabStudy.Text).Clean(..Email), "L")

    quit $$$OK
}

/// Business rules that the property parameters cannot express.
Method %OnValidateObject() As %Status [ Private, ServerOnly = 1 ]
{
    if (..BirthDate '= "") && (..BirthDate > +$HOROLOG) {
        quit $$$ERROR($$$GeneralError, "BirthDate cannot be in the future")
    }

    if (..RecordNumber '= "") && '##class(LabStudy.Text).IsRecordNumber(..RecordNumber) {
        quit $$$ERROR($$$GeneralError,
            "RecordNumber must look like REG-000000, got: "_..RecordNumber)
    }

    if (..Email '= "") && '##class(LabStudy.Text).IsEmail(..Email) {
        quit $$$ERROR($$$GeneralError, "Invalid e-mail: "_..Email)
    }

    quit $$$OK
}

/// Name ready for display, abbreviated if needed.
Method DisplayName(width As %Integer = 0) As %String
{
    quit:width=0 ..Name
    quit ##class(LabStudy.Text).Abbreviate(..Name, width)
}

/// E-mail with the local part hidden.
Method MaskedEmail() As %String [ CodeMode = expression ]
{
##class(LabStudy.Text).MaskEmail(..Email)
}
```

E reescreva o painel em `src/LabStudy/Reports.cls`:

```objectscript
/// Overview of the whole laboratory, formatted.
ClassMethod Dashboard() As %Status
{
    set W = $LISTBUILD(22, 12)
    set A = $LISTBUILD("L", "R")

    do ##class(LabStudy.Formatter).Line(35, "=")
    write ##class(LabStudy.Text).Pad("LabStudy dashboard", 35, "C"), !
    do ##class(LabStudy.Formatter).Line(35, "=")

    set withExams = ##class(LabStudy.Patient).Statistics(.patients, .exams)

    do ##class(LabStudy.Formatter).Row($LISTBUILD("Patients", patients), W, A)
    do ##class(LabStudy.Formatter).Row($LISTBUILD("Exams", exams), W, A)
    do ##class(LabStudy.Formatter).Row($LISTBUILD("With exams", withExams), W, A)
    do ##class(LabStudy.Formatter).Row($LISTBUILD("Without exams", patients - withExams), W, A)

    write !
    set W2 = $LISTBUILD(10, 8, 14)
    set A2 = $LISTBUILD("L", "R", "R")

    do ##class(LabStudy.Formatter).Header($LISTBUILD("code", "count", "average"), W2, A2)

    set rs = ##class(%SQL.Statement).%ExecDirect(,
        "SELECT TOP 8 TestCode, COUNT(*) AS Total, AVG(ResultValue) AS Average "
        _"FROM LabStudy.EXAM WHERE ResultStatus = 'final' "
        _"GROUP BY TestCode ORDER BY COUNT(*) DESC")

    while rs.%Next() {
        do ##class(LabStudy.Formatter).Row(
            $LISTBUILD(rs.%Get("TestCode"),
                       rs.%Get("Total"),
                       ##class(LabStudy.Text).Number(rs.%Get("Average"), 2)),
            W2, A2)
    }

    do ##class(LabStudy.Formatter).Line(##class(LabStudy.Formatter).TotalWidth(W2))
    quit $$$OK
}
```

E em `src/LabStudy/App.cls`:

```objectscript
Parameter VERSION = "1.6";
```

Execução esperada:

```
LABSTUDY>SET p = ##class(LabStudy.Patient).%New()
LABSTUDY>SET p.Name = "  MARIA   DA silva  ", p.RecordNumber = "reg-000099"
LABSTUDY>SET p.BirthDate = $ZDATEH("1990-05-17",3), p.Email = "  MARIA@Exemplo.COM "
LABSTUDY>WRITE $$$ISOK(p.%Save()), " -> [", p.Name, "] [", p.RecordNumber, "]", !
1 -> [Maria da Silva] [REG-000099]

LABSTUDY>WRITE p.Email, !
maria@exemplo.com

LABSTUDY>WRITE p.MaskedEmail(), !
ma***@exemplo.com

LABSTUDY>SET p.RecordNumber = "REG-99"
LABSTUDY>DO $SYSTEM.Status.DisplayError(p.%Save())
ERROR #5001: RecordNumber must look like REG-000000, got: REG-99

LABSTUDY>DO ##class(LabStudy.Reports).Dashboard()
===================================
        LabStudy dashboard
===================================
Patients                        201
Exams                          2000
With exams                      200
Without exams                     1

code       count        average
-------------------------------
GLU            250         202,68
HGB            250         200,14
CHOL           250         202,86
TRIG           250         202,56
UREA           250         201,92
CREA           250         203,10
TGO            250         199,88
TGP            250         201,44
-------------------------------
```

**Por que cada decisão:**

- **`LabStudy.Text` concentra todas as regras de texto do sistema.** Antes deste capítulo, `$ZSTRIP` e `$ZCONVERT` apareciam espalhados por vários callbacks. Agora existe um lugar só — e quando a regra de capitalização mudar, muda-se ali. **A dispersão de regras de texto é uma das causas mais comuns de inconsistência de dados.**
- **A normalização acontece no `%OnBeforeSave`**, e não no método `Create`. É a lição do Capítulo 3: a regra vale para **toda** gravação, venha de onde vier. Repare que `"  MARIA   DA silva  "` virou `"Maria da Silva"` mesmo tendo sido atribuído diretamente à propriedade.
- **A validação de formato acontece no `%OnValidateObject`**, e a mensagem **inclui o valor recebido**. `"got: REG-99"` transforma um erro em diagnóstico. Uma mensagem que apenas diz "formato inválido" obriga quem recebeu a adivinhar o que enviou de errado.
- **O e-mail é normalizado para minúsculas.** Endereços de e-mail são, na prática, insensíveis a maiúsculas no domínio; normalizar evita duplicatas como `Maria@x.com` e `maria@x.com` convivendo na base.
- **`MaskEmail` mostra as duas primeiras letras.** É o padrão que permite ao usuário reconhecer o próprio endereço sem expô-lo a quem olha a tela por cima do ombro — a preocupação do Capítulo 7 aplicada à camada de apresentação.
- **`Formatter` recebe larguras e alinhamentos como listas.** Chamar `Row($LISTBUILD(a, b, c), W, A)` é legível e não sofre com valores que contenham separadores — a lição do Capítulo 14 aplicada a um caso concreto.
- **`Pad` chama `Abbreviate` antes de alinhar.** Sem isso, um valor mais longo que a coluna destruiria o alinhamento de toda a tabela. Uma tabela desalinhada é ilegível, e ilegível é inútil.
- **As médias saíram no formato brasileiro** (`202,68`), porque passaram por `Text.Number`. Repare que a formatação é aplicada na **borda de saída**, não nos dados — o valor continua numérico no banco, consultável e comparável. **Formate na apresentação, nunca no armazenamento.**
- **O painel agora é uma tabela alinhada**, e não uma sequência de `write` com espaçamento improvisado. A diferença de esforço é pequena; a diferença de legibilidade, grande — e o `Formatter` serve para todos os relatórios seguintes.

---

## 8. Quiz do capítulo

**Q1.** O que `$LENGTH("a;b;c", ";")` devolve?

- A) `5`
- B) `3`
- C) `2`
- D) `1`

---

**Q2.** E `$LENGTH("", ";")`?

- A) `1`
- B) `0`
- C) `""`
- D) Erro.

---

**Q3.** O que `$EXTRACT("Maria", *-2, *)` devolve?

- A) `Mar`
- B) `ria`
- C) `ari`
- D) `a`

---

**Q4.** `$EXTRACT("abc", 99)` faz o quê?

- A) Gera erro.
- B) Devolve string vazia.
- C) Devolve `c`.
- D) Devolve `0`.

---

**Q5.** `$FIND("Maria Silva", "Silva")` devolve o quê?

- A) `7`, onde `Silva` começa.
- B) `12`, a posição imediatamente **após** o texto encontrado.
- C) `1`, porque encontrou.
- D) `5`, o comprimento do texto procurado.

---

**Q6.** Qual é a diferença entre `$REPLACE` e `$TRANSLATE`?

- A) Nenhuma.
- B) `$REPLACE` troca sequências de caracteres; `$TRANSLATE` mapeia caractere a caractere.
- C) `$TRANSLATE` é mais rápido mas faz o mesmo.
- D) `$REPLACE` só funciona com um caractere.

---

**Q7.** O que `$TRANSLATE("a.b,c", ".,", "")` devolve?

- A) `a.b,c`
- B) `abc`
- C) Erro, porque o terceiro argumento está vazio.
- D) `a b c`

---

**Q8.** Qual expressão remove os espaços das duas pontas de um texto?

- A) `$ZSTRIP(t, "W")`
- B) `$ZSTRIP(t, "<>W")`
- C) `$ZSTRIP(t, "*W")`
- D) `$TRANSLATE(t, " ", "")`

---

**Q9.** O que faz `$ZCONVERT(t, "T")`?

- A) Converte para maiúsculas.
- B) Converte para minúsculas.
- C) Coloca a inicial de cada palavra em maiúscula.
- D) Traduz o texto.

---

**Q10.** Qual operador testa se um texto **contém** outro?

- A) `]`
- B) `[`
- C) `]]`
- D) `?`

---

**Q11.** Qual operador compara textos na ordem alfabética?

- A) `<` e `>`
- B) `]`
- C) `[`
- D) `=`

---

**Q12.** `"abc123" ? 1.A` devolve o quê, e por quê?

- A) `1`, porque começa com letras.
- B) `0`, porque o padrão precisa cobrir o texto inteiro e sobraram dígitos.
- C) `1`, porque `.A` aceita qualquer coisa.
- D) Erro de sintaxe.

---

**Q13.** `"" ? 1.N` devolve o quê?

- A) `1`, porque vazio casa com tudo.
- B) `0`, porque `1.` exige pelo menos uma ocorrência.
- C) Erro.
- D) `""`

---

**Q14.** Como obter `"000042"` a partir do número 42?

- A) `$JUSTIFY(42, 6)`
- B) `$TRANSLATE($JUSTIFY(42, 6), " ", "0")`
- C) `$FNUMBER(42, "0", 6)`
- D) `$EXTRACT("000000"_42, 1, 6)`

---

**Q15.** `$FNUMBER(1234.5, ",", 2)` produz `1,234.50`. Como obter o formato brasileiro?

- A) `$FNUMBER(1234.5, ".", 2)`
- B) `$TRANSLATE($FNUMBER(1234.5, ",", 2), ".,", ",.")`
- C) `$REPLACE($FNUMBER(1234.5, ",", 2), ",", ".")`
- D) Não é possível.

---

**Q16.** `$ASCII("")` devolve o quê?

- A) `0`
- B) `-1`
- C) `""`
- D) Erro.

---

**Q17.** Qual é a regra de escolha entre `$PIECE` e `$LIST`?

- A) Usar sempre `$PIECE`.
- B) `$LIST` por dentro do sistema; `$PIECE` na fronteira com dados externos separados.
- C) `$PIECE` para números, `$LIST` para texto.
- D) `$LIST` só em globais.

---

### Gabarito comentado

**Q1 — Resposta: B.**
- **B está certa:** dois separadores produzem três pedaços.
- **A está errada:** `5` seria o comprimento em caracteres.
- **C e D estão erradas:** a contagem é separadores + 1.

**Q2 — Resposta: B.**
- **B está certa:** o texto vazio tem **zero** pedaços. É a exceção à regra de "separadores + 1".
- **A está errada:** essa é a resposta intuitiva e errada.
- **C e D estão erradas:** devolve um número, sem erro.

**Q3 — Resposta: B.**
- **B está certa:** `*-2` é a antepenúltima posição, e `*` é a última: os três últimos caracteres.
- **A está errada:** seriam os três primeiros.
- **C e D estão erradas:** não correspondem à faixa pedida.

**Q4 — Resposta: B.**
- **B está certa:** `$EXTRACT` fora de faixa devolve vazio.
- **A está errada:** quem gera erro fora de faixa é `$LIST`. As duas funções se comportam de forma diferente.
- **C e D estão erradas:** não há valor devolvido nessas condições.

**Q5 — Resposta: B.**
- **B está certa:** `$FIND` devolve a posição **após** o achado, para que a busca possa continuar dali.
- **A está errada:** para o início, subtraia o comprimento do texto procurado.
- **C e D estão erradas:** o retorno é uma posição, não um indicador nem um comprimento.

**Q6 — Resposta: B.**
- **B está certa:** um trabalha com sequências, o outro com mapeamento caractere a caractere.
- **A e C estão erradas:** produzem resultados diferentes para a mesma entrada.
- **D está errada:** `$REPLACE` aceita sequências de qualquer tamanho.

**Q7 — Resposta: B.**
- **B está certa:** quando o destino é mais curto que a origem, os caracteres sobrando são **removidos**.
- **A está errada:** os caracteres são de fato eliminados.
- **C está errada:** o terceiro argumento vazio é válido e intencional.
- **D está errada:** eles não viram espaço; somem.

**Q8 — Resposta: B.**
- **B está certa:** `<` é início, `>` é fim, `W` é espaço em branco.
- **A está errada:** falta indicar onde agir.
- **C está errada:** `*` removeria **todos** os espaços, inclusive os internos.
- **D está errada:** também removeria os internos.

**Q9 — Resposta: C.**
- **C está certa:** `"T"` é *title case*: inicial maiúscula em cada palavra.
- **A e B estão erradas:** são `"U"` e `"L"`.
- **D está errada:** `$ZCONVERT` não traduz idiomas.

**Q10 — Resposta: B.**
- **B está certa:** `[` é o operador "contém".
- **A está errada:** `]` é "vem depois" na ordenação de texto.
- **C está errada:** `]]` é "vem depois" na ordenação padrão de subscritos.
- **D está errada:** `?` é casamento de padrão.

**Q11 — Resposta: B.**
- **B está certa:** `]` compara textos caractere a caractere.
- **A está errada:** `<` e `>` são comparações **numéricas**.
- **C está errada:** `[` testa continência.
- **D está errada:** `=` testa igualdade.

**Q12 — Resposta: B.**
- **B está certa:** o padrão precisa casar com o texto **inteiro**; os dígitos ficaram de fora.
- **A e C estão erradas:** não há casamento parcial no operador `?`.
- **D está errada:** a sintaxe é válida.

**Q13 — Resposta: B.**
- **B está certa:** `1.` significa "um ou mais", e o texto vazio tem zero.
- **A está errada:** `.N` (zero ou mais) casaria; `1.N` não.
- **C e D estão erradas:** o operador devolve 1 ou 0.

**Q14 — Resposta: B.**
- **B está certa:** `$JUSTIFY` alinha à direita com espaços e `$TRANSLATE` troca espaços por zeros.
- **A está errada:** produz espaços, não zeros.
- **C está errada:** não é essa a sintaxe de `$FNUMBER`.
- **D está errada:** funciona por acaso para 42, mas quebra para números com mais dígitos que a folga.

**Q15 — Resposta: B.**
- **B está certa:** a troca cruzada num só `$TRANSLATE` converte ponto em vírgula e vírgula em ponto simultaneamente.
- **A está errada:** o primeiro argumento de formato não define o separador decimal dessa forma.
- **C está errada:** um `$REPLACE` faria uma troca por vez e se atrapalharia na segunda.
- **D está errada:** é perfeitamente possível.

**Q16 — Resposta: B.**
- **B está certa:** `$ASCII` sobre texto vazio ou posição inválida devolve `-1`.
- **A está errada:** `0` é um código de caractere válido.
- **C e D estão erradas:** devolve um número, sem erro.

**Q17 — Resposta: B.**
- **B está certa:** listas por dentro, separadores só onde o mundo externo os impõe.
- **A está errada:** ignora a fragilidade de separadores com dados arbitrários.
- **C está errada:** o tipo do conteúdo não é o critério.
- **D está errada:** `$LIST` funciona em qualquer contexto.

---

## 9. Resumo relâmpago

1. Três óculos: **`$EXTRACT`** (posição de caractere), **`$PIECE`** (campos separados), **`[` / `$FIND`** (busca).
2. **`$LENGTH(t)`** conta caracteres; **`$LENGTH(t, sep)`** conta pedaços = separadores + 1 — **exceto no texto vazio, que dá 0**.
3. **`*`** significa "o último": `$EXTRACT(t, *)`, `$EXTRACT(t, *-3, *)`, `$EXTRACT(t, 2, *)`.
4. **`$EXTRACT` fora de faixa devolve vazio**; `$LIST` fora de faixa gera **erro**. Comportamentos diferentes.
5. `$EXTRACT` e `$PIECE` podem ficar **à esquerda de um `SET`**. Atribuir a um pedaço inexistente **cria os separadores**.
6. **`$PIECE(t, sep, de, ate)`** devolve **um texto** com os separadores internos, não uma lista.
7. **`$PIECE` não distingue "campo vazio" de "campo inexistente"** — os dois devolvem vazio.
8. **`$FIND` devolve a posição APÓS o achado**, ou `0`. Para o início, subtraia o comprimento.
9. **`[`** = contém. **`]`** = vem depois como texto. **`]]`** = vem depois na ordenação padrão de subscritos. **`<` `>`** são **numéricos**.
10. **`$REPLACE`** troca sequências; **`$TRANSLATE`** mapeia caractere a caractere — e **remove** quando o destino é mais curto.
11. **`$TRANSLATE(n, ".,", ",.")`** faz a troca cruzada que converte número americano em brasileiro.
12. **`$ZSTRIP(t, ondeOque)`**: onde = `<`, `>`, `<>`, `*`; o quê = `W`, `P`, `C`, `N`, `A`, `E`. O uso mais comum é **`"<>W"`**.
13. **`$ZCONVERT(t, "U"/"L"/"T"/"W")`** para caixa; com **três argumentos**, converte **codificação** (`"I"`/`"O"` mais `"UTF8"`).
14. **`$JUSTIFY(v, largura)`** alinha à direita; **`$JUSTIFY(n, largura, decimais)`** arredonda e alinha.
15. **`$TRANSLATE($JUSTIFY(n, largura), " ", "0")`** é o preenchimento com zeros.
16. **`$FNUMBER`** formata números, em padrão americano por padrão. `","`, `"+"`, `"-"` e `"P"` são os códigos mais usados.
17. **`$CHAR`** e **`$ASCII`** convertem entre caractere e código; `$ASCII("")` devolve **`-1`**.
18. **Operador `?`**: classes `A N U L P C E` e literais entre aspas; contadores `3N`, `.N` (zero ou mais), `1.N` (um ou mais), `2.5N` (faixa); alternativas entre parênteses.
19. **O padrão precisa cobrir o texto inteiro.** Use `.E` para permitir sobra.
20. **`.N` casa com vazio; `1.N` não.** Em campo obrigatório, use `1.`.
21. **Padrão valida forma, não conteúdo.** `"2026-13-01"` tem formato de data e não é data.
22. Comparação de texto **diferencia maiúsculas**: normalize os dois lados com `$ZCONVERT`.
23. **`$PIECE` na fronteira, `$LIST` por dentro** — e `$EXTRACT` quando a posição é fixa.
24. Concatenar milhares de linhas arrisca **`<MAXSTRING>`**: use stream.
25. **Formate na apresentação, nunca no armazenamento.**

---

## 10. Cartões de memorização

**Frente:** Quais são as três "ferramentas-óculos" para texto?
**Verso:** `$EXTRACT` (posição), `$PIECE` (campos separados), `[` e `$FIND` (busca).

**Frente:** `$LENGTH("a;b;c", ";")` e `$LENGTH("", ";")` devolvem o quê?
**Verso:** `3` e `0`. Pedaços = separadores + 1, exceto no texto vazio, que tem zero.

**Frente:** O que significa `*` nas funções de texto?
**Verso:** "O último". `$EXTRACT(t, *)` é o último caractere; `$EXTRACT(t, *-2, *)` são os três últimos.

**Frente:** `$EXTRACT` fora de faixa: erro ou vazio?
**Verso:** **Vazio**. Quem gera erro fora de faixa é `$LIST`.

**Frente:** O que `$FIND` devolve?
**Verso:** A posição **imediatamente após** o texto encontrado, ou `0`. Para o início, subtraia o comprimento.

**Frente:** Diferença entre `$REPLACE` e `$TRANSLATE`.
**Verso:** `$REPLACE` troca **sequências**; `$TRANSLATE` mapeia **caractere a caractere**.

**Frente:** O que `$TRANSLATE(t, ".,", "")` faz?
**Verso:** **Remove** todos os pontos e vírgulas. Destino mais curto que a origem significa remoção.

**Frente:** Como converter um número americano para o formato brasileiro?
**Verso:** `$TRANSLATE($FNUMBER(n, ",", 2), ".,", ",.")` — troca cruzada num passo.

**Frente:** Como remover espaços das duas pontas?
**Verso:** `$ZSTRIP(t, "<>W")`. `<` é início, `>` é fim, `W` é espaço em branco.

**Frente:** Quais são os modos de caixa do `$ZCONVERT`?
**Verso:** `"U"` maiúsculas, `"L"` minúsculas, `"T"` inicial de cada palavra, `"W"` inicial da primeira palavra.

**Frente:** O que faz `$ZCONVERT` com três argumentos?
**Verso:** Converte **codificação de caracteres**: `"I"` entrando, `"O"` saindo, mais o nome da codificação (`"UTF8"`).

**Frente:** Qual operador testa "contém"? E "vem depois"?
**Verso:** `[` testa continência. `]` testa ordem de texto. `]]` testa ordem padrão de subscritos.

**Frente:** `<` e `>` comparam texto ou número?
**Verso:** **Número**. Para ordem alfabética, use `]`.

**Frente:** Como preencher um número com zeros à esquerda?
**Verso:** `$TRANSLATE($JUSTIFY(42, 6), " ", "0")` → `000042`.

**Frente:** `$JUSTIFY` com três argumentos faz o quê?
**Verso:** Arredonda para o número de casas decimais **e** alinha à direita na largura dada.

**Frente:** O que `$ASCII("")` devolve?
**Verso:** `-1`.

**Frente:** Quais são as classes do operador de padrão `?`
**Verso:** `A` letras, `N` dígitos, `U` maiúsculas, `L` minúsculas, `P` pontuação, `C` controle, `E` qualquer coisa, e literais entre aspas.

**Frente:** Diferença entre `.N` e `1.N` num padrão.
**Verso:** `.N` é zero ou mais (casa com vazio); `1.N` é um ou mais (não casa com vazio).

**Frente:** `"abc123" ? 1.A` é verdadeiro?
**Verso:** Não. O padrão precisa cobrir o texto **inteiro**, e sobraram os dígitos. Use `1.A.N` ou `1.A.E`.

**Frente:** O padrão `?` valida conteúdo?
**Verso:** Não, só **forma**. `"2026-13-01"` tem formato de data e não é uma data válida.

**Frente:** Como comparar textos ignorando maiúsculas?
**Verso:** Normalizando os dois lados: `$ZCONVERT(a, "U") = $ZCONVERT(b, "U")`.

**Frente:** Quando usar `$EXTRACT`, `$PIECE` e `$LIST`?
**Verso:** `$EXTRACT` para posição fixa de caractere; `$PIECE` para campos separados vindos de fora; `$LIST` para dados internos.

**Frente:** Onde formatar números e datas?
**Verso:** Na **apresentação**. No armazenamento, guarde o valor bruto, que é consultável, comparável e indexável.

---

Digite CONTINUAR para o próximo capítulo.
