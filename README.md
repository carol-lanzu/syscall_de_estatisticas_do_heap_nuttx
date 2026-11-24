# Implementação de Syscall para Estatísticas e Fragmentação do Heap no Apache NuttX

**Disciplina:** Sistemas Operacionais Embarcados (DEC7562)  
**Instituição:** Universidade Federal de Santa Catarina (UFSC)  
**Plataformas Alvo:** Espressif ESP32 (ESP32-DevKitC) / ESP32-S3 (ESP32-S3-DevKit)  
**Sistema Operacional:** Apache NuttX (v13.1.1+) — Modo FLAT (`CONFIG_BUILD_FLAT=y`)  

---

## 1. Descrição da Funcionalidade Implementada

O objetivo deste projeto é fornecer uma nova interface de serviço no kernel do Apache NuttX — `mm_get_custom_stats` — capaz de realizar a inspeção direta das estruturas internas do gerenciador de memória (*heap*) para extrair métricas precisas sobre consumo e **fragmentação externa**.

Diferente das rotinas convencionais como `mallinfo()`, que oferecem apenas um sumário simplificado, esta funcionalidade:
1. **Inspeciona diretamente as listas encadeadas de nós livres** (`mm_nodelist`), computando em tempo real o volume de blocos livres e identificando com exatidão o maior bloco contíguo disponível.
2. **Garante segurança contra concorrência**, utilizando o mecanismo de bloqueio interno (`mm_lock` e `mm_unlock`) para evitar condições de corrida com chamadas simultâneas de `malloc()` e `free()` vindas de outras tarefas.
3. **Valida parâmetros de entrada** conforme padrões POSIX/NuttX, retornando erro `-EINVAL` caso seja fornecido um ponteiro nulo (`NULL`), evitando falhas de segmentação no kernel.
4. **Disponibiliza métricas estruturadas** através da `struct mem_custom_stats_s`:
   - `total_heap`: Memória total sob controle do heap (em bytes).
   - `total_free`: Memória total livre disponível (soma de todos os nós livres).
   - `total_used`: Memória atualmente alocada (`total_heap - total_free`).
   - `largest_free_block`: Tamanho do maior fragmento contíguo livre (em bytes).
   - `free_chunks`: Quantidade de blocos livres fragmentados na lista.

---

## 2. Identificação dos Arquivos Modificados e Adicionados

As alterações foram distribuídas entre o núcleo do sistema operacional (`nuttx/`) e o diretório de aplicações em espaço de usuário (`apps/`), conforme detalhado a seguir:

```
nuttx/
├── include/nuttx/mm/mm.h       [MODIFICADO] Declaração da estrutura e protótipo da syscall
├── mm/mm_heap/
│   ├── Make.defs               [MODIFICADO] Registro do arquivo-fonte no build system do kernel
│   └── mm_get_stats.c          [ADICIONADO] Implementação da lógica de inspeção do heap
└── syscall/syscall.csv         [MODIFICADO] Registro formal na tabela de chamadas de sistema

apps/examples/mem_stats_test/
├── Kconfig                     [ADICIONADO] Menu de configuração no menuconfig do NuttX
├── Make.defs                   [ADICIONADO] Inclusão condicional da aplicação no build
├── Makefile                    [ADICIONADO] Diretivas de compilação da aplicação
└── mem_stats_test_main.c       [ADICIONADO] Roteiro dos 4 testes de validação no console serial
```

### Detalhamento das Modificações no Código do NuttX

#### 1. `nuttx/include/nuttx/mm/mm.h` (Interface Pública do Kernel)
* **Status:** Modificado.
* **Modificação:** Inclusão da definição da estrutura de estatísticas e da assinatura pública da função.

```c
/* Definição dos dados expostos pela chamada */
struct mem_custom_stats_s
{
  size_t total_heap;          /* Tamanho total gerenciado pelo heap (bytes) */
  size_t total_free;          /* Total de memória livre disponível (bytes) */
  size_t total_used;          /* Total de memória atualmente alocada (bytes) */
  size_t largest_free_block;  /* Maior bloco contíguo disponível (bytes) */
  uint32_t free_chunks;       /* Número de fragmentos livres na lista */
};

/* Protótipo da função */
int mm_get_custom_stats(FAR struct mem_custom_stats_s *stats);
```

