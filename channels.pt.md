---
description: Workshop Básico de Treinamento Nextflow
---

# Canais

Os canais são uma estrutura de dados chave do Nextflow que permite a implementação de fluxos de trabalho computacionais orientados a função reativa com base no paradigma de programação [Dataflow](https://en.wikipedia.org/wiki/Dataflow_programming) .

Eles são usados para conectar logicamente tarefas entre si ou para implementar transformações de dados de estilo funcional.

<figure class="excalidraw">--8</figure>

## Tipos de canal

O Nextflow distingue dois tipos diferentes de canais: canais **de fila** e canais **de valor** .

### canal de fila

Um canal *de fila* é uma fila *FIFO* unidirecional *assíncrona* que conecta dois processos ou operadores.

- *assíncrono* significa que as operações são sem bloqueio.
- *unidirecional* significa que os dados fluem de um produtor para um consumidor.
- *FIFO* significa que os dados são entregues na mesma ordem em que são produzidos. Primeiro a entrar, primeiro a sair.

Um canal de fila é criado implicitamente por definições de saída de processo ou usando fábricas de canal, como [Channel.of](https://www.nextflow.io/docs/latest/channel.html#of) ou [Channel.fromPath](https://www.nextflow.io/docs/latest/channel.html#frompath) .

Tente os seguintes trechos:

!!! informações ""

```
Click the :material-plus-circle: icons in the code for explanations.
```

```groovy
ch = Channel.of(1, 2, 3)
println(ch) // (1)!
ch.view() // (2)!
```

1. Use a função de linha de impressão integrada `println` para imprimir o canal `ch`
2. Aplicar o operador `view` channel ao canal `ch` imprime cada item emitido pelos canais

!!! exercício

```
Try to execute this snippet. You can do that by creating a new `.nf` file or by editing an already existing `.nf` file.

```groovy linenums="1"
ch = Channel.of(1, 2, 3)
ch.view()
```
```

### canais de valor

Um canal **de valor** (também conhecido como canal singleton), por definição, está vinculado a um único valor e pode ser lido infinitas vezes sem consumir seu conteúdo. Um canal `value` é criado usando a fábrica de canal [de valor](https://www.nextflow.io/docs/latest/channel.html#value) ou por operadores que retornam um único valor, como [first](https://www.nextflow.io/docs/latest/operator.html#first) , [last](https://www.nextflow.io/docs/latest/operator.html#last) , [collect](https://www.nextflow.io/docs/latest/operator.html#operator-collect) , [count](https://www.nextflow.io/docs/latest/operator.html#operator-count) , [min](https://www.nextflow.io/docs/latest/operator.html#operator-min) , [max](https://www.nextflow.io/docs/latest/operator.html#operator-max) , [reduce](https://www.nextflow.io/docs/latest/operator.html#operator-reduce) e [sum](https://www.nextflow.io/docs/latest/operator.html#operator-sum) .

Para entender melhor a diferença entre canais de valor e de fila, salve o trecho abaixo como `example.nf` .

```groovy
ch1 = Channel.of(1, 2, 3)
ch2 = Channel.of(1)

process SUM {
    input:
    val x
    val y

    output:
    stdout

    script:
    """
    echo \$(($x+$y))
    """
}

workflow {
    SUM(ch1, ch2).view()
}
```

Ao rodar o script, ele imprime apenas 2, como você pode ver abaixo:

```console
2
```

Para entender o motivo, podemos inspecionar o canal da fila e executar o Nextflow com DSL1 nos dá uma compreensão mais explícita do que está por trás das cortinas.

```groovy
ch1 = Channel.of(1)
println ch1
```

```console
$ nextflow run example.nf -dsl1
...
DataflowQueue(queue=[DataflowVariable(value=1), DataflowVariable(value=groovyx.gpars.dataflow.operator.PoisonPill@34be065a)])
```

Temos o valor 1 como único elemento do nosso canal de fila e uma pílula de veneno, que dirá ao processo que não há mais nada para ser consumido. É por isso que temos apenas uma saída para o exemplo acima, que é 2. Vamos inspecionar um canal de valor agora.

```groovy
ch1 = Channel.value(1)
println ch1
```

```console
$ nextflow run example.nf -dsl1
...
DataflowVariable(value=1)
```

Não há pílula de veneno, e é por isso que obtemos uma saída diferente com o código abaixo, onde `ch2` é transformado em um canal de valor por meio do `first` operador.

```groovy
ch1 = Channel.of(1, 2, 3)
ch2 = Channel.of(1)

process SUM {
    input:
    val x
    val y

    output:
    stdout

    script:
    """
    echo \$(($x+$y))
    """
}

workflow {
    SUM(ch1, ch2.first()).view()
}
```

```console
4

3

2
```

Além disso, em muitas situações, o Nextflow converterá implicitamente variáveis em canais de valor quando forem usadas em uma chamada de processo. Por exemplo, quando você chama um processo com um parâmetro de fluxo de trabalho ( `params.example` ) que possui um valor de string, ele é convertido automaticamente em um canal de valor.

## fábricas de canais

Estes são comandos do Nextflow para criar canais que possuem entradas e funções esperadas implícitas.

### `value()`

A fábrica de canal `value` é usada para criar um canal *de valor* . Um argumento não `null` opcional pode ser especificado para vincular o canal a um valor específico. Por exemplo:

```groovy
ch1 = Channel.value() // (1)!
ch2 = Channel.value('Hello there') // (2)!
ch3 = Channel.value([1, 2, 3, 4, 5]) // (3)!
```

1. Cria um canal de valor *vazio*
2. Cria um canal de valor e vincula uma string a ele
3. Cria um canal de valor e vincula a ele um objeto de lista que será emitido como uma única emissão

### `of()`

A fábrica `Channel.of` permite a criação de um canal de fila com os valores especificados como argumentos.

```groovy
ch = Channel.of(1, 3, 5, 7)
ch.view { "value: $it" }
```

A primeira linha neste exemplo cria uma variável `ch` que contém um objeto de canal. Este canal emite os valores especificados como parâmetro `of` fábrica de canais. Assim, a segunda linha imprimirá o seguinte:

```console
value: 1
value: 3
value: 5
value: 7
```

A fábrica de canais `Channel.of` funciona de maneira semelhante a `Channel.from` (que agora está [obsoleto](https://www.nextflow.io/docs/latest/channel.html#of) ), corrigindo alguns comportamentos inconsistentes do último e fornecendo melhor manuseio ao especificar um intervalo de valores. Por exemplo, o seguinte funciona com um intervalo de 1 a 23:

```groovy
Channel
    .of(1..23, 'X', 'Y')
    .view()
```

### `fromList()`

A fábrica de canais `Channel.fromList` cria um canal emitindo os elementos fornecidos por um objeto de lista especificado como um argumento:

```groovy
list = ['hello', 'world']

Channel
    .fromList(list)
    .view()
```

### `fromPath()`

A fábrica de canais `fromPath` cria um canal de fila emitindo um ou mais arquivos correspondentes ao padrão glob especificado.

```groovy
Channel.fromPath('./data/meta/*.csv')
```

Este exemplo cria um canal e emite tantos itens quanto arquivos com extensão `csv` na pasta `./data/meta` . Cada elemento é um objeto de arquivo implementando a interface [Path](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Paths.html) .

!!! dica

```
Two asterisks, i.e. `**`, works like `*` but cross directory boundaries. This syntax is generally used for matching complete paths. Curly brackets specify a collection of sub-patterns.
```

Nome | Descrição
--- | ---
glob | Quando `true` interpreta os caracteres `*` , `?` , `[]` e `{}` como curingas glob, caso contrário, trata-os como caracteres normais (padrão: `true` )
tipo | Tipo de caminho retornado, `file` , `dir` ou `any` (padrão: `file` )
escondido | Quando `true` inclui arquivos ocultos nos caminhos resultantes (padrão: `false` )
profundidade máxima | Número máximo de níveis de diretório a serem visitados (padrão: `no limit` )
seguirLinks | Quando links simbólicos `true` são seguidos durante a travessia da árvore de diretórios, caso contrário, eles são gerenciados como arquivos (padrão: `true` )
relativo | Quando os caminhos de retorno `true` são relativos ao diretório mais comum (padrão: `false` )
checkIfExists | Quando `true` lança uma exceção quando o caminho especificado não existe no sistema de arquivos (padrão: `false` )

Saiba mais sobre a sintaxe dos padrões glob [neste link](https://docs.oracle.com/javase/tutorial/essential/io/fileOps.html#glob) .

!!! exercício

```
Use the `Channel.fromPath` channel factory to create a channel emitting all files with the suffix `.fq` in the `data/ggal/` directory and any subdirectory, in addition to hidden files. Then print the file names.

??? solution

    ```groovy linenums="1"
    Channel.fromPath('./data/ggal/**.fq', hidden: true)
        .view()
    ```
```

### `fromFilePairs()`

A fábrica de canais `fromFilePairs` cria um canal emitindo os pares de arquivos correspondentes a um padrão glob fornecido pelo usuário. Os arquivos correspondentes são emitidos como tuplas, em que o primeiro elemento é a chave de agrupamento do par correspondente e o segundo elemento é a lista de arquivos (classificados em ordem lexicográfica).

```groovy
Channel
    .fromFilePairs('./data/ggal/*_{1,2}.fq')
    .view()
```

Ele produzirá uma saída semelhante à seguinte:

```groovy
[liver, [/workspace/gitpod/nf-training/data/ggal/liver_1.fq, /workspace/gitpod/nf-training/data/ggal/liver_2.fq]]
[gut, [/workspace/gitpod/nf-training/data/ggal/gut_1.fq, /workspace/gitpod/nf-training/data/ggal/gut_2.fq]]
[lung, [/workspace/gitpod/nf-training/data/ggal/lung_1.fq, /workspace/gitpod/nf-training/data/ggal/lung_2.fq]]
```

!!! aviso

```
The glob pattern _must_ contain at least a star wildcard character (`*`).
```

Nome | Descrição
--- | ---
tipo | Tipo de caminhos retornados, `file` , `dir` ou `any` (padrão: `file` )
escondido | Quando `true` inclui arquivos ocultos nos caminhos resultantes (padrão: `false` )
profundidade máxima | Número máximo de níveis de diretório a serem visitados (padrão: `no limit` )
seguirLinks | Quando links simbólicos `true` são seguidos durante a travessia da árvore de diretórios, caso contrário, eles são gerenciados como arquivos (padrão: `true` )
tamanho | Define o número de arquivos que cada item emitido deve conter (padrão: `2` ). Defina como `-1` para qualquer
plano | Quando `true` os arquivos correspondentes são produzidos como elementos únicos nas tuplas emitidas (padrão: `false` )
checkIfExists | Quando `true` , lança uma exceção do caminho especificado que não existe no sistema de arquivos (padrão: `false` )

!!! exercício

```
Use the `fromFilePairs` channel factory to create a channel emitting all pairs of fastq read in the `data/ggal/` directory and print them. Then use the `flat: true` option and compare the output with the previous execution.

??? solution

    Use the following, with or without `flat: true`:

    ```groovy linenums="1"
    Channel.fromFilePairs('./data/ggal/*_{1,2}.fq', flat: true)
        .view()
    ```

    Then check the square brackets around the file names, to see the difference with `flat`.
```

### `fromSRA()`

A fábrica de canais `Channel.fromSRA` permite consultar o arquivo [NCBI SRA](https://www.ncbi.nlm.nih.gov/sra) e retorna um canal que emite os arquivos FASTQ correspondentes aos critérios de seleção especificados.

A consulta pode ser ID(s) de projeto ou número(s) de acesso suportados pela [API NCBI ESearch](https://www.ncbi.nlm.nih.gov/books/NBK25499/#chapter4.ESearch) .

!!! informação

```
This function now requires an API key you can only get by logging into your NCBI account.
```

??? exemplo "Instruções para login NCBI e aquisição de chave"

```
1. Go to: <https://www.ncbi.nlm.nih.gov/>
2. Click the top right "Log in" button to sign into NCBI. Follow their instructions.
3. Once into your account, click the button at the top right, usually your ID.
4. Go to Account settings
5. Scroll down to the API Key Management section.
6. Click on "Create an API Key".
7. The page will refresh and the key will be displayed where the button was. Copy your key.
```

Por exemplo, o trecho a seguir imprimirá o conteúdo de um ID de projeto NCBI:

```groovy
params.ncbi_api_key = '<Your API key here>'

Channel
    .fromSRA(['SRP073307'], apiKey: params.ncbi_api_key)
    .view()
```

!!! informações ""

```
:material-lightbulb: Replace `<Your API key here>` with your API key.
```

Isso deve imprimir:

```groovy
[SRR3383346, [/vol1/fastq/SRR338/006/SRR3383346/SRR3383346_1.fastq.gz, /vol1/fastq/SRR338/006/SRR3383346/SRR3383346_2.fastq.gz]]
[SRR3383347, [/vol1/fastq/SRR338/007/SRR3383347/SRR3383347_1.fastq.gz, /vol1/fastq/SRR338/007/SRR3383347/SRR3383347_2.fastq.gz]]
[SRR3383344, [/vol1/fastq/SRR338/004/SRR3383344/SRR3383344_1.fastq.gz, /vol1/fastq/SRR338/004/SRR3383344/SRR3383344_2.fastq.gz]]
[SRR3383345, [/vol1/fastq/SRR338/005/SRR3383345/SRR3383345_1.fastq.gz, /vol1/fastq/SRR338/005/SRR3383345/SRR3383345_2.fastq.gz]]
// (remaining omitted)
```

Vários IDs de acesso podem ser especificados usando um objeto de lista:

```groovy
ids = ['ERR908507', 'ERR908506', 'ERR908505']
Channel
    .fromSRA(ids, apiKey: params.ncbi_api_key)
    .view()
```

```groovy
[ERR908507, [/vol1/fastq/ERR908/ERR908507/ERR908507_1.fastq.gz, /vol1/fastq/ERR908/ERR908507/ERR908507_2.fastq.gz]]
[ERR908506, [/vol1/fastq/ERR908/ERR908506/ERR908506_1.fastq.gz, /vol1/fastq/ERR908/ERR908506/ERR908506_2.fastq.gz]]
[ERR908505, [/vol1/fastq/ERR908/ERR908505/ERR908505_1.fastq.gz, /vol1/fastq/ERR908/ERR908505/ERR908505_2.fastq.gz]]
```

!!! informação

```
Read pairs are implicitly managed and are returned as a list of files.
```

É fácil usar este canal como uma entrada usando a sintaxe usual do Nextflow. O código abaixo cria um canal contendo duas amostras de um estudo SRA público e executa o FASTQC nos arquivos resultantes. Ver:

```groovy
params.ncbi_api_key = '<Your API key here>'

params.accession = ['ERR908507', 'ERR908506']

process FASTQC {
    input:
    tuple val(sample_id), path(reads_file)

    output:
    path("fastqc_${sample_id}_logs")

    script:
    """
    mkdir fastqc_${sample_id}_logs
    fastqc -o fastqc_${sample_id}_logs -f fastq -q ${reads_file}
    """
}

workflow {
    reads = Channel.fromSRA(params.accession, apiKey: params.ncbi_api_key)
    FASTQC(reads)
}
```

Se você deseja executar o fluxo de trabalho acima e não possui o fastqc instalado em sua máquina, não se esqueça do que aprendeu na seção anterior. Execute este fluxo de trabalho com `-with-docker biocontainers/fastqc:v0.11.5` , por exemplo.

### arquivos de texto

O operador `splitText` permite dividir strings de várias linhas ou itens de arquivo de texto, emitidos por um canal de origem em blocos contendo n linhas, que serão emitidos pelo canal resultante. Ver:

```groovy
Channel
    .fromPath('data/meta/random.txt') // (1)!
    .splitText() // (2)!
    .view() // (3)!
```

1. Instrui o Nextflow a criar um canal a partir do caminho `data/meta/random.txt`
2. O operador `splitText` divide cada item em blocos de uma linha por padrão.
3. Veja o conteúdo do canal.

Você pode definir o número de linhas em cada bloco usando o parâmetro `by` , conforme mostrado no exemplo a seguir:

```groovy
Channel
    .fromPath('data/meta/random.txt')
    .splitText(by: 2)
    .subscribe {
        print it;
        print "--- end of the chunk ---\n"
    }
```

!!! informação

```
The `subscribe` operator permits execution of user defined functions each time a new value is emitted by the source channel.
```

Um encerramento opcional pode ser especificado para transformar os blocos de texto produzidos pelo operador. O exemplo a seguir mostra como dividir arquivos de texto em blocos de 10 linhas e transformá-los em letras maiúsculas:

```groovy
Channel
    .fromPath('data/meta/random.txt')
    .splitText(by: 10) { it.toUpperCase() }
    .view()
```

Você também pode fazer contagens para cada linha:

```groovy
count = 0

Channel
    .fromPath('data/meta/random.txt')
    .splitText()
    .view { "${count++}: ${it.toUpperCase().trim()}" }
```

Finalmente, você também pode usar o operador em arquivos simples (fora do contexto do canal):

```groovy
def f = file('data/meta/random.txt')
def lines = f.splitText()
def count = 0
for (String row : lines) {
    log.info "${count++} ${row.toUpperCase()}"
}
```

### Valores separados por vírgula (.csv)

O operador `splitCsv` permite analisar itens de texto emitidos por um canal, que são formatados em CSV.

Em seguida, ele os divide em registros ou os agrupa como uma lista de registros com um comprimento especificado.

No caso mais simples, basta aplicar o operador `splitCsv` a um canal que emite arquivos de texto ou entradas de texto no formato CSV. Por exemplo, para visualizar apenas a primeira e a quarta colunas:

```groovy
Channel
    .fromPath("data/meta/patients_1.csv")
    .splitCsv()
    // row is a list object
    .view { row -> "${row[0]},${row[3]}" }
```

Quando o CSV começa com uma linha de cabeçalho definindo os nomes das colunas, você pode especificar o parâmetro `header: true` que permite fazer referência a cada valor por seu nome de coluna, conforme mostrado no exemplo a seguir:

```groovy
Channel
    .fromPath("data/meta/patients_1.csv")
    .splitCsv(header: true)
    // row is a list object
    .view { row -> "${row.patient_id},${row.num_samples}" }
```

Como alternativa, você pode fornecer nomes de cabeçalho personalizados especificando uma lista de strings no parâmetro de cabeçalho, conforme mostrado abaixo:

```groovy
Channel
    .fromPath("data/meta/patients_1.csv")
    .splitCsv(header: ['col1', 'col2', 'col3', 'col4', 'col5'])
    // row is a list object
    .view { row -> "${row.col1},${row.col4}" }
```

Você também pode processar vários arquivos CSV ao mesmo tempo:

```groovy
Channel
    .fromPath("data/meta/patients_*.csv") // <-- just use a pattern
    .splitCsv(header: true)
    .view { row -> "${row.patient_id}\t${row.num_samples}" }
```

!!! dica

```
Notice that you can change the output format simply by adding a different delimiter.
```

Finalmente, você também pode operar em arquivos CSV fora do contexto do canal:

```groovy
def f = file('data/meta/patients_1.csv')
def lines = f.splitCsv()
for (List row : lines) {
    log.info "${row[0]} -- ${row[2]}"
}
```

!!! exercício

```
Try inputting fastq reads into the RNA-Seq workflow from earlier using `.splitCsv`.

??? solution

    Add a CSV text file containing the following, as an example input with the name "fastq.csv":

    ```csv
    gut,/workspace/gitpod/nf-training/data/ggal/gut_1.fq,/workspace/gitpod/nf-training/data/ggal/gut_2.fq
    ```

    Then replace the input channel for the reads in `script7.nf`. Changing the following lines:

    ```groovy linenums="1"
    Channel
        .fromFilePairs(params.reads, checkIfExists: true)
        .set { read_pairs_ch }
    ```

    To a splitCsv channel factory input:

    ```groovy linenums="1" hl_lines="2 3 4"
    Channel
        .fromPath("fastq.csv")
        .splitCsv()
        .view { row -> "${row[0]},${row[1]},${row[2]}" }
        .set { read_pairs_ch }
    ```

    Finally, change the cardinality of the processes that use the input data. For example, for the quantification process, change it from:

    ```groovy linenums="1"
    process QUANTIFICATION {
        tag "$sample_id"

        input:
        path salmon_index
        tuple val(sample_id), path(reads)

        output:
        path sample_id, emit: quant_ch

        script:
        """
        salmon quant --threads $task.cpus --libType=U -i $salmon_index -1 ${reads[0]} -2 ${reads[1]} -o $sample_id
        """
    }
    ```

    To:

    ```groovy linenums="1" hl_lines="6 13"
    process QUANTIFICATION {
        tag "$sample_id"

        input:
        path salmon_index
        tuple val(sample_id), path(reads1), path(reads2)

        output:
        path sample_id, emit: quant_ch

        script:
        """
        salmon quant --threads $task.cpus --libType=U -i $salmon_index -1 ${reads1} -2 ${reads2} -o $sample_id
        """
    }
    ```

    Repeat the above for the fastqc step.

    ```groovy linenums="1"  hl_lines="5 13"
    process FASTQC {
        tag "FASTQC on $sample_id"

        input:
        tuple val(sample_id), path(reads1), path(reads2)

        output:
        path "fastqc_${sample_id}_logs"

        script:
        """
        mkdir fastqc_${sample_id}_logs
        fastqc -o fastqc_${sample_id}_logs -f fastq -q ${reads1} ${reads2}
        """
    }
    ```

    Now the workflow should run from a CSV file.
```

### Valores separados por tabulação (.tsv)

A análise de arquivos TSV funciona de maneira semelhante, basta adicionar a opção `sep: '\t'` no contexto `splitCsv` :

```groovy
Channel
    .fromPath("data/meta/regions.tsv", checkIfExists: true)
    // use `sep` option to parse TAB separated files
    .splitCsv(sep: '\t')
    .view()
```

!!! exercício

```
Try using the tab separation technique on the file `data/meta/regions.tsv`, but print just the first column, and remove the header.


??? solution

    ```groovy linenums="1"
    Channel
        .fromPath("data/meta/regions.tsv", checkIfExists: true)
        // use `sep` option to parse TAB separated files
        .splitCsv(sep: '\t', header: true)
        // row is a list object
        .view { row -> "${row.patient_id}" }
    ```
```

## Formatos de arquivo mais complexos

### JSON

Também podemos analisar facilmente o formato de arquivo JSON usando o seguinte esquema bacana:

```groovy
import groovy.json.JsonSlurper

def f = file('data/meta/regions.json')
def records = new JsonSlurper().parse(f)


for (def entry : records) {
    log.info "$entry.patient_id -- $entry.feature"
}
```

!!! aviso

```
When using an older JSON version, you may need to replace `parse(f)` with `parseText(f.text)`
```

### YAML

Isso também pode ser usado como uma forma de analisar arquivos YAML:

```groovy
import org.yaml.snakeyaml.Yaml

def f = file('data/meta/regions.yml')
def records = new Yaml().load(f)


for (def entry : records) {
    log.info "$entry.patient_id -- $entry.feature"
}
```

### Armazenamento de analisadores em módulos

A melhor maneira de armazenar scripts de analisador é mantê-los em um arquivo de módulo Nextflow.

Veja o seguinte script Nextflow:

```groovy
include { parseJsonFile } from './modules/parsers.nf'

process FOO {
    input:
    tuple val(patient_id), val(feature)

    output:
    stdout

    script:
    """
    echo $patient_id has $feature as feature
    """
}

workflow {
    Channel.fromPath('data/meta/regions*.json')
        | flatMap { parseJsonFile(it) }
        | map { record -> [record.patient_id, record.feature] }
        | unique
        | FOO
        | view
}
```

Para que este script funcione, um arquivo de módulo chamado `parsers.nf` precisa ser criado e armazenado em uma pasta de módulos no diretório atual.

O arquivo `parsers.nf` deve conter a função `parseJsonFile` . Por exemplo:

```groovy
import groovy.json.JsonSlurper

def parseJsonFile(json_file) {
    def f = file('data/meta/regions.json')
    def records = new JsonSlurper().parse(f)
    return records
}
```

O Nextflow usará isso como uma função personalizada dentro do escopo do fluxo de trabalho.

!!! dica

```
You will learn more about module files later in the [Modularization section](../modules/) of this tutorial.
```
