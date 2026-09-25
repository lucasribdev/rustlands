# Rodadas de Rustlands

## Inicialização

Na raiz do repositório, use o perfil dedicado:

```sh
dotnet run --project Content.Server -- --config-file Resources/ConfigPresets/Rustlands.toml
```

Em uma instalação com configuração própria, adicione `presets = "Rustlands"` à seção
`[config]` do arquivo do servidor. Presets são valores padrão: valores explícitos
no arquivo do host e na linha de comando têm precedência. Remova overrides antigos
conflitantes de mapa, modo, eventos e shuttle. Os scripts genéricos `runserver.*`
continuam genéricos; executar apenas esses scripts não seleciona Rustlands.
O antigo override de Wasteland foi movido do exemplo `server_config.toml` para
`Rustlands.toml`. Nenhum padrão global em C# ou modo genérico foi alterado.

## Investigação e decisões

| Área | Herança encontrada | Base Rustlands |
| --- | --- | --- |
| Modo | Sem modo explícito, o padrão é `Secret`; fallback permite `Traitor,Extended`. | Preset `Rustlands`, `rules: []`, sem fallback e sem voto de troca de modo. |
| Eventos | O exemplo habilitava eventos. `Extended` ainda inclui meteoros, eventos de estação e tráfego espacial. `Greenshift` ainda aplica `BasicRoundstartVariation`. | Sem schedulers; `events.enabled = false`. Sem variações automáticas de fios, luzes, lixo, contrabando ou painéis solares. |
| Estação | `TestStation` já não tinha arrivals, CentComm ou evacuação, mas incluía alertas de estação. | `RustlandsStation` reutiliza somente `BaseStation`, `BaseStationJobsSpawning` e `BaseStationRecords`. Sem alertas, carga, CentComm, expedições ou elegibilidade a eventos. |
| Funções | O mapa só oferece `Passenger`, ilimitado no início e na entrada tardia. | Mantido para reutilizar criação de personagem, loadouts e registros. Não há seleção automática de antagonistas. |
| Objetivos | Regras de antagonistas podem gerar missões e condições de vitória ligadas à estação. | Sem objetivos atribuídos pelo modo, vitória obrigatória ou prazo. Explorar, sobreviver e interagir são escolhas dos jogadores. |
| Chegada | Existe um `SpawnPointLatejoin` no grid associado a `Wasteland`. | Adicionado `SpawnPointPassenger` no mesmo local para roundstart, evitando o fallback de spawn inválido; arrivals desligado. Lobby e latejoin habilitados. |
| Evacuação | `RoundEndSystem` verifica a chamada automática independentemente da lista de regras; o exemplo usa 90 minutos. | `shuttle.emergency = false`, chamada inicial e extensão zeradas, sem componente de evacuação na estação. |
| Encerramento | Regras específicas e evacuação podem encerrar a rodada; votação também pode reiniciar diretamente. | Sem regra de limite de tempo, inatividade ou vitória. Voto de reinício e comandos administrativos permanecem. |

Selecionar somente o modo Rustlands não aplica CVars do perfil de servidor.
É necessário carregar o perfil completo para impedir chamadas automáticas e
mudanças de modo. Admins ainda podem deliberadamente mudar CVars, adicionar regras
ou encerrar a sessão; esta configuração não bloqueia ferramentas administrativas.

## O que permanece ativo

Nenhuma entidade de `GameRule` é adicionada pelo novo preset. Continuam ativos o
GameTicker, lobby, personagens, vagas, spawn, registros e os sistemas normais dos
objetos e criaturas: dano, morte, fome, sede, inventário, interação e construção.
Não é o modo `Sandbox`: não concede ferramentas de criação aos jogadores.
O mapa mantém atmosfera respirável, gravidade e iluminação já configuradas.

A função provisória ainda se chama Passenger e reutiliza PDA, headset, acessos e
loadouts da estação. Não há cadeia de comando implementada para o deserto; textos
de supervisão e aparência do equipamento são dívida temática conhecida. Não foi
criado um sistema novo de profissões, missões, respawn ou persistência. Morrer não
encerra a rodada automaticamente nem garante renascimento.

O voto de reinício mantém as restrições genéricas (incluindo população/fantasmas,
quórum e presença de admin). `restartround` encerra e agenda a próxima rodada pelo
fluxo existente; `restartroundnow` reinicia imediatamente. Um reinício não preserva
o mundo. Não usar chamada de evacuação como mecanismo normal de encerramento.

## Validação e roteiro em jogo

A validação estática deve conferir TOML, CVars existentes, referências YAML,
localização, herança da estação e vínculo do ponto de entrada ao grid Wasteland.
O linter completo pode ser executado com:

```sh
dotnet run --project Content.YAMLLinter
```

Teste em jogo ainda necessário:

1. Iniciar pelo comando acima; conferir mapa Wasteland, modo Rustlands, lobby e
   ausência de regras automáticas com as ferramentas administrativas.
2. Entrar como Passenger no início e com um segundo cliente após o início:
   ambos devem aparecer no ponto local, sem terminal/nave de arrivals. Conferir
   preferências de personagem, loadout e ausência de objetivos de antagonista.
3. Explorar e testar dano, morte, fome/sede e interação. Conferir que não há
   concessão de ferramentas de Sandbox ou término automático com todos mortos.
4. Ultrapassar 90 minutos: nenhuma chamada de evacuação, meteoros ou tráfego
   espacial automático. Conferir também os valores efetivos dos CVars do perfil.
5. Conferir indisponibilidade de voto de modo/mapa; testar voto de reinício nas
   condições genéricas permitidas e `restartround`. A rodada seguinte deve manter
   Wasteland/Rustlands e permitir novas entradas.
6. Iniciar separadamente sem o perfil e conferir que modos e mapas genéricos
   continuam disponíveis. Não reutilizar overrides Rustlands nesse teste.

### Resultado desta execução

- Validação estática aprovada: TOML e nomes de CVars, referências YAML,
  localização, composição da estação, spawns inicial/tardio, contagem de entidades
  e restauração do exemplo genérico. Tags YAML do engine foram aceitas pelo
  parser estático; sua semântica ainda depende da validação pelo engine.
- `git diff --check`: sem erros.
- Tentativa do linter com o binário disponível:
  `dotnet bin/Content.YAMLLinter/Content.YAMLLinter.dll`. Bloqueada antes da
  validação por `ArgumentOutOfRangeException` em `ResourceCache.PreloadRsis`
  (`ImageSharp`, altura de imagem igual a zero). Isso não confirma aprovação
  dos protótipos pelo engine. Reexecutar o linter em ambiente funcional.
- Nenhum teste com clientes em jogo foi executado; roteiro acima pendente.