#### 2. `nuttx/mm/mm_heap/mm_get_stats.c` (Implementação no Kernel)
* **Status:** Adicionado (arquivo novo).
* **Modificação:** Implementação da lógica interna de inspeção. O fluxo segue estritamente:
  1. Verificação de integridade: `if (stats == NULL) return -EINVAL;`
  2. Inicialização: `memset(stats, 0, sizeof(struct mem_custom_stats_s));`
  3. Obtenção do heap de usuário: `heap = USR_HEAP;`
  4. Esvaziamento de desalocações atrasadas: `mm_free_delaylist(heap);`
  5. Seção crítica: bloqueio via `mm_lock(heap);`
  6. Varredura da lista `mm_nodelist`: iteração partindo de `heap->mm_nodelist[0].flink`, filtrando nós sentinela (`node->size != 0`), somando `total_free`, computando `largest_free_block` e incrementando `free_chunks`.
  7. Finalização: cálculo de `total_used`, liberação via `mm_unlock(heap);` e retorno `OK` (`0`).

#### 3. `nuttx/mm/mm_heap/Make.defs` (Build do Heap)
* **Status:** Modificado.
* **Modificação:** Adicionado `mm_get_stats.c` à lista `CSRCS` sob a condição `CONFIG_MM_DEFAULT_MANAGER=y`:
```makefile
CSRCS += mm_get_stats.c
```

#### 4. `nuttx/syscall/syscall.csv` (Tabela de Syscalls)
* **Status:** Modificado.
* **Modificação:** Inserção em ordem alfabética da entrada de registro formal:
```csv
"mm_get_custom_stats","nuttx/mm/mm.h","","int","FAR struct mem_custom_stats_s *"
```

#### 5. Aplicação de Teste: `apps/examples/mem_stats_test/`
* **Status:** Adicionado.
* Foram criados os arquivos `Kconfig`, `Make.defs`, `Makefile` e `mem_stats_test_main.c` para registrar o comando `mem_stats_test` no shell interativo (NSH).

---

## 3. Explicação Resumida das Estruturas e Funções Internas Utilizadas

A implementação interage diretamente com o subsistema de alocação dinâmica do NuttX (`mm_heap`):

* **`struct mm_heap_s` (`mm/mm_heap/mm.h`):**  
  Estrutura descritora de um heap no NuttX. Dela são acessados:
  - `mm_heapsize`: Guarda o tamanho bruto total alocado para a região de memória.
  - `mm_lock`: Mutex do tipo `mutex_t` que protege a estrutura contra acessos concorrentes.
  - `mm_nodelist[MM_NNODES]`: Vetor de listas duplamente encadeadas organizadas em classes de tamanho (*bins*).
* **`struct mm_freenode_s` (`mm/mm_heap/mm.h`):**  
  Cabeçalho presente em cada pedaço livre de memória. Contém ponteiros para o nó seguinte (`flink`) e nó anterior (`blink`), além do campo `size` (tamanho do bloco e sinalizadores de alocação).
* **Nós Sentinela e Condição de Parada:**  
  As entradas em `heap->mm_nodelist[ndx]` atuam como nós sentinela de cabeçalho das classes de tamanho e possuem `node->size == 0`. Na nossa implementação, o loop percorre a lista encadeada e ignora expressamente nós com `node->size == 0`, garantindo que apenas blocos reais sejam somados e contabilizados em `free_chunks`.
* **Macro `MM_SIZEOF_NODE(node)`:**  
  Aplica a máscara de bits `(~MM_MASK_BIT)` sobre o campo `node->size` para extrair o tamanho real em bytes do bloco, descartando os bits de metadados (`MM_ALLOC_BIT` e `MM_PREVFREE_BIT`).
* **Macro `USR_HEAP` (`include/nuttx/mm/mm.h`):**  
  Resolve a referência para o heap padrão do sistema (`g_mmheap` em compilações FLAT).
