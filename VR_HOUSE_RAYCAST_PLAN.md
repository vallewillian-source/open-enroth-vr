# Plano Incremental — Raycast “Mouse” nas Telas 2D de Casas (MM7 VR / OpenEnroth)

## Contexto (onde estamos hoje)

Quando o jogador entra em uma casa/loja, o jogo troca para uma UI 2D (screen type `SCREEN_HOUSE`) e pausa o tempo do jogo. A renderização base continua acontecendo via OpenGL, mas a “cena” exibida é essencialmente 2D (fundo + botões + textos).

Para VR, já existe um mecanismo que captura o frame 2D nativo e o reapresenta no headset como um “monitor virtual”.

Pontos relevantes do fluxo atual:

- **Detecção de casa e captura do frame para overlay**
  - Em cada `Engine::Draw()`, o frame 2D é capturado para uma textura de overlay (`VRManager::CaptureScreenToOverlay`) e, se o screen type for `SCREEN_HOUSE`, o “indicador de casa” é ativado via `VRManager::SetDebugHouseIndicator(true)` ([Engine.cpp:L220-L267](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/Engine.cpp#L220-L267)).
- **Renderização do “monitor virtual” no loop VR**
  - Ainda em `Engine::Draw()`, durante o loop de olhos VR, após `drawWorld()`, o overlay é desenhado via `VRManager::RenderOverlay3D()` ([Engine.cpp:L245-L264](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/Engine.cpp#L245-L264)).
- **Implementação atual do overlay e do indicador de casa**
  - `RenderOverlay3D()` desenha um quad “head-locked” com a textura do overlay e, quando `m_debugHouseIndicator` está ativo, desenha a “tela de casa” com `glScissor`/projeção ortográfica, usando uma âncora capturada ao entrar na casa ([VRManager.cpp:L559-L627](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/VR/VRManager.cpp#L559-L627)).
- **Entrada/saída de casas**
  - A entrada e criação da UI de casas ocorre em `enterHouse()` e `createHouseUI()` ([UIHouses.cpp](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/GUI/UI/UIHouses.cpp)).
  - O escape/saída é tratado por `houseDialogPressEscape()` ([UIHouses.cpp](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/GUI/UI/UIHouses.cpp)).

Observação importante para este plano: Existe um cursor VR diferente para o **menu principal** (telas de menu), via `VRManager::GetMenuMouseState()`, que hoje move um cursor 2D com thumbstick/triggers (sem raycast) e injeta posição/clique no `Mouse` ([GameMenu.cpp:L468-L519](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Application/GameMenu.cpp#L468-L519), [Mouse.cpp:L107-L152](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Io/Mouse.cpp#L107-L152), [VRManager.cpp:L1145-L1289](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/VR/VRManager.cpp#L1145-L1289)).

O raycast “mouse” para casas deve ser implementado sem interferir nesse caminho de menus.

## Objetivo geral (visão final)

Somente dentro de casas/lojas (`SCREEN_HOUSE`), permitir que cada controle VR funcione como um “mouse” sobre a tela 2D (monitor virtual), incluindo:

- Ray visível saindo do controle (debug e UX).
- Interseção com a superfície da tela 2D, gerando coordenadas (X,Y) em pixels.
- Cursor desenhado na UI e movimentação do “mouse” interno do jogo.
- Clique (trigger) e eventualmente arrastar/scroll, sem quebrar navegação padrão.
- Suporte a **qualquer mão** (esquerda/direita), com fallback simples.

## Primeira meta (etapa 1)

Implementar apenas um **raycast visual** saindo do **controle esquerdo** enquanto estiver em `SCREEN_HOUSE`, sem interação com a UI (sem mover mouse, sem clique, sem alterar lógica do jogo).

Critério de sucesso:

- Em VR e em `SCREEN_HOUSE`, o ray aparece e se atualiza com a pose do controle esquerdo.
- Fora de `SCREEN_HOUSE`, nada muda.
- O restante do jogo (mundo 3D, menus, inputs existentes) não é afetado.

## Restrições e princípios

- **Isolamento por estado**: tudo deve ficar “atrás” de `current_screen_type == SCREEN_HOUSE` e `VRManager::IsInitialized()/IsSessionRunning()`.
- **Isolamento por feature flag**: cada etapa deve ter uma flag (config/env) para habilitar/desabilitar rapidamente durante debug.
- **Não contaminar pipeline OpenGL do jogo**: qualquer desenho adicional em VR deve restaurar estados de GL (blend/depth/scissor/viewport) para não quebrar renderização de mundo/UI.
- **Sem side-effects** na etapa 1: não chamar `mouse->setPosition`, não disparar `UI_OnMouseLeftClick`, não modificar `currentHouseNpc`, não enviar mensagens para `_messageQueue`.

## Plano incremental (etapas)

### Etapa 0 — “Mapa e trilhos de segurança” (pré-código do ray)

Objetivo:

- Garantir pontos claros de integração e toggles para não quebrar o resto.

Entrega:

- Um único “gate” de runtime, por exemplo (nomes ilustrativos):
  - `OPENENROTH_VR_HOUSE_POINTER=1` (habilita a feature inteira)
  - `OPENENROTH_VR_HOUSE_POINTER_DEBUG=1` (habilita desenho do ray)
- Logs mínimos em eventos críticos:
  - transição `SCREEN_GAME -> SCREEN_HOUSE` e `SCREEN_HOUSE -> SCREEN_GAME` (já existe a detecção em [Engine.cpp:L236-L243](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/Engine.cpp#L236-L243)).

Critério de sucesso:

- Com flag desligada: comportamento idêntico ao atual.
- Com flag ligada: ainda sem desenho/interação, mas sem regressões.

### Etapa 1 — Pose do controle esquerdo + ray visual (sem UI)

Objetivo:

- Obter pose estável do controle esquerdo via OpenXR e renderizar uma linha (“laser”) em VR.

Mudanças previstas (alto nível):

- Criar uma `XrAction` de pose para a mão esquerda (ex.: `XR_ACTION_TYPE_POSE_INPUT`) e uma `XrSpace` (action space) associada.
- Em cada frame VR (após `xrWaitFrame/xrBeginFrame`), localizar a pose do controle (`xrLocateSpace`) no mesmo space usado para views (hoje `m_appSpace`).
- Transformar pose (posição + orientação) em:
  - `rayOrigin` (posição do controle)
  - `rayDirection` (eixo “forward” do controle, normalmente `-Z` no espaço do action pose)
- Desenhar o ray:
  - Alternativa A (mais barata): desenhar apenas um “segmento” em tela (debug 2D), sem geometria 3D real.
  - Alternativa B (recomendada): desenhar um segmento 3D no espaço do mundo VR como um “line mesh” simples (2 vértices) no mesmo framebuffer do olho, dentro do caminho VR (onde hoje já existe código raw OpenGL em `RenderOverlay3D()`).

Riscos e mitigação:

- **Risco: estados OpenGL** (blend/depth/scissor) vazarem e quebrarem o mundo 3D.
  - Mitigação: salvar/restaurar `glIsEnabled` e `glGetIntegerv` (viewport/scissor) no início/fim do bloco de debug do ray.
- **Risco: controle sem tracking** em alguns runtimes.
  - Mitigação: se `xrLocateSpace` não retornar flags de pose válidos, esconder ray (não “inventar” pose).

Critério de sucesso:

- Ray do controle esquerdo aparece em `SCREEN_HOUSE`.
- Sem alterações no mouse/cursor do jogo.

### Etapa 2 — Modelo físico do “monitor de casa” (plano de interseção)

Objetivo:

- Definir uma superfície 3D consistente para a tela 2D de casas, compatível com o que já é renderizado.

Estado atual (importante):

- A “tela de casa” hoje é desenhada como um retângulo **em coordenadas de tela** (scissor + ortho), a partir da projeção do ponto âncora `m_houseOverlayWorldPos` ([VRManager.cpp:L578-L626](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/VR/VRManager.cpp#L578-L626)).
- Existe `m_houseOverlayWorldRot`, mas ela ainda não participa do desenho do retângulo (ela é capturada, porém não aplicada na renderização atual) ([VRManager.cpp:L564-L575](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/VR/VRManager.cpp#L564-L575)).

Duas opções (incrementais) para tornar o raycast robusto:

- **Opção A (mínima, compatível com o desenho atual)**:
  - Converter o ray 3D do controle para “ray em tela” por olho (projetar pontos do ray com `view/projection` e trabalhar em NDC/pixels).
  - Intersectar com o retângulo em pixels usado pelo scissor (mesma área em que a textura é desenhada).
  - Isso produz (x,y) em pixels do framebuffer do olho, e daí UV da textura.
  - Vantagem: não exige refatorar o desenho do monitor de casa.
  - Desvantagem: é mais “hacky” e depende de detalhes do scissor.
- **Opção B (melhor base para UX, recomendada antes de clicks)**:
  - Renderizar o monitor de casa como um **quad 3D world-locked real** (model matrix usando `m_houseOverlayWorldPos` + `m_houseOverlayWorldRot`).
  - Definir o tamanho físico do quad (metros) e manter proporção do conteúdo (ex.: 4:3 ou usar dimensões reais do backbuffer).
  - Fazer interseção ray vs plano/quad em espaço 3D e obter UV diretamente.
  - Vantagem: raycast fica simples e previsível; alinhamento visual tende a ser melhor.
  - Desvantagem: exige mexer em `RenderOverlay3D()` (mas ainda isolado a `SCREEN_HOUSE`).

Critério de sucesso:

- A superfície da tela tem um “sistema de coordenadas” consistente (origem, eixos, UV).

### Etapa 3 — Hit test (ray → UV → pixels), ainda sem mover mouse

Objetivo:

- Calcular, quando o ray “atinge” a tela, um ponto de hit:
  - `hitWorldPos` (para debug 3D)
  - `hitUV` (0..1)
  - `hitPixel` (x,y no buffer 2D do jogo)

Entrega de debug:

- Desenhar um marcador de hit (ex.: pequeno ponto) sobre a tela no VR:
  - se Opção A: desenhar marcador em coordenadas de tela (overlay 2D no olho)
  - se Opção B: desenhar um pequeno quad/point 3D na própria superfície do monitor

Critério de sucesso:

- O marcador acompanha corretamente onde o ray “aponta” na tela, sem atrasos perceptíveis.
- Ainda sem efeitos colaterais no jogo.

### Etapa 4 — Injeção de movimento do mouse (sem clique)

Objetivo:

- Quando houver hit válido, mover o mouse interno do jogo para `(x,y)` durante `SCREEN_HOUSE`.

Recomendação de integração:

- Reusar o padrão já aplicado em menus:
  - `mouse->setPosition({x,y})` (como em [GameMenu.cpp:L490-L500](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Application/GameMenu.cpp#L490-L500)).
- Gating estrito:
  - Só injetar mouse em `SCREEN_HOUSE`.
  - Preferir “modo exclusivo” (quando pointer ativo, esconder cursor OS e não ler mouse físico), mas isso deve ser incremental.

Critério de sucesso:

- A UI 2D de casa reage a hover (quando existir) e o cursor do jogo acompanha.
- Nada muda fora de casas.

### Etapa 5 — Clique (trigger) e “botão esquerdo”

Objetivo:

- Simular clique esquerdo do mouse com trigger (mão esquerda inicialmente; depois ambas).

Integração:

- Reusar `mouse->UI_OnMouseLeftClick()` (como em [Mouse.cpp:L116-L121](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Io/Mouse.cpp#L116-L121)).
- Implementar debounce (edge detect) como no caminho de menu (`m_menuSelectPressedPrev` em `GetMenuMouseState`) ([VRManager.cpp:L1269-L1286](file:///c:/Users/Usu%C3%A1rio/www/mm7_vr/src/Engine/VR/VRManager.cpp#L1269-L1286)).

Critério de sucesso:

- Selecionar NPCs, opções de diálogo e “Exit” funciona via VR pointer.
- Escape continua funcionando.
- Sem double-click acidental por frame.

### Etapa 6 — Duas mãos e regras de prioridade

Objetivo:

- Permitir pointer por ambas as mãos e definir política clara:
  - “mão ativa” (ex.: última que moveu o ray)
  - ou “mão dominante configurável”
  - ou “ambas visíveis, mas só uma controla o cursor”

Critério de sucesso:

- UX consistente: cursor não “teleporta” entre mãos sem intenção.

### Etapa 7 — Polimento (scroll/drag, feedback visual, conforto)

Possíveis incrementos:

- Scroll via thumbstick.
- Drag & drop (itens, se aplicável).
- Feedback visual (cursor com profundidade, highlight).
- Ajustes de tamanho/distância do monitor em casa.

## Estratégia de validação (por etapa)

- **Validação de regressão geral**
  - Iniciar jogo, entrar/sair de casa repetidamente, verificar que:
    - não há crash
    - overlay continua aparecendo no VR em `SCREEN_HOUSE`
    - menus fora de casa continuam funcionando
- **Validação de performance**
  - Garantir que o debug do ray não degrade FPS perceptivelmente (principalmente se desenhar geometria extra).
- **Validação de isolamento**
  - Confirmar que nenhuma chamada relacionada a mouse/click executa na etapa 1–3.

## Notas de implementação (direção técnica sugerida)

- Centralizar o estado do “House Pointer” dentro do `VRManager` (mesmo que temporariamente) para evitar espalhar lógica no engine.
- Evitar acoplar no pipeline de `Mouse.cpp` na etapa 1–3 (somente na etapa 4+).
- Preferir “um lugar” para render debug (idealmente dentro do caminho VR, junto de `RenderOverlay3D()`), para não interferir no render 2D do monitor.

