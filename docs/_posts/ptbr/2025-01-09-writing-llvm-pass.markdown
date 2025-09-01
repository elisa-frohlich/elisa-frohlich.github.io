---
layout: post
title:  "Escrevendo um Passe LLVM"
date:   2025-08-09 13:00:00 -0300
category: LLVM
lang: ptbr
---

### Introdução

Após um certo tempo, hora de voltar a escrever alguma coisa aqui. Dessa vez, vou sair um pouco da teoria e ir para um lado prático (não que as próximas postagens daquele assunto não tenham um lado prático).

Nessa postagem, irei explicar, ou tentar explicar, como construir um passe para a infraestrutura LLVM. Tá, eu sei, existem diversos tutoriais que ensinam a escrever, compilar e executar um passe LLVM, mas será que fará mal escrever mais um? Acho que excesso de opções não é de fato um problema aqui, além do mais, quando eu for falar sobre aplicações de passes LLVM mais pra frente, não terei que sair caçando algum blog em português que explique de uma forma que eu ache satisfatória, que as coisas funcionem e esse tipo de coisa. Note que essa postagem será inicialmente escrita só em português (a versão em inglês vai sair quando eu tiver paciência pra traduzir isso aqui). Além disso, eu gostaria muito de poder só indicar o [tutorial](https://llvm.org/docs/WritingAnLLVMNewPMPass.html) da LLVM sobre o assunto mas, infelizmente, eu não achei a documentação da LLVM traduzida para português.

Dessa forma, nessa postagem eu irei cobrir conteúdos bem básicos, assumindo que meus textos podem ser lidos por pessoas que não entendem quase nada de compiladores, porém têm um grande interesse na área. Porém, é esperado que a pessoa entenda um pouco de C++ para entender a parte da implementação.

Por fim, um pequeno resumo do que será abordado:

- A infraestrutura LLVM;
- O que são passes LLVM;
- Como escrever um passe;
- Como compilar e executar um passe.

### LLVM

A infraestrutura LLVM é um conjunto de ferramentas que permitem construir (e usar) diversos compiladores. Com ela, você pode construir um compilador muito eficiente para qualquer linguagem de programação sem **muito** esforço.

Dentre as principais ferramentas, podemos destacar:
- **O núcleo da LLVM**: que permite uma série de otimizações no programa fonte, e também permite a geração de código de máquina para diversas arquiteturas de CPU;
- **O compilador Clang**, que é o compilador "nativo" da LLVM para C/C++; e
- **As bibliotecas padrões libc/libc++**, que é uma implementação muito otimizada da biblioteca padrão de C/C++.

Claro, existem outras ferramentas incríveis, e você pode dar uma olhada nelas no [site da LLVM](https://llvm.org)

Porém, o que mais nos interessa agora é a representação intermediária da LLVM, a LLVM IR. Essa representação fica no meio termo entre uma linguagem de programação e uma linguagem assembly, e é muito útil para executar análises e transformações do código.

Quando se pensa no desenvolvimento de um compilador para uma linguagem  de programação qualquer utilizando a LLVM, normalmente o processo envolve implementar o _front end_ do compilador para traduzir o código fonte para LLVM IR, e a partir daí usar o conjunto de ferramentas da LLVM para fazer umas magias no código e enfim gerar o código de máquina que será executado.

### Passes LLVM

A infraestrutura LLVM é altamente modular. Parte da utilização dessa modularidade é feita a partir dos passes LLVM. Eles são _plugins_ individuais que executam determinadas tarefas no código que está sendo compilado.

Para executar um passe LLVM, você precisa executar a ferramenta `opt`, que faz parte do arcabouço da LLVM. Um passe LLVM recebe um programa como entrada e percorre (daí o nome passe) todo o programa, podendo coletar informações acerca dele ou modificá-lo.

Por conta disso, um passe pode ser dividido em 3 categorias:

- Análise: passes que extraem informações de programas e disponibilizam elas para serem utilizadas por outros passes;
- Transformação: passes que modificam o programa de alguma forma;
- Utilidade: passes que fornecem alguma utilidade na visualização de informações do programa, mas que não se encaixam nem como passe de análise, nem como passe de transformação.

Os passes podem operar sob diferentes níveis de hierarquia, porém normalmente trabalhamos com 2 tipos de hierarquias:

- Módulos: a grosso modo, representam um arquivo fonte do programa. Um módulo contêm funções, variáveis globais e metadados deste arquivo do programa;
- Funções: representam as funções do programa, sendo que uma função contém uma assinatura (nome, parâmetros, tipo de retorno), blocos básicos (compostos por várias instruções), atributos e metadados.

A partir disso, temos dois tipos de passes: passes de função (quando o passe percorre as funções do programa uma a uma) e passes de módulo (quando o passe percorre os módulos um a um).

Por fim, um pequeno parênteses histórico: a LLVM possui dois gerenciadores de passes. O antigo é chamado de _Legacy Pass Manager_ (ou Gerenciador de Passe Legado), que foi descontinuado, e o novo é chamado de _New Pass Manager_ (Gerenciador de Passe Novo) ou NPM.

Todas as minhas postagens em que eu falar sobre passes LLVM, incluindo essa, sempre estará se referindo ao NPM.

### Desenvolvendo um Passe

Além dos gerenciadores de passes, também existem duas formas diferentes de desenvolver os passes pra LLVM: "dentro da árvore" e "fora da árvore".

Confesso que nunca escrevi um passe "dentro da árvore", mas a ideia é que você coloca o código do seu passe dentro da pasta da LLVM (por isso dentro da árvore) e, para compilar o passe, você recompila o `opt`.

Então vamos nos contentar com a escrita de passes "fora da árvore", porque é a forma que eu conheço (e, até onde eu vi, considero muito mais elegante). A questão aqui é que você pode desenvolver esse passe no diretório que você quiser, apenas tendo que escrever um pouco de código a mais (que praticamente não muda entre passes e, quando muda, muda pouco).

A estrutura do passe vai ter uma organização de arquivos bem simples de ser utilizada:

```
meu_passe/
    |
    +--include/
    |   |
    |   +--MeuPasse.h
    |
    +--lib/
    |   |
    |   +--MeuPasse.cpp
    |   +--MeuPassePlugin.cpp
    |
    +--CMakeLists.txt
```

Note que, para escrever o passe, iremos 
usar C++ para a implementação e o CMake como gerenciador de compilação. Se você não sabe usar CMake, não tem problemas, porque vou mostrar um jeito meio padrão de utilizar ele para compilar passes LLVM.

Nessa estrutura, temos:
- `MeuPasse.h`: nosso arquivo de cabeçalho;
- `MeuPasse.cpp`: nosso arquivo de implementação; e
- `MeuPassePlugin.cpp`: onde registraremos nosso passe no sistema de _plugins_ da LLVM, para que a aplicação `opt` consiga encontrá-lo.

Para nosso primeiro passe, vamos desenvolver um simples passe de análise: para cada função, vamos imprimir o nome dessa função e a quantidade de blocos básicos presentes nela. Note que, para isso, podemos desenvolver um passe de função, pois podemos analisar cada função independentemente.

Vou explicar agora detalhadamente como funciona o desenvolvimento de cada arquivo, começando pelo

#### MeuPasse.h

O código desse cabeçalho consiste na declaração de uma classe que define o passe. Isto porque cada passe na LLVM é representado por uma classe em C++.

```cpp
#ifndef MEU_PASSE_H
#define MEU_PASSE_H

#include "llvm/IR/PassManager.h"

namespace llvm {

class MeuPasse : public PassInfoMixin<MeuPasse> {
public:
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM);
};

}

#endif // MEU_PASSE_H
```

Vamos comentar alguns detalhes mais a fundo.

A classe `MeuPasse` herda da classe `PassInfoMixin`, que é uma exemplo de uma classe modelo [CRTP](https://en.cppreference.com/w/cpp/language/crtp.html) (_Curiously Recurrent Template Pattern_). Esse modelo de classe parece estranho a princípio, mas é um truque muito interessante que permite a configuração automática de um conjunto de informações necessários para a LLVM entender o seu passe.

Essa classe declara apenas uma função: `run`, que é responsável pela execução do passe (quase que equivalente a uma função `main` de um programa qualquer).

Essa função possui a seguinte assinatura:
- Recebe dois parâmetros:
    1. `Function &F`: Uma referência para a função em LLVM IR que o passe está analisando
    2. `FunctionAnalysisManager &FAM`: Uma referência para um objeto que gerencia a execução de diferentes análises para um objeto do tipo `Function`.
- Retorna um objeto do tipo `PreservedAnalyses`, que será explicado mais pra frente.

Por fim, note que estamos declarando a classe dentro do `namespace llvm`. Fazemos isso porque o passe precisa ser declarado como uma classe dentro deste namespace para ser devidamente integrado à LLVM (portanto ficando algo como `llvm::MeuPasse` quando "visto de fora").

Agora, tendo explicado a declaração da classe, vamos ver a implementação da função `run`.

#### MeuPasse.cpp

Vamos lembrar o que queremos fazer nesse passe. Para cada função, queremos imprimir:
- O nome da função; e
- A quantidade de blocos básicos dela.

Para nossa sorte, todas essas informações podem ser facilmente acessadas e, dessa forma, o código se torna simples:

```cpp
#include "MeuPasse.h"

using namespace llvm;

PreservedAnalyses MeuPasse::run(Function &F,
                                FunctionAnalysisManager &FAM) {
    outs() << F.getName() << " " << F.size() << "\n";
    return PreservedAnalyses::all();
}
```

Nessa função, estamos utilizando a função `outs()`, que usa o formato de saída da LLVM. Esse formato é quase equivalente ao da função `std::cout` da biblioteca `iostream`, mas otimizado para lidar com os tipos da LLVM, sendo capaz de lidar com o tipo `StringRef`, que é o tipo de retorno da função `getName()`.

A método `getName()` da classe `Function` retorna o nome da função, enquanto que o método `size()` retorna o número de blocos básicos da função. 

Por fim, retornamos `PreservedAnalyses::all()`, então vamos explicar o que é o tipo `PreservedAnalyses`. Esse tipo representa um conjunto de análises que afirmamos que nosso passe preserva. Nesse contexto, a função `all()` indica que nosso passe garante que **todas** as análises são preservadas.

Num passe de análise, normalmente é garantido que todas as análises são preservadas, porém quando lidarmos com passes de transformação, existe sim a possibilidade de algumas análises não serem preservadas.


> Extra: para ir além do básico do básico, vou mostrar como iterar pelos blocos básicos da função para obter o número de instruções presentes na função:
> ```cpp
> int num_instr = 0;
> for (BasicBlock &BB : F) {
>     num_instr += BB.size();
> }
> outs() << num_instr << "\n";
> ```


#### MeuPassePlugin.cpp

Agora, vamos "registrar" o passe que acabamos de implementar, afinal queremos ser capazes de executar ele. Para isso, vamos implementar um monte de código padrão que eu vou _tentar_ explicar pra que serve cada coisa. Vamos começar com as importações:

```cpp
#include "MeuPasse.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"

using namespace llvm;
```

Que não têm nada de especial. Agora, vamos para a função que registra o _pipeline_ do passe:

```cpp
bool registerPipeline(StringRef Name, FunctionPassManager &FPM,
                      ArrayRef<PassBuilder::PipelineElement>) {
    if (Name == "meu-passe") {
        FPM.addPass(MeuPasse());
        return true;
    }
    return false;
}
```

Nessa função, estamos dizendo que, quando o `opt` for executado pedindo para executar o passe `meu-passe`, será registrado um pipeline composto pelo passe `MeuPasse()`. Note que `MeuPasse()` é a função construtora da classe `MeuPasse` que definimos anteriormente. O valor de retorno da função indica se o passe `meu-passe` foi encontrado ou não. Além disso, é possível definir um pipeline modificado com múltiplos passes, os quais serão executados na ordem em que forem adicionados.

```cpp
PassPluginLibraryInfo getMeuPasse() {
    return {
        LLVM_PLUGIN_API_VERSION, "meu-passe",
        LLVM_VERSION_STRING, [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(registerPipeline);
        }
    };
}
```

Aqui, estamos definindo uma função que diz como o passe deve ser carregado. O tipo `PassPluginLibraryInfo` é, nesse caso, uma struct que contém a versão da API da LLVM do plugin, o nome do passe, a versão da LLVM, e uma função que registra a pipeline do passe (nesse caso, a função que implementamos acima).

Por fim, indicamos como inicializar o plugin (que diz como o passe será carregado) com:

```cpp
extern "C" LLVM_ATTRIBUTE_WEAK PassPluginLibraryInfo
llvmGetPassPluginInfo() {
    return getMeuPasse();
}
```

Agora, vamos ver como compilar isso tudo.

#### CMakeLists.txt

Vou supor que você já tenha o LLVM instalado e, mais especificamente, tenha compilado e instalado ele usando CMake.

Caso não tenha (e esteja disposto a passar muito tempo vendo seu computador fritar) veja o [Apêndice A - Compilando a LLVM usando CMake](#apêndice-a---compilando-a-llvm-usando-cmake).

Vamos começar o arquivo CMake com duas linhas obrigatórias, onde definimos a versão mínima do CMake necessária para compilar o projeto e o nome do projeto:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MeuPasseLegal)
```

Agora vem um monte de código pra dizer onde encontrar os cabeçalhos da LLVM e configurar algumas _flags_ de compilação. Inclusive, isso é meio padrão pra qualquer passe fora da árvore:

```cmake
set(CMAKE_CXX_STANDARD 17 CACHE STRING "")

set(LLVM_INSTALL_DIR "" CACHE PATH "LLVM installation directory")
set(LLVM_CMAKE_CONFIG_DIR "" "${LLVM_INSTALL_DIR}/lib/cmake/llvm/")
list(APPEND CMAKE_PREFIX_PATH "${LLVM_CMAKE_CONFIG_DIR}")

find_package(LLVM REQUIRED CONFIG)

include_directories(${LLVM_INCLUDE_DIRS})

if(NOT LLVM_ENABLE_RTTI)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
endif()

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
```

Depois, vamos mostrar como o CMake deve compilar nosso código. Primeiro, definimos uma biblioteca de nome `MeuPasse` do tipo `MODULE`, com as implementações do passe:

```cmake
add_library(MeuPasse MODULE
    lib/MeuPasse.cpp
    lib/MeuPassePlugin.cpp)
```

Por fim, dizemos onde a biblioteca `MeuPasse` deve buscar os arquivos de cabeçalho:

```cmake
target_include_directories(MeuPasse PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/include")
```

Dessa forma, permitimos que o arquivo `MeuPasse.cpp` enxergue o cabeçalho `MeuPasse.h` sem precisar descrever o caminho exato para ele (que, relativo a `MeuPasse.cpp`, seria `../include/MeuPasse.h`). Lembre que, quando incluímos este cabeçalho neste arquivo, o fizemos apenas com `#include "MeuPasse.h"`.

Pronto, agora nós temos um passe implementado, com instruções de como registrar ele no pipeline de passes da LLVM, e um arquivo CMake configurado para compilar o passe. Vamos enfim testar se isso tudo está funcionando.

### Testando um Passe

Vamos começar pelo mais importante, que é compilar o passe. Para isso, vamos utilizar o CMake para gerar os arquivos de configuração numa pasta chamada `build`:

```bash
mkdir build
cd build
cmake ..
```

Agora, vamos compilar com _Unix Makefiles_ executando:

```bash
make
```

Se tudo der certo (e você não tiver mexido na variável `CMAKE_LIBRARY_OUTPUT_DIRECTORY`), o seu passe compilado estará no arquivo `lib/libMeuPasse.so` dentro da pasta `build`.

Agora queremos executar, certo? Bom, primeiro, precisamos de um código para isso. Lembre que fizemos um passe para analisar **códigos**. Vamos usar o código a seguir (que vou nomeá-lo preguiçosamente de `a.c`), com uma função recursiva pra calcular o n-ésimo termo da sequência de Fibonacci recursivamente:

```c
#include <stdio.h>

int f(int x) {
    if (x < 2) return x;
    return f(x-1)+f(x-2);
}

int main() {
    printf("%d\n", f(5));
    return 0;
}
```

Vamos compilar este código para LLVM IR usando o clang com os seguintes parâmetros:

```bash
clang a.c -Xclang -disable-O0-optnone -S -emit-llvm -o a.ll
```

Os parâmetros `-Xclang -disable-O0-optnone` vão impedir que a LLVM marque as funções deste código como não otimizáveis, o que impediria nosso passe de executar nelas. Já os parâmetros `-S -emit-llvm` fazem com que o clang gere o código em LLVM IR. O código gerado deve ficar algo como:

```llvm
...
define dso_local i32 @f(i32 noundef %0) #0 {
  ...
  br i1 %5, label %6, label %8

6:  ; preds = %1
  ...
  br label %16

8:  ; preds = %1
  ...
  br label %16

16: ; preds = %8, %6
  %17 = load i32, ptr %2, align 4
  ret i32 %17
}

define dso_local i32 @main() #0 {
  ...
  ret i32 0
}

...
```

Perceba que a função `f` possui 4 blocos básicos (0, que é omitido, 6, 8 e 16) e a função `main` possui apenas 1 bloco básico (0, que é omitido). Agora, vamos executar nosso passe com o seguinte comando:

```bash
opt -disable-output -load-pass-plugin lib/libMeuPasse.so -passes="meu-passe" a.ll
```

Note que, como nosso passe não está incluso no conjunto de passes padrões da LLVM, precisamos carregar o arquivo compilado do nosso passe com `-load-pass-plugin`. Além disso, `-disable-output` fala para o `opt` não imprimir o arquivo transformado como um código binário. A saída do passe com a função `outs()` não é afetada por essa _flag_, por mais contraintuitivo que pareça.

Ao executar, a saída esperada é:
```
f 4
main 1
```

Nosso passe conseguiu analisar o programa, então parece que funcionou!

### Conclusão

Enfim, temos um passe da LLVM. Note que esse é um passe de análise muito simples, que não nos diz quase nada. De todo modo, a ideia era tentar explicar o conceito dos passes e como desenvolver um, mostrando como declarar ele, implementar as funções necessárias, integrar ele à LLVM e configurar o CMake para compilar o passe.

Mas isso é só o começo. Na próxima postagem sobre construção de passes LLVM, vou tentar explicar como fazer um passe de transformação de código que faz algo um pouco mais útil (Spoiler: vamos contar quantas vezes cada aresta do CFG é atravessada).

Caso tenha dúvidas e queira conversar sobre o assunto, lembre que meu email está no [sobre]({{ "/about" | relative_url }}). Até (espero) breve.

### Referências

- https://llvm.org/
- https://llvm.org/doxygen/


### Apêndice A - Compilando a LLVM usando CMake

Aqui eu vou mostrar como baixar, configurar, compilar e instalar a LLVM usando CMake no Linux. Não faço ideia de como isso funciona no Windows ou no Mac (apesar que esse é Unix-based pelo menos).

Além disso, estarei mostrando aqui como instalar a LLVM 18.1.8, que é meio antiga (a versão mais atualizada, que não está em pre-release, é a 20.1.8), mas é a versão que eu uso no mestrado.

Começamos baixando o código fonte da LLVM com

```bash
git clone https://github.com/llvm/llvm-project/ --depth 1 --branch=release/18.x
```

Recomendo utilizar `--depth 1` se você não pretende navegar nas diferentes branches e releases da LLVM.

Agora, dentro da pasta `llvm-project`, configuramos o projeto com (assumo que você possua `gcc` e `g++` instalados, e esteja numa arquitetura X86):

```bash
mkdir build
cd build
cmake -G "Unix Makefiles" ../llvm \\
    -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_BUILD_TYPE=Release \\
    -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ \\
    -DCMAKE_ASM_COMPILER=gcc \\
    -DLLVM_ENABLE_PROJECTS="clang;lld" \\
    -DCMAKE_INSTALL_PREFIX=/usr/local
```

Opcional: se você tiver o sistema de compilação [Ninja](https://ninja-build.org/) (recomendo muito), você pode configurar pra compilar a LLVM usando ele com `-G Ninja`.

Com isso, o CMake irá configurar os arquivos de compilação. Agora, você pode compilar isso tudo usando:

```bash
make -jn
```

Aqui, troque `n` pela quantidade de núcleos da CPU que você quer usar. Se você estiver usando Ninja, você pode executar apenas `ninja` que ele irá compilar pra você usando todos os núcleos do seu processador (caso você esteja vendo sua memória RAM sumir, vale a dica do `-jn` também pra limitar a quantidade de núcleos compilando).

> Pode acontecer da compilação falhar. Minha recomendação é mandar compilar de novo quantas vezes forem necessárias. Já teve caso em que eu mandei compilar a LLVM com algo como:
> ```bash
> for i in 0..100; do ninja; done
> ```
> E não ironicamente isso funcionou, com ele terminando de compilar na tentativa 50 e alguma coisa. Isso acontece porque, como está sendo compilado em paralelo (e o projeto não é tão bem configurado), ele tenta compilar alguma biblioteca que depende de outra biblioteca que ainda está sendo compilada, aí obviamente dá erro.

Quando terminar de compilar (e vai demorar muito tempo, vai por mim), você pode instalar com

```bash
sudo make install
```

Ou `sudo ninja install`. Com a configuração do `DCMAKE_INSTALL_PREFIX=/usr/local`, esse comando irá inserir os binários em `/usr/local/bin`, os cabeçalhos das classes em `/usr/local/include` e as implementações das classes em `/usr/local/lib`.

Se quiser ver se deu tudo certo, basta rodar `clang --version` e ver se ele imprime algo como:

```bash
clang version 18.1.8 ...
...
```