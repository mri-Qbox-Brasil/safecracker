# safecracker — Manual

Minigame de arrombamento de cofre: o jogador gira o disco com as setas até acertar, em ordem, cada número da combinação.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Como o minigame funciona](#como-o-minigame-funciona)
5. [Controles](#controles)
6. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
7. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` | Sim | `exports['qb-core']:GetCoreObject()` e `QBCore.Functions.Notify` para o resultado |

O `fxmanifest.lua` não declara dependências, mas o `client.lua` chama o `qb-core` na primeira linha. O recurso é **client-only** — não há arquivo de servidor.

---

## Instalação

1. Copie a pasta `safecracker` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure safecracker
   ```
3. Este recurso não faz nada sozinho: ele só reage ao evento `SafeCracker:StartMinigame`, que precisa ser disparado por outro recurso (um assalto, um roubo de loja etc.). Veja [Entrypoints](#entrypoints-para-outros-recursos).

Não há SQL nem itens de inventário. Não há conflitos conhecidos.

---

## Configuração

Tudo fica em `config.lua`, na tabela global `SafeCracker`.

### `SafeCracker.Config`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `LockTolerance` | number | Sim | Quantos "cliques" o jogador pode passar do pino antes de a fechadura falhar. Internamente é multiplicado por `3.60` (graus por clique) |
| `AudioBankName` | string | Sim | Banco de áudio ambiente carregado no start (`SAFE_CRACK`) |
| `TextureDict` | string | Sim | Nome do dicionário de textura em runtime criado a partir dos PNGs do disco |
| `SafeSoundset` | string | Sim | Soundset usado em todos os efeitos sonoros (`SAFE_CRACK_SOUNDSET`) |
| `SafeTurnSound` | string | Sim | Som de cada clique ao girar o disco |
| `SafePinSound` | string | Sim | Som ao acertar um pino |
| `SafeFinalSound` | string | Sim | Som ao acertar o último pino (cofre aberto) |
| `SafeResetSound` | string | Sim | Som de reset da fechadura |
| `SafeOpenSound` | string | Sim | Som de abertura da porta do cofre |

### `SafeCracker.SafeModels`

| Campo | Tipo | Descrição |
|---|---|---|
| `Safe` | string | Modelo do corpo do cofre (`bkr_prop_biker_safebody_01a`) |
| `Door` | string | Modelo da porta do cofre (`bkr_prop_biker_safedoor_01a`) |

### `SafeCracker.SafeObjects`

Objetos spawnados por `SafeCracker:SpawnSafe`. Cada entrada tem `ModelName`, `Pos` (offset `vector3` relativo à posição informada), `Heading` (offset), `Rot` e `Frozen`.

### `Keys`

Tabela global de mapeamento de teclas para IDs de controle do GTA V. É definida no `config.lua`, mas o `client.lua` usa os IDs numéricos diretamente — a tabela existe apenas como referência.

---

## Como o minigame funciona

1. Outro recurso dispara `SafeCracker:StartMinigame` passando a combinação (`combo`): um array de números que representam **rotações do disco em graus** (não os números impressos no mostrador).
2. O ped é congelado, alinhado com o cofre mais próximo (`v_ilev_gangsafedoor`) e toca a animação `mini@safe_cracking`.
3. O disco começa numa rotação aleatória, escolhida para ficar longe do primeiro número da combinação.
4. O jogador gira o disco. Ao passar exatamente pelo número da vez, um pino cai (som + avanço para o próximo número da combinação).
5. **Falha:** se o jogador ultrapassar o pino já acertado em mais de `LockTolerance` cliques na direção errada, a fechadura reseta e o minigame termina em falha. Morrer também encerra em falha.
6. **Sucesso:** ao acertar todos os números da combinação, o minigame termina em sucesso.
7. Nos dois casos, o recurso emite `SafeCracker:EndMinigame` com um booleano, notifica via `QBCore.Functions.Notify`, descongela o ped e limpa as tasks.

---

## Controles

| Tecla | Ação |
|---|---|
| Seta esquerda / Seta direita | Gira o disco |
| `Z` (segurando) | Gira devagar (maior precisão) |
| `Left Shift` (segurando) | Gira rápido |
| `ESC` | Cancela o minigame (conta como falha) |

Sem nenhum modificador, a velocidade de giro é intermediária.

---

## Entrypoints para outros recursos

### Iniciar o minigame

```lua
-- combo = rotações em graus, na ordem em que devem ser acertadas
TriggerEvent('SafeCracker:StartMinigame', { 120, 300, 45 })
```

### Receber o resultado

O recurso emite este evento no fim do minigame — é a forma de saber se o jogador conseguiu:

```lua
AddEventHandler('SafeCracker:EndMinigame', function(won)
    if won then
        -- abrir o cofre, dar loot etc.
    end
end)
```

### Encerrar o minigame por fora

```lua
TriggerEvent('SafeCracker:EndGame')
```

> Cuidado: `EndGame` chama `EndMinigame()` sem argumento, então `won` chega como `nil` (tratado como falha) e o evento `SafeCracker:EndMinigame` é emitido com `nil`.

### Spawnar um cofre

Cria os props de corpo e porta do cofre na posição informada.

```lua
TriggerEvent('SafeCracker:SpawnSafe', SafeCracker.SafeObjects, vector3(x, y, z), heading, function(objs)
    -- objs = { [ModelName] = handle }
end)
```

Passar `nil` no primeiro argumento faz o recurso usar `SafeCracker.SafeObjects`. Os objetos criados ficam em `SafeCracker.Objects` e podem ser removidos com a função global `DelSafe()`.

---

## Estrutura de arquivos

```
safecracker/
├── client.lua        — loop do minigame, input, sons, sprites do disco, spawn/remoção do cofre e eventos
├── config.lua        — tolerância, sons, dicionário de textura, modelos e objetos do cofre; tabela Keys de referência
├── LockPart1.png     — sprite do disco giratório (mostrador com os números)
├── LockPart2.png     — sprite da moldura fixa do disco
├── fxmanifest.lua
└── README.md
```
