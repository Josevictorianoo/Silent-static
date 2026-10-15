# S T A T I C

> Um protótipo de survival horror em texto, inspirado em **Silent Hill**, escrito em C++20 moderno.

[![CI](https://github.com/Josevictorianoo/silent-static/actions/workflows/ci.yml/badge.svg)](https://github.com/Josevictorianoo/silent-static/actions/workflows/ci.yml)
![C++20](https://img.shields.io/badge/C%2B%2B-20-blue)
![CMake](https://img.shields.io/badge/build-CMake-informational)
![Tests](https://img.shields.io/badge/tests-Catch2-brightgreen)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

Você acorda no Rosewater Park com uma carta no bolso. Em algum lugar além da névoa fica o
**quarto 312 do Lakeview Hotel**. O rádio chia quando algo está perto. A sirene toca. A cidade muda.

O projeto foi pensado como **vitrine de engenharia**: regras de jogo isoladas de I/O, algoritmos
de grafos, build com CMake, testes automatizados, sanitizers e CI.

```text
== Toluca Prison [OTHERWORLD] ==
Cell doors hang open. The rust on them is not the colour rust should be.
Everything here is rusted, wet and wrong.
!! Something is here with you: Pyramid Head, Lying Figure !!
Exits: Historical Society, Labyrinth
The Pyramid Head strikes you. (-3 HP, 5/10)
The Lying Figure strikes you. (-1 HP, 4/10)
Radio: #### KRRRRZZZZZZZZT ####
```

---

## Como jogar

**Objetivo:** chegar ao Lakeview Hotel com vida. O portão final está acorrentado: você precisa do
**Steel Pipe**, que fica fora do caminho mais curto (Brookhaven Hospital).

| Comando | Atalho | O que faz |
|---|---|---|
| `look` | `l` | descreve o lugar atual |
| `go <lugar>` | `g` | anda até um lugar vizinho (prefixos funcionam: `go neely`) |
| `take <item>` | | pega um item do chão (`take radio`) |
| `use <item>` | `u` | usa item de cura ou o rádio |
| `attack` | `a` | ataca com a melhor arma da mochila |
| `wait` | `w` | passa um turno |
| `radio` | `r` | escuta o chiado: quanto mais alto, mais perto o perigo |
| `map` | `m` | mostra o que a névoa deixa você ver |
| `inventory` | `i` | HP e itens |
| `quit` | `q` | desiste |

Andar, atacar, esperar e se curar gastam um turno. O resto é grátis.

**Dica:** a sirene avisa um turno antes de a cidade virar. Monstros do Otherworld ficam
*dormentes* quando a névoa volta. Às vezes esperar é a melhor jogada.

## Referências a Silent Hill

| Referência | Onde está no código |
|---|---|
| O **rádio** que chia perto de monstros | `radio_level()` em `src/radio.cpp` |
| A **névoa** que limita a visão (a lanterna aumenta o alcance) | `TownMap::within()` + `Game::visible_radius()` |
| A **sirene** e o **Otherworld** | `Game::world_at()`: ciclo de névoa e Otherworld, com aviso de um turno |
| **Pyramid Head**: só existe no Otherworld e te caça pelo caminho mais curto | `kMonsters` + `TownMap::next_step()` |
| **Lying Figure** e **Mannequin** | `kMonsters` em `include/sh/entities.hpp` |
| Rosewater Park, Neely/Saul/Carroll Street, Brookhaven, Toluca Prison, Labyrinth, Lakeview Hotel | `TownMap::silent_hill()` |
| Seed padrão **312** (Room 312) | `kDefaultSeed` em `include/sh/rng.hpp` |

## Compilando e rodando

Requisitos: **CMake ≥ 3.16** e um compilador **C++20** (desenvolvido com GCC 13; o CI cobre
Linux, macOS e Windows). Na primeira configuração o CMake baixa o Catch2 automaticamente,
a menos que ele já esteja instalado no sistema.

```bash
git clone https://github.com/Josevictorianoo/silent-static.git
cd silent-static

cmake -S . -B build
cmake --build build

./build/silent_static        # seed padrão (312)
./build/silent_static 42     # outra seed = outra cidade
```

No Windows (MSVC): `cmake --build build --config Release` e `build\Release\silent_static.exe`.

### Testes

```bash
ctest --test-dir build --output-on-failure
```

Opções do CMake:

| Opção | Padrão | Para quê |
|---|---|---|
| `-DSH_BUILD_TESTS=OFF` | ON | não compila os testes (nem baixa o Catch2) |
| `-DSH_WARNINGS_AS_ERRORS=ON` | OFF | qualquer warning quebra o build |
| `-DSH_ENABLE_SANITIZERS=ON` | OFF | AddressSanitizer + UBSan (GCC/Clang) |

## Estrutura

```text
.
├── CMakeLists.txt            # biblioteca + executável + testes
├── include/sh/
│   ├── town_map.hpp          # grafo da cidade + BFS (distance, next_step, within)
│   ├── entities.hpp          # itens, monstros e jogador (tabelas constexpr)
│   ├── radio.hpp             # nível de chiado em função da distância
│   ├── command.hpp           # parser de comandos
│   ├── game.hpp              # regras do jogo, sem nenhum I/O
│   ├── rng.hpp               # RNG determinístico
│   └── text.hpp              # normalização de texto
├── src/                      # implementações + main.cpp (a única parte com I/O)
├── tests/                    # Catch2: 35 casos de teste
└── .github/workflows/ci.yml  # Linux/macOS/Windows + ASan/UBSan
```

## Decisões técnicas

**Grafo com lista de adjacência + BFS.** A cidade é um grafo não ponderado (cada rua custa 1
passo), então BFS dá o caminho mínimo em O(V+E). Uma única rotina de BFS alimenta quatro
funcionalidades: distância entre lugares (rádio), próximo passo do Pyramid Head, raio de visão
(névoa) e a checagem de conectividade nos testes. A lista de adjacência é mais enxuta que uma
matriz para um grafo esparso e mantém a ordem de inserção, o que deixa a iteração determinística.

**Núcleo sem I/O.** Toda a regra do jogo fica na biblioteca `sh_core`; `Game::handle(string)`
recebe uma linha e devolve o texto da resposta. Só o `main.cpp` conhece `std::cin`/`std::cout`.
Resultado: os testes exercitam o jogo inteiro sem mocks, e trocar o terminal por uma TUI ou GUI
não toca em nenhuma regra.

**O mundo é função pura do turno.** `world_at(turn)` calcula se estamos na névoa ou no
Otherworld a partir do número do turno, em vez de guardar um booleano que alguém precisa lembrar
de alternar. Não existe estado que possa ficar dessincronizado, e o aviso da sirene sai de graça
(`world_at(turn + 1)`).

**RNG determinístico entre plataformas.** O `std::mt19937` é totalmente especificado pelo padrão,
mas o `std::uniform_int_distribution` não é: cada biblioteca padrão pode mapear os números de um
jeito. Por isso o jogo usa o motor diretamente, e a mesma seed gera a mesma cidade em Linux,
macOS e Windows. Isso torna os testes de seed confiáveis.

**Conteúdo orientado a dados.** Itens e monstros são tabelas `constexpr` (`kItems`, `kMonsters`).
Adicionar um monstro novo é adicionar uma linha, sem tocar na lógica.

**Dois tipos de erro, tratados de forma diferente.** Usar a API errado (ex.: pedir um lugar que
não existe) lança exceção. Já o jogador digitar bobagem nunca lança: vira uma mensagem de
volta. Um teste de fuzz joga milhares de comandos aleatórios e confere os invariantes.

**`Game` é um tipo-valor.** Como o estado é copiável e não há globais, "e se eu fizer isso?"
é só copiar o jogo. Os testes usam isso para rodar um **solver DFS** que prova que
toda seed testada tem pelo menos uma sequência vencedora, ou seja, o balanceamento nunca gera um
mapa impossível.

**CMake enxuto.** Biblioteca + executável + testes, um alvo alias (`sh::core`), política de
warnings definida em uma função só, `find_package(Catch2)` com fallback para `FetchContent`
(versão fixada) e `catch_discover_tests` para o `ctest` enxergar cada caso individualmente.

### O que os testes cobrem

- **Mapa:** caminhos mínimos, atalhos em ciclos, nós isolados, raio de visão, consistência do
  mapa real (conectado, ruas de mão dupla, hotel a 7 passos).
- **Rádio:** tabela de limiares (`GENERATE`), efeito do Otherworld, saturação no nível máximo.
- **Parser:** aliases, caixa alta, espaços extras, apóstrofos (`Jack's Inn`).
- **Regras:** movimento, vitória/derrota, itens, cura com limite, combate, portão acorrentado,
  monstros dormentes, aviso da sirene.
- **Propriedades:** determinismo por seed, nada hostil no início, invariantes sob entrada aleatória,
  toda seed é vencível.

## Roadmap

- [ ] Salvar e carregar partida
- [ ] Mapas carregados de arquivo (JSON), sem recompilar
- [ ] Interface TUI (ncurses / FTXUI) reaproveitando `sh_core`
- [ ] Mais finais, dependendo de HP, turnos e itens
- [ ] Inglês e português selecionáveis

## Aviso legal

Projeto de fã, sem fins lucrativos, **sem relação com a Konami**. Silent Hill e seus personagens
e locais são marcas de seus respectivos donos. Todo o código e todos os textos aqui são originais:
nenhum asset nem trecho de diálogo dos jogos foi usado.

## Licença

[MIT](LICENSE)