* **`mm_lock(heap)` e `mm_unlock(heap)` (`mm/mm_heap/mm_lock.c`):**  
  Garantem a aquisição e liberação do mutex do heap, impedindo condições de corrida com `malloc` e `free` em execução por outras threads.
* **`mm_free_delaylist(heap)` (`mm/mm_heap/mm_malloc.c`):**  
  Garante que qualquer bloco cuja liberação tenha sido postergada seja devidamente retornado à lista livre antes da contagem das estatísticas.

---

## 4. Instruções para Compilação e Execução

### Pré-requisitos
* Toolchain GCC para Xtensa:
  * Para **ESP32 clássico**: `xtensa-esp32-elf-gcc` (ex: em `/opt/xtensa/xtensa-esp32-elf/bin`).
  * Para **ESP32-S3**: `xtensa-esp-elf-gcc` (ex: em `/home/carol-lanzu/toolchains/xtensa-esp-elf/bin`).
* Utilitário `esptool.py` (disponível no ambiente virtual Python em `.venv/bin/esptool.py`).
* Terminal serial `picocom` (ou `minicom`).

---

### Procedimento A: Compilação para ESP32 Clássico (ESP32-DevKitC)

1. **Configurar o alvo:**
   ```bash
   cd nuttx
   make distclean
   ./tools/configure.sh esp32-devkitc:nsh
   ```

2. **Habilitar a aplicação de teste:**
   ```bash
   sed -i 's/# CONFIG_EXAMPLES_MEM_STATS_TEST is not set/CONFIG_EXAMPLES_MEM_STATS_TEST=y/' .config
   export PATH=/opt/xtensa/xtensa-esp32-elf/bin:/home/carol-lanzu/nuttxspace/.venv/bin:$PATH
   make olddefconfig
   ```

3. **Compilar:**
   ```bash
   make -j$(nproc)
   ```

4. **Gravar na placa (*Flash*):**
   ```bash
   sudo chmod 666 /dev/ttyUSB0
   /home/carol-lanzu/nuttxspace/.venv/bin/esptool.py -c esp32 -p /dev/ttyUSB0 -b 921600 write_flash -fs detect -fm dio -ff 40m 0x1000 nuttx.bin
   ```

---

### Procedimento B: Compilação para ESP32-S3 (ESP32-S3-DevKit via porta USB ACM0)

1. **Configurar o alvo para USB nativa:**
   ```bash
   cd nuttx
   make distclean
   ./tools/configure.sh esp32s3-devkit:usbnsh
   ```

2. **Habilitar a aplicação de teste:**
   ```bash
   sed -i 's/# CONFIG_EXAMPLES_MEM_STATS_TEST is not set/CONFIG_EXAMPLES_MEM_STATS_TEST=y/' .config
   export PATH=/home/carol-lanzu/toolchains/xtensa-esp-elf/bin:/home/carol-lanzu/nuttxspace/.venv/bin:$PATH
   make olddefconfig
   ```

3. **Compilar:**
   ```bash
   make -j$(nproc)
   ```

4. **Gravar na placa (*Flash*):**
   ```bash
   sudo chmod 666 /dev/ttyACM0
   /home/carol-lanzu/nuttxspace/.venv/bin/esptool.py -c esp32s3 -p /dev/ttyACM0 -b 921600 write_flash -fs detect -fm dio -ff 40m 0x0000 nuttx.bin
   ```

---

### Execução dos Testes na Placa

1. **Conectar ao console serial:**
   * Para ESP32 clássico:
     ```bash
     picocom -b 115200 /dev/ttyUSB0
     ```
   * Para ESP32-S3 (porta ACM0):
     ```bash
     picocom -b 115200 /dev/ttyACM0
     ```

2. **Inicializar o shell:**
   Pressione a tecla `Enter` (ou o botão físico `RESET/EN` na placa). O prompt `nsh>` será exibido:
   ```text
   NuttShell (NSH) NuttX-13.1.1
   nsh>
   ```

3. **Disparar a aplicação de validação:**
   ```text
   nsh> mem_stats_test
   ```

---

## 5. Descrição dos Testes Realizados e Resultados Obtidos

A aplicação `mem_stats_test` executa 4 cenários sequenciais que validam os requisitos obrigatórios do trabalho:

