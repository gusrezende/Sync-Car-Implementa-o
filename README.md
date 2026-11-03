# Sincronização do Metrô (sync_car)

Implementação das funções de sincronização entre vagões e passageiros usando
apenas `pthread_mutex` e `pthread_cond` (sem semáforos, sem busy-wait, um único
lock por `struct station`).

## Arquivos

| Arquivo | Descrição |
|---|---|
| `sync_car.h` | Declaração da `struct station` e dos protótipos das funções. |
| `sync_car.c` | Implementação de `station_init`, `station_load_car`, `station_wait_for_car` e `station_on_board`. |
| `sync_car-runner.c` | Programa de teste fornecido pelo professor (cria 100 passageiros e vários carros com quantidades aleatórias de assentos). |
| `Makefile` | Compila o projeto e gera o executável `sync_car`; `make clean` remove os arquivos compilados. |

## Como compilar

```bash
make
```

## Como executar / testar

```bash
./sync_car        # ou: make run
```

Se tudo estiver correto, a última linha impressa é `Looks good!`.
Para testar várias vezes seguidas (bom para pegar condições de corrida):

```bash
for i in $(seq 1 50); do ./sync_car > /dev/null || echo "falhou na rodada $i"; done
```

## Como limpar

```bash
make clean
```

## Ideia da implementação

A `struct station` guarda um mutex, duas variáveis de condição e três contadores:

- `waiting`: passageiros esperando na plataforma;
- `free_seats`: assentos livres no carro que está na estação (0 = nenhum carro / carro cheio);
- `boarding`: passageiros que já pegaram um assento mas ainda não chamaram `station_on_board`.

1. **`station_load_car`**: define `free_seats = count`, acorda todos os passageiros
   (`broadcast` em `car_arrived`) e espera em `car_loaded` enquanto
   *(ainda há assentos **e** passageiros esperando)* **ou** *(alguém ainda está embarcando)*.
   Ao sair, zera `free_seats` para que ninguém embarque num carro que já partiu.
2. **`station_wait_for_car`**: incrementa `waiting` e espera em `car_arrived` enquanto
   `free_seats == 0`. Ao acordar com assento livre, reserva o assento
   (`waiting--`, `free_seats--`, `boarding++`) e retorna — vários passageiros podem
   retornar antes de qualquer um chamar `station_on_board`.
3. **`station_on_board`**: decrementa `boarding`; se ninguém mais está embarcando e o carro
   está cheio ou a plataforma está vazia, sinaliza `car_loaded` para o carro partir.

Todas as esperas usam `while` em volta do `pthread_cond_wait` para tratar
*spurious wakeups*.

## Sistema operacional

Desenvolvido e testado em Linux (gcc, glibc/pthreads).
