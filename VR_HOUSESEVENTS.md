# Sistema de Casas no OpenEnroth (MM7 VR)

Este documento detalha o funcionamento técnico da transição do ambiente 3D para as telas 2D de interiores de casas (lojas, guildas, templos, etc.) no projeto OpenEnroth, servindo como base para a implementação da interface em VR.

## 1. Fluxo de Entrada (3D -> 2D)

A entrada em uma casa é disparada por eventos no mapa (geralmente ao clicar em uma porta ou decoração interativa).

### Gatilho de Evento
- O interpretador de eventos ([EvtInterpreter.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Evt/EvtInterpreter.cpp)) processa o opcode `EVENT_SpeakInHouse`.
- Este opcode chama a função `enterHouse(houseId)`.

### Validação e Preparação
- **Arquivo**: [UIHouses.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIHouses.cpp)
- **Função**: `enterHouse(HouseId uHouseID)`
    - Limpa a fila de mensagens e a barra de status.
    - Verifica o horário de funcionamento da casa (com base na `houseTable`).
    - Verifica se o grupo está banido da loja.
    - Prepara a lista de NPCs presentes na casa via `prepareHouse()`.
    - Retorna `true` se a entrada for permitida.

### Instanciação da Interface
- Se `enterHouse` for bem-sucedido, o interpretador chama `createHouseUI(houseId)`.
- **Função**: `createHouseUI(HouseId houseId)`
    - Cria uma instância específica de `GUIWindow_House` baseada no tipo da casa (ex: `GUIWindow_Bank`, `GUIWindow_MagicGuild`, `GUIWindow_Temple`).
    - A variável global `window_SpeakInHouse` armazena a janela ativa.

### Estado da Tela
- No construtor de `GUIWindow_House`:
    - `current_screen_type` é alterado para `SCREEN_HOUSE`.
    - `pEventTimer->setPaused(true)` pausa o tempo do jogo para evitar ataques enquanto o jogador está no menu.
    - Carrega o fundo 2D (shop background) e cria os botões dos NPCs.

## 2. Interface e Interação 2D

Diferente do menu principal, as casas possuem uma estrutura de interface mais complexa:
- **Fundo Estático**: Imagem 2D representando o interior.
- **Retratos de NPCs**: Localizados na parte inferior, permitem alternar entre os personagens da casa.
- **Menu Lateral (Direito)**: Opções de diálogo e serviços (comprar, vender, treinar).
- **Diálogo**: Área central/lateral para textos de NPCs.

## 3. Fluxo de Saída (2D -> 3D)

A saída ocorre quando o jogador clica no botão "Exit" ou pressiona Escape.

### Lógica de Fechamento
- **Função**: `houseDialogPressEscape()` em [UIHouses.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIHouses.cpp).
    - Se estiver em um sub-menu (ex: comprando itens), volta para o menu principal da casa.
    - Se estiver no menu principal da casa, limpa `currentHouseNpc`, libera a janela `pDialogueWindow` e retorna `false`, sinalizando a saída definitiva.
- No `EvtInterpreter.cpp` ou no loop principal, ao detectar a saída:
    - `window_SpeakInHouse->Release()` é chamado.
    - `current_screen_type` volta para `SCREEN_GAME`.
    - O tempo do jogo é retomado.

## 4. Implementação da Tela 2D em VR (Virtual Monitor)

Para integrar as interfaces 2D das casas na experiência VR, implementamos um sistema de "Monitor Virtual" que ancora a tela no espaço 3D.

### Fluxo de Renderização
A renderização da interface em VR ocorre em três etapas principais:

1. **Captura do Frame (Blit):**
   - No loop principal em [Engine.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Engine.cpp), a função `CaptureScreenToOverlay` é chamada.
   - Ela utiliza `glBlitFramebuffer` para copiar o conteúdo do backbuffer (onde a interface 2D padrão é desenhada) para uma textura dedicada `m_overlayTexture` no `VRManager`.

2. **Ancoragem no Mundo (World-Locking):**
   - Ao detectar que `current_screen_type == SCREEN_HOUSE`, o `VRManager` ativa o `m_debugHouseIndicator`.
   - Na primeira execução dentro da casa, o sistema captura a posição e orientação atual do HMD via `m_views[m_currentViewIndex].viewMatrix`.
   - A tela é posicionada a **2.5 metros** à frente da posição capturada, criando um ponto fixo no espaço 3D ([VRManager.cpp:L570](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/VR/VRManager.cpp#L570)).

3. **Projeção Estéreo e Scissor:**
   - A cada frame de cada olho, a posição 3D da tela é projetada de volta para coordenadas de tela (NDC) usando as matrizes de visão e projeção específicas do olho.
   - Utilizamos `glScissor` para definir a área de exibição e desenhamos a textura capturada usando uma projeção ortogonal ([VRManager.cpp:L606-612](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/VR/VRManager.cpp#L606-612)).
   - Isso garante que cada olho veja a tela com o deslocamento correto, proporcionando profundidade 3D natural e conforto visual (convergência estéreo).

### Controle de Estado
- **Entrada**: `SetDebugHouseIndicator(true)` reseta `m_housePoseInitialized`, forçando a captura de uma nova âncora baseada na posição atual do jogador ao entrar na casa.
- **Saída**: Ao retornar para `SCREEN_GAME`, o indicador é desativado, removendo a tela 2D da visão VR.

## 5. Próximos Passos (VR Port)

Seguiremos estas diretrizes para refinamento:
1. **Cursor**: Mapear o cursor do mouse para o analógico esquerdo (ou movimento da mão).
2. **Seleção**: Utilizar os triggers dos controladores para simular o clique do mouse.
3. **Logs de Depuração**: Foram adicionados logs em `enterHouse` e `houseDialogPressEscape` para rastrear os IDs das casas e validar o mapeamento de eventos.