### Teste 1: Tratamento de Erro (Entrada Inválida)
* **Objetivo:** Verificar se a syscall valida ponteiros nulos antes de desreferenciá-los.
* **Ação:** Chamada com parâmetro `NULL`: `ret = mm_get_custom_stats(NULL);`.
* **Resultado:** O kernel intercepta a condição e retorna imediatamente o código de erro POSIX `-EINVAL` (`-22`), preservando a integridade do sistema operacional.

### Teste 2: Linha de Base (Baseline em Repouso)
* **Objetivo:** Coletar a métrica inicial da memória da placa em estado estável antes de qualquer fragmentação induzida.
* **Ação:** `mm_get_custom_stats(&stats);`.
* **Resultado:** Os 5 campos são impressos no terminal serial com coerência (`total_used + total_free == total_heap`).

### Teste 3: Simulação de Fragmentação Externa
* **Objetivo:** Forçar um cenário de fragmentação e comprovar que a rotina acusa o aumento no número de blocos livres e a redução do maior bloco contíguo.
* **Ação:** 
  1. Alocam-se 3 blocos consecutivos na memória: `p1 = malloc(4096)`, `p2 = malloc(4096)` e `p3 = malloc(4096)`.
  2. Libera-se exclusivamente o bloco do meio: `free(p2)`.
  3. Isso cria um "buraco" isolado de 4096 bytes cercado por regiões alocadas (`p1` e `p3`), impedindo que ele seja fundido imediatamente.
  4. Nova coleta com `mm_get_custom_stats(&stats);`.
* **Resultado:** O campo `free_chunks` é incrementado em decorrência do bloco isolado. O teste evidencia a discrepância entre a memória livre total (`total_free`) e o maior bloco contíguo disponível (`largest_free_block`), demonstrando na prática o impacto da fragmentação externa.

### Teste 4: Limpeza e Coalescing (Fusão de Blocos Adjacentes)
* **Objetivo:** Comprovar que o algoritmo de *coalescing* do NuttX funciona em conjunto com a nossa contagem após a liberação dos blocos vizinhos.
* **Ação:**
  1. Liberação dos blocos adjacentes: `free(p1); free(p3);`.
  2. Nova coleta com `mm_get_custom_stats(&stats);`.
* **Resultado:** Os blocos adjacentes que ficaram livres são reunificados pelo gerenciador em um bloco contíguo amplo. O número de fragmentos (`free_chunks`) é reduzido e o `largest_free_block` é restaurado, confirmando a recuperação da memória contígua.

---

### Exemplo de Saída Obtida no Console Serial (NSH)

```text
nsh> mem_stats_test
=== Teste da Chamada de Sistema mm_get_custom_stats ===

Teste 1 - Entrada Invalida: ret=-22 (esperado -22)
Teste 1: SUCESSO

[Teste 2 - Baseline]
  Total Heap:         178496 B
  Total Free:         161824 B
  Total Used:         16672 B
  Largest Free Block: 159744 B
  Free Chunks:        3
Teste 2: SUCESSO

[Teste 3 - Fragmentacao Apos Liberar p2]
  Total Heap:         178496 B
  Total Free:         153600 B
  Total Used:         24896 B
  Largest Free Block: 147424 B
  Free Chunks:        4
Teste 3: SUCESSO (buraco de 4096B criado entre p1 e p3)

[Teste 4 - Apos Liberar p1 e p3 (Fusao)]
  Total Heap:         178496 B
  Total Free:         161824 B
  Total Used:         16672 B
  Largest Free Block: 159744 B
  Free Chunks:        3
Teste 4: SUCESSO (blocos adjacentes fundidos)

=== Bateria de Testes Finalizada com Exito ===
nsh>
```

---

## 6. Conclusão

A rotina `mm_get_custom_stats` foi integrada com sucesso ao kernel do Apache NuttX, permitindo diagnóstico em tempo real do heap. A validação demonstrou que o código é resiliente a entradas nulas, seguro contra condições de corrida e capaz de refletir fidedignamente o estado de fragmentação externa da memória física em microcontroladores ESP32 e ESP32-S3.
