<!--- cSpell:enable --->

# Criando abstrações de hardware

> [!NOTE]
> :brain:

O desenvolvimento em sistemas embarcados reais pode ser complexo. Uma abordagem para reduzir essa complexidade é tentar desenvolver o máximo possível do firmware no PC, deixando apenas aspectos mais críticos de tempo real ou hardwares mais complexos de serem emulados no PC como foco no desenvolvimento posterior, direto na plataforma embarcada.

## Desenvolva no PC o quanto puder

Em sistemas embarcados é comum ficarmos escravos do hardware que estamos usando. Normalmente existem muitos periféricos de hardware que tendem a dificultar a tarefa de realizar parte do desenvolvimento no PC. Aspectos temporais podem tornar isso ainda mais complexo, ao exigir que eventos sejam sincronizados de forma precisa. Além disso, a ausência de uma placa pode atrasar o desenvolvimento caso se decida fazer tudo diretamente no hardware final.

Enquanto isso possa realmente justificar alguns casos de desenvolvimento, a boa notícia é que muita coisa pode ser feita sem que se tenha um hardware em mãos, bastando usar boas técnicas de desenvolvimento, criando abstrações de hardware capazes de permitir o desenvolvimento em outro ambiente. Com tudo desenvolvido, o trabalho de porte e verificação de funcionamento na plataforma final de hardware se torna mais simples e, principalmente, reduz o tempo de desenvolvimento. Se preferir, entenda como redução de custo do projeto.

## Hardware Abstraction Layer (HAL)

Os fabricantes de microcontroladores costumam fornecer suas implementações (ou drivers) para os periféricos disponíveis no chip. Especialmente para microcontroladores baseados em núcleos de processamento Cortex M, a arquitetura de software conta com pelo menos dois níveis de abstração. A primeira é a abstração do núcleo de processamento, fornecida pela ARM e conhecida como [CMSIS](https://github.com/ARM-software/CMSIS_6) (Common Microcontroller Software Interface Standard). É comum o fabricante que emprega núcleos ARM escrever a sua biblioteca de drivers para periféricos próprios com o suporte dessa abstração. 

Já o usuário final (nosso caso), usando ferramentas da ST, tem a possibilidade de usar essa camada de abstração de hardware da ST, comumente fornecida em dois níveis diferentes:

- **HAL** (High Level Hardware Abstraction Layer): abstração de alto nível, com uso facilitado para o usuário e requerendo pouco conhecimento dos periféricos e registros da ST
- **LL** (Low Level Abstraction Layer): abstração de nível mais baixo, normalmente com macros e funções que fazem uso direto de registros dos periféricos. Requer um conhecimento bem maior dos periféricos da ST mas gera um código menor e mais rápido.

Para quem está acostumado a usar o HAL da ST, sabe que basta incluir o arquivo de inclusão `main.h` para ter acesso a todos os periféricos e funções da biblioteca. Infelizmente, essa inclusão irá condenar o seu projeto a uma eterna dependência do HAL da ST. Se pretende validar seu código antes no PC ou ter um projeto mais portável entre fabricantes, é determinante criar o seu nível abstração, sem qualquer dependência de registros ou periféricos específicos. Sim, dá um pouco de trabalho, perde-se um pouco de desempenho, mas pode reduzir o tempo de desenvolvimento (ou custo) final.

```               
  ┌────────────────────────┐     
  │        CÓDIGO          │     
  │        PORTÁVEL        │     
  ├────────────────────────┤     
  │ CAMADA DE ABSTRAÇÃO    │     
  ├────────────────────────┤      
  │    HAL DO FABRICANTE   │      
  ├────────┐               │     
  │ CMSIS  │               │     
  ├────────┘───────────────┤     
  │        HARDWARE        │     
  └────────────────────────┘                           
```

Como exemplo, vamos criar a seguir algumas abstrações de hardware para um microcontrolador, seguindo bons padrões de projeto e evitando qualquer dependência de hardware. A partir desses exemplos e suas implementações para diversos sistemas operacionais, será mais fácil criar outras.

Alguns pontos relevantes devem ser considerados nesse processo de abstração:

- Podemos ter várias instâncias de um mesmo hardware. Por exemplo, mais de uma porta serial ou SPI.
- Todo o processo de configuração precisa ser replicado na sua interface, sem depender de qualquer pré-definição fornecida pelo fabricante. 
- Como simular o comportamento de um periférico pode mudar significativamente. Por exemplo, uma memória flash pode ser simulada como um arquivo no PC. Isso permitiria criar e validar o que é escrito nela. Já um pino de IO pode ser um pouco mais complexo, principalmente se pretende simular alterações nas entradas desses pinos ou interrupções. Uma alternativa poderia ser dar suporte a leitura de comandos do console que pudessem configurar um valor de um pino. Outra ideia seria a criação de uma tarefa com suporte a uma biblioteca como ZeroMQ, capaz de receber comandos remotos via rede. Um pouco de criatividade pode ajudar nesse processo.
- Algumas interfaces são facilmente criadas com a técnica de ponteiros opacos, explicada posteriormente nessa seção. Isso é importante para uma separação completa entre definição e implementação.
- Apresentar todas as chamadas de função de um determinado módulo como driver, através de uma estrutura de ponteiros de função, é uma técnica interessante também. Isso permite acoplar dinamicamente o driver desejado, dando suporte para revisões diferentes ou implementações personalizadas.
- Se possível, é interessante evitar o uso de alocação dinâmica sempre que possível. Enquanto não seja um impeditivo no PC, levar essa estratégia pra a implementação embarcada pode não ser desejado em várias aplicações.

Como vai ser notado, a implementação para o microcontrolador acaba sendo até mesmo mais simples do que a implementação para o PC. No PC, a menos que se escreva um código todo em pooling, pode ser necessário lidar com threads, mutexes e outras estruturas de sincronização, além de arquivos e bibliotecas adicionais, a depender da sua escolha de implementação e sistema operacional. 

A seguir iremos tratar de dois pontos importantes: a utilização de uma espécie de _namespacing_ para evitar conflitos de nomes e a estrutura básica de abstração de hardware, com o uso de driver genéricos e ponteiros opacos.

### Namespaces

Quem trabalha com linguagens mais modernos do que a linguagem C está acostumado com o conceito de _namespacing_. A idéia é encapsular, aninhando definições em módulos relacionados, evitando conflitos de nomes ou definições. Infelizmente, esse conceito não existe em C. 

Uma forma de tentar imitar o comportamento do namespacing é forçar uma rígida estratégia de nomeação de arquivos e definições, empregando o underscore como separador (o "_"), na notação conhecida como "snake case". Sim, muitos desenvolvedores preferem a lógica do "pascal case", onde se omite o underscore e usa-se a primeira letra de cada palavra em maiúscula e o resto em minúsculas. O autor considera a forma snake case mais legível, dando a impressão de um espaço entre palavras e facilitando a leitura. Caso prefira outra, não tem problema: o importante é ter uma notação comum para todos os desenvolvedores do seu time.

Veja um exemplo de uso do conceito de _namespacing_ em C, onde o nome do arquivo e as definições são baseadas no nome do hardware e na função que está sendo implementada:

```C copy
#pragma once

typedef enum hal_gpio_pin_e
{
	HAL_GPIO_PIN_0 = 0,
	HAL_GPIO_PIN_1,
	HAL_GPIO_PIN_2,
    HAL_GPIO_MAX_PINS,
} hal_gpio_pin_t;

void hal_gpio_set(hal_gpio_pin_t pin, bool state);
bool hal_gpio_get(hal_gpio_pin_t pin);
void hal_gpio_toggle(hal_gpio_pin_t pin);
```

No arquivo apresentado, foram criados três níveis de hierarquia: `hal` para indicar que é um arquivo de interface de hardware, `gpio` que é uma abreviação para _general purpose input/output_ e depois funções ou definições relacionadas com pinos de I/O, como `set`, `get` e como `toggle`. O nome final, por exemplo `hal_gpio_set()` permite reduzir conflitos com várias outras partes do código e de forma hierárquica. Num primeiro nível, isola tudo que não tem relação com o hal proposto. Em um segundo nível, impede que uma possível função `open` de um adc (`hal_adc_open()`) colida com a função do GPIO. Confie em mim: apesar de um pouco pedante, é uma excelente prática. 

Além disso, algumas notações relacionadas a sufixos são também recomendadas para facilitar a identificação do tipo de dado utilizado. Por exemplo, sufixos comuns são:

- `_e`: para enumerações
- `_t`: para tipos definidos
- `_s`: para estruturas

Além de evitar conflitos de nomes, dão visibilidade imediata ao tipo de dado utilizado. E, antes que pergunte, não estamos incentivando o uso de [notação Húngara](https://en.wikipedia.org/wiki/Hungarian_notation). Entendemos que é preferível padronizar tudo com o que já se tem na própria linguagem via `stdint.h`, `stdbool.h` e `stddef.h`.

Outros padrões de nomenclatura são recomendados:

- **Nomes de arquivos**: devem ser escritos em letras minúsculas, com palavras separadas por underscore. Em especial importante para evitar problemas em sistemas de arquivos que diferenciam maiúsculas de minúsculas, como o Linux e MacOS.
- **definições de macros**: devem ser escritas em letras maiúsculas, com palavras separadas por underscore.
- **Nomes de funções**: devem ser escritas em letras minúsculas, com a parte inicial relacionada ao namespace, seguidas de um substantivo e terminando com um verbo que indica a ação realizada. Por exemplo, `hal_gpio_set()`, `hal_cpu_interrupt_disable()`, `hal_cpu_watchdog_refresh()` e `hal_cpu_random_seed_get()`. Isso ajuda a identificar rapidamente o que a função faz e qual o seu contexto, deixando o código mais legível e fácil de entender.

### Exemplo de abstração da CPU

Com isso em mente, a proposta é desenvolver um modelo de abstração inicial com o mínimo necessário para operação. Você pode extender esse arquivo depois, com mais funcionalidades. Como exemplo, vamos criar uma abstração para a CPU do microcontrolador, com funções básicas de controle de interrupção, controle de watchdog, reset, entre outras operações comuns.

https://github.com/marcelobarrosufu/fwdev/blob/08a71f1459560b3683de074d63580a8db6cab999/source/hal/hal_cpu.h#L1-L51

Algumas explicações são importantes. Longe de serem apenas manias do autor, elas refletem anos de desenvolvimento e estudo, criando interfaces que funcionam, geram poucos erros e são fáceis de serem portadas e mantidas.

A estrutura `hal_cpu_dev_s` é o coração desse arquivo. Ela define um driver genérico para a CPU através de um conjunto de ponteiros de função:

```C copy
typedef struct hal_cpu_driver_s 
 { 
     void (*init)(void); 
     void (*deinit)(void); 
     void (*reset)(void); 
     void (*watchdog_refresh)(void); 
     void (*id_get)(uint8_t id[HAL_CPU_ID_SIZE]); 
     uint32_t (*random_seed_get)(void); 
     uint32_t (*critical_section_enter)(hal_cpu_cs_level_t level); 
     void (*critical_section_leave)(uint32_t last_level); 
     void (*low_power_enter)(void); 
     void (*sleep_ms)(uint32_t tmr_ms); 
     uint32_t (*time_get_ms)(void); 
     uint32_t (*time_elapsed_get_ms)(uint32_t tmr_old_ms); 
 } hal_cpu_driver_t; 
 ```

Cada função é um método que pode ser implementado de forma diferente, dependendo do porte desejado. Por exemplo, a função `hal_cpu_interrupt_disable()` pode ser implementada de forma diferente para um STM32F4, Linux ou Windows, mas a interface permanece a mesma. Veremos uma implementação dessa estrutura mais adiante.

Ao final do arquivo, existem protótipos de funções relacionadas a essa estrutura, para facilitar o uso do driver no código. Por exemplo, como no trecho a seguir:

```C copy
void hal_cpu_init(void);
void hal_cpu_deinit(void);
void hal_cpu_reset(void);
void hal_cpu_watchdog_refresh(void);
void hal_cpu_id_get(uint8_t id[HAL_CPU_ID_SIZE]);
````

Essas funções são chamadas diretamente no código, sem a necessidade de acessar a estrutura `hal_cpu_driver_t` diretamente. Isso facilita o uso do driver e torna o código mais legível.

Na abstração proposta, mais dois arquivos são importantes. O primeiro é o arquivo [hal.h](https://github.com/marcelobarrosufu/fwdev/blob/f2b6c7ec997e6cf6f05c5cb11786ed2dff5d01f7/source/hal/hal.h), que indica que deve existir, em algum lugar, uma implementação do driver da CPU. Isso pode ser visto através da seguinte linha, que é uma "promessa" de implementação do driver pelo programador para o linker:

```C copy
extern hal_cpu_driver_t HAL_CPU_DRIVER;
```

O segundo arquivo é o [hal_cpu.c](https://github.com/marcelobarrosufu/fwdev/blob/f2b6c7ec997e6cf6f05c5cb11786ed2dff5d01f7/source/hal/hal_cpu.c). Esse arquivo é o ponto de entrada para as chamadas relacionadas à implementação do driver, usando a promessa feita. Ele pode ser usado também para implementar ações que são portáveis já, evitando levar esse conteúdo para um driver. O início do arquivo é o seguinte:

```C copy
#include "hal.h"

static hal_cpu_driver_t *drv = &HAL_CPU_DRIVER;

void hal_cpu_init(void)
{
    drv->init();
}

void hal_cpu_deinit(void)
{
    drv->deinit();
}

void hal_cpu_reset(void)
{
    drv->reset();
}

void hal_cpu_watchdog_refresh(void)
{
    drv->watchdog_refresh();
}

void hal_cpu_id_get(uint8_t id[HAL_CPU_ID_SIZE])
{
    drv->id_get(id);
}

uint32_t hal_cpu_random_seed_get(void)
{
    return drv->random_seed_get();
}

// ...
````

Nesse arquivo é criado um ponteiro para a implementação do driver, usado nas funções disponibilizadas na estrutura `hal_cpu_driver_t`. Enquanto represente novamente um ponto onde se perde um pouco de desempenho, a abstração e portabilidade são favorecidas. Vale lembrar que, a depender do nível de otimização do compilador, essa perda pode ser quase nula.

O que falta agora é criar a implementação do driver da CPU, que será feita de forma diferente para cada porte. O trecho a seguir mostra como isso poderia ser feito para um microcontrolador STM32F4, por exemplo. A implementação não foi colocada completa aqui, mas você pode ver o exemplo completo no repositório do projeto, no arquivo [port_cpu.c](https://github.com/marcelobarrosufu/fwdev/blob/f2b6c7ec997e6cf6f05c5cb11786ed2dff5d01f7/source/port/stm32/port_cpu.c). Estamos também assumindo que o projeto foi criado pelo STM32CubeIDE usando os drivers de alto nível da ST (HAL).

```C copy
#include "main.h"
#include "hal.h"

extern RNG_HandleTypeDef hrng;

static void port_cpu_init(void)
{
#if HAL_DEBUG_IN_SLEEP_MODE == 1
	LL_DBGMCU_EnableDBGStopMode();
	LL_DBGMCU_EnableDBGStandbyMode();
#endif
}

static void port_cpu_deinit(void)
{
}

static void port_cpu_reset(void)
{
    NVIC_SystemReset();
}

static void port_cpu_watchdog_refresh(void)
{
#if HAL_WDG_ENABLED == 1
    LL_IWDG_ReloadCounter(IWDG);
#endif
}

// more functions...

hal_cpu_driver_t HAL_CPU_DRIVER =
{
    .init = port_cpu_init,
    .deinit = port_cpu_deinit,
    .reset = port_cpu_reset,
    .watchdog_refresh = port_cpu_watchdog_refresh,
    // more function pointer initializations ...
};
````

Ao final do arquivo, existe finalmente a realização do driver, na declaração `hal_cpu_driver_t HAL_CPU_DRIVER = ...`, onde os ponteiros de função da estrutura `hal_cpu_driver_t` são preenchidos com as funções implementadas. Um cuidado adicional é definir todas as funções como `static`, para que não sejam visíveis fora do arquivo, evitando possíveis conflitos de nomes com outras realizações dessa interface. Note que a inclusão do arquivo `main.h` da ST traz todas as definições necessárias para o funcionamento do driver, como a definição do `RNG_HandleTypeDef` e outras constantes, tornando esse arquivo totalmente dependente de plataforma, como esperado.

Com isso, fica claro a forma básica de abstração de hardware que será usada daqui pra diante. No entanto, para alguns dispositivos que permitam múltiplas instâncias, como uma porta serial ou SPI, é necessário um pouco mais de trabalho, com o emprego da técnica de ponteiros opacos. Vamos ver como isso é feito a seguir. Ah, e se você ficou curioso com as funções relacionada a interrupção, recomendo ler a seção sobre [interrupções no Cortex M](./interrupts.md).

### Ponteiros opacos

Para discutir o conceito de ponteiro opaco, é interessante usar outro exemplo de abstração de hardware, dessa vez relacionado a uma porta serial. A ideia é criar uma interface que permita o uso de diversas portas seriais, sem depender de qualquer implementação específica. Um dos problemas está justamente relacionado a estruturas de dados de controle de uma porta serial, que podem variar bastante entre implementações. No Windows, por exemplo, o dispositivo serial é tratado como um arquivo arquivo via funções como CreateFile(), ReadFile() e WriteFile(), usadas através de um manipulador de arquivos (HANDLE). A configuração dos parâmetros da serial exige uma estrutura do tipo `DCB` (Device Control Block), que é bem diferente da estrutura usada no Linux, por exemplo. Todos esses dados precisar estar ocultos do usuário e não devem ser expostos na interface de uso da porta serial.



Um outro ponto chama a atenção no arquivo de inclusão: a declaração do tipo de dados personalizado `hal_uart_dev_t`. Ele é declarado como um ponteiro para uma estrutura do tipo `struct *hal_uart_dev_s` mas note que não existe nenhuma definição do conteúdo da estrutura. Essa é uma técnica conhecida como ponteiro opaco, onde é possível declarar um ponteiro para uma estrutura sem se conhecer o conteúdo da mesma. Como será visto posteriormente, a estrutura `struct hal_uart_dev_s` será declarada apenas na realização da implementação da serial, no arquivo `hal_uart.c`. Com isso, nenhum detalhe é relevado sobre o dispositivo e a estrutura pode ser personalizada, a depender do porte desejado.

Para dar vida a nossa implementação, vamos apresentar (só dessa vez!) quatro portes diferentes: Win32, Linux, MacOS e STM32L411 (BlackPill). Assim você vai poder ver claramente as diferenças na realização da implementação. 

### Implementação para Win32


### Implementação para Linux


### Implementação para MacOS

### Implementação para STM32L411

```C
#include "main.h"
#include "hal_uart.h"

#define PORT_SER_BUFFER_SIZE 128

```

## Compilando para uma plataforma específica

Nesse ponto, existem diversos caminhos que podem ser tomados. Vai depender muito do tipo de ferramenta utilizada no build system. As IDEs, em geral, tornam isso mais complicado do que o uso de arquivos de Makefile ou sistemas baseados em cmake. É preciso que se garanta que apenas a realização desejada seja de fato compilada, sem nenhuma definição relacionada a outra plataforma sendo incluída.

Para facilitar a compatibilização com build systems baseados em IDE, como o STM32CubeIDE, vamos adotar a abordagem de definir o porte desejado através de uma definição do pré-processador. Assim é simples tratar via Makefile, onde é possível passar o porte na linha de comando, com algo como `-DHAL_PORT=STM32L4` ou criar a definição `HAL_PORT=STM32L4` manualmente dentro das opções da IDE.

Também iremos organizar todos os portes dentro de um mesmo diretório e criar um arquivo geral que irá selecionar a implementação desejada. 

```bash
source/hal/hal_uart.h
source/hal/hal_uart.c
source/hal/win32/port_ser.c
source/hal/linux/port_ser.c
source/hal/macos/port_ser.c
source/hal/stm32l4/port_ser.c
```

O arquivo `hal_uart.c` irá decidir qual versão do porte compilar através da definição `HAL_PORT`. Uma possível implementação seria:

```C
#if HAL_PORT == "WIN32"
    #include "./win32/port_ser.c"
#elif HAL_PORT == "LINUX"
    #include "./linux/port_ser.c"
#elif HAL_PORT == "MACOS"
    #include "./macos/port_ser.c"
#elif HAL_PORT == "STM32L4"
    #include "./stm32l4/port_ser.c"
#else
    #error Port not defined
#endif
```

Nesse ponto já é possível apresentar um exemplo completo mostrando o uso da porta serial. Vamos utilizar um Makefile mas lembre-se que o mesmo poderia ser feito na IDE, bastando que exista uma definição adequada para `HAL_PORT`.

https://github.com/marcelobarrosufu/fwdev/blob/ff0fde98f0564b7762885dedb0cd874bd1858465/test/hal/utl_dbg/main.c#L1-L22

Para compilação, use o Makefile abaixo:

```make

```

E compile com a seguinte chamada (substitua adequadamente para a sua plataforma)

```
make -DHAL_PORT=win32
```

Os projetos STM32 também pode ser compilados via Makefile, caso prefira. O inconveniente é ter que fazer a compilação manualmente antes de carregar o programa via IDE para depurar. 

## Verificação formal

Uma vez que seu código roda no PC, existem diversas ferramentas que estão à sua disposição para validação e testes. Para verificação de ponteiros e uso de buffers,alocação de memória, etc, alternativas como Valgrind, memcheck, cppcheck e drmemory podem ser úteis. Para desenvolvimento de testes automatizados, o módulo Unity pode ajudar.


## Wraping up

Apesar de requerer um trabalho adicional, os ganhos ao criar abstrações genéricas e portáveis são enormes. E, uma vez desenvolvidas, podem ser reusadas em outros projetos. Considere isso ao iniciar um novo trabalho !

<!-- Por exemplo, pode adicionar callbacks para recepção por interrupção ou mesmo DMA, callbacks para indicação de fim de transmissão, etc. O arquivo de inclusão está a seguir, denominado de `hal_uart.h`. As explicações serão dadas posteriormente.

https://github.com/marcelobarrosufu/fwdev/blob/b156bcffe07d7005796ca77682ff5ff5acee0ab9/source/port/hal_uart.h#L1-L38

-->