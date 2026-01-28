# Relatório — Problema 2: posicionamento “aleatório” de assets 2D no VR

## Contexto e sintoma
No desktop (visão flat), sprites/billboards/partículas e efeitos (assets 2D) aparecem corretamente posicionados no mundo 3D. No VR, esses mesmos assets aparecem com posicionamento que “parece aleatório”.

O padrão do sintoma é compatível com uma pipeline que:
1) calcula a posição/tamanho do billboard em *screen-space* (pixels) na CPU usando a câmera “flat”; e
2) reaproveita esse resultado durante a renderização estereoscópica (por olho), onde a projeção e a view diferem.

## Como o pipeline funciona hoje (desktop)
### 1) Projeção e escala em screen-space na CPU
O caminho principal para sprites “no mundo” passa por:
- [BaseRenderer::AddBillboardIfVisible](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/BaseRenderer.cpp#L349-L382):
  - Faz `ViewClip(pos, &viewspace)` e depois `Project(viewspace)` para obter `screenPos` (pixels).
  - Calcula escala em pixels via `ViewPlaneDistPixels / viewspace.x`.
  - Faz culling por viewport (`pViewport.intersects(billboardRect)`).
  - Armazena `RenderBillboard` com `screenPos`, `view_space_z` etc.

Em seguida, esses billboards são “convertidos” para quads:
- [BaseRenderer::TransformBillboard](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/BaseRenderer.cpp#L267-L347):
  - Constrói os 4 vértices do quad em *screen-space* (x/y em pixels) usando `screenPos` + fatores de projeção.
  - Escreve em `RenderBillboardD3D::pQuads[].pos.x/.pos.y` (screen-space) e `pos.z = view_space_z`.

Partículas e alguns efeitos seguem o mesmo modelo:
- [BaseRenderer::MakeParticleBillboardAndPush](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/BaseRenderer.cpp#L384-L442) usa `p.uScreenSpaceX/Y`.
- [SpellFX_Billboard::SpellFXProject](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/SpellFxRenderer.cpp#L1373-L1396) usa `pCamera3D->Project(...)` e grava `field_104[i].pos.x/y` em screen-space.

### 2) Renderização dos billboards em ortho
Os quads já pré-projetados (em pixels) são renderizados em um pass “2D/ortho”:
- [OpenGLRenderer::DoRenderBillboards_D3D](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/OpenGLRenderer.cpp#L1751-L1873)
  - Chama `_set_ortho_projection(1)` e `_set_ortho_modelview()`.
  - Alimenta `_billboardVertices` com `pos=(x_pixels, y_pixels, depth_normalizado)`.

Ou seja: o “billboard no mundo” aqui é, na prática, um *sprite* já convertido para “onde ele cai na tela”, com um z aproximado.

## Por que isso quebra no VR (o desafio real)
### 1) VR exige projeção estereoscópica e frustums assimétricos
No VR, cada olho tem:
- uma matriz de view diferente (offset interpupilar + rotação/pose do HMD);
- uma matriz de projeção diferente e tipicamente **assimétrica** (centro de projeção deslocado).

O pipeline atual de `Camera3D::Project()`:
- assume uma projeção **simétrica** e baseada em viewport/“distância do plano de projeção” em pixels:
  - [Camera3D::CreateViewMatrixAndProjectionScale](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Camera.cpp#L163-L193)
  - [Camera3D::Project](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Camera.cpp#L364-L367)

Isso não modela o caso VR corretamente, porque no VR o “centro” do olho não coincide com `pViewport.center()` e o frustum não é necessariamente simétrico.

### 2) O pass de billboards está em ortho (screen-space), não em 3D
Mesmo quando a cena 3D usa a projeção do OpenXR por olho:
- [OpenGLRenderer::_set_3d_projection_matrix](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/OpenGLRenderer.cpp#L942-L954)

os billboards são renderizados com `_set_ortho_projection(1)`:
- [OpenGLRenderer::_set_ortho_projection](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/OpenGLRenderer.cpp#L986-L1003)

Isso força o billboard a “existir” em pixels na tela, o que não é compatível com:
- paralaxe (diferença entre olhos),
- convergência correta,
- consistência de profundidade no buffer (a cena 3D foi rasterizada com outra projeção).

### 3) Mesmo que recalculássemos screen-space, teríamos que fazer “por olho”
Hoje, `TransformBillboards()` é chamado no fluxo de desenho do mundo:
- Outdoor: [OutdoorLocation::ExecDraw](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Outdoor.cpp#L155-L220)
- Indoor: [IndoorLocation::Draw](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Indoor.cpp#L191-L199)

Para VR, o “jeito certo” exigiria (no mínimo) recalcular `screenPos`, escala e quads **por olho** (porque a projeção muda), o que implica:
- duplicar listas/render state por olho, ou
- recalcular imediatamente antes de renderizar cada olho.

## Melhores caminhos para resolver
### Caminho A (recomendado): billboards em world-space (GPU) com view/proj por olho
**Ideia:** parar de pré-projetar sprites para pixels na CPU. Em vez disso, enviar para GPU:
- posição no mundo (centro do billboard),
- tamanho em unidades do mundo (ou tamanho em “metros virtuais”),
- UVs, cor, flags (mirrored, palette etc),
- e deixar o shader gerar os 4 vértices “sempre virados para a câmera” usando a view/projection do olho atual.

**Por que é o melhor no VR:**
- naturalmente estereoscópico (cada olho projeta de seu ponto de vista);
- elimina a dependência de `pViewport` e `Camera3D::Project` (que são flat/simétricos);
- evita inconsistências com depth buffer (o z vem do pipeline 3D normal).

**Custo/impacto:**
- mudança de representação: `RenderBillboardD3D` hoje guarda quads em screen-space; precisaria guardar dados em world-space (ou criar uma estrutura paralela “VR billboards”);
- revisão de sorting/transparência (ver “Pontos de atenção”).

### Caminho B (rápido, porém paliativo): recalcular billboards em screen-space por olho
**Ideia:** manter o modelo atual, mas:
- fazer `Project()` usando a projeção do OpenXR do olho atual (assimétrica),
- atualizar `pViewport`/parâmetros de projeção por olho,
- rodar `TransformBillboards()` para cada olho.

**Benefícios:**
- mexe menos no renderer de billboards a curto prazo.

**Riscos:**
- `Camera3D::Project` não suporta frustum assimétrico (não é só ajustar viewport).
- tende a virar um emaranhado de “modo VR” dentro de `Camera3D` e de sistemas que assumem a câmera flat.

### Caminho C (intermediário): “câmera ciclope” para CPU + render 3D no GPU
**Ideia:** calcular alguns dados em CPU usando uma câmera central (média dos olhos), apenas para:
- culling grosseiro,
- sorting aproximado,
e renderizar quads em world-space no GPU por olho.

É um meio-termo útil para preservar performance e reduzir complexidade de duplicar listas por olho.

## Pontos de atenção (onde o problema costuma morder)
1) **Transparência e ordenação por profundidade**
   - Billboards transparentes dependem de sorting por profundidade.
   - Em VR, sorting “por olho” pode diferir; o mais comum é aceitar sorting por câmera central (Caminho C) como aproximação.

2) **Coerência com depth buffer**
   - Hoje o pipeline usa ortho + `thisdepth` derivado de `view_space_z`:
     - [OpenGLRenderer::DoRenderBillboards_D3D](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Renderer/OpenGLRenderer.cpp#L1767-L1866)
   - No VR, isso fica ainda mais frágil porque a cena 3D é rasterizada com projeção OpenXR. Em world-space, o depth “cai no lugar” automaticamente.

3) **Separar “UI 2D” de “2D no mundo”**
   - UI deve continuar em overlay/head-locked (e provavelmente não participa desse bug).
   - “Assets 2D projetados no mundo” = sprites de atores/decorações/objetos, partículas e spell FX.

4) **Escala física no VR**
   - No desktop, tamanho do sprite em pixels faz sentido.
   - No VR, tamanho precisa ser consistente no mundo (e.g. metros virtuais) para evitar sensação de miniatura/gigantismo.

5) **Picking e interações**
   - Há código que usa quads em screen-space para testes (ex.: visibilidade/picking):
     - [Vis.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/Engine/Graphics/Vis.cpp) usa `RenderBillboardD3D` em múltiplos pontos.
   - Em VR, idealmente isso migraria para raycast (controlador) em world-space. No curto prazo, pode ser mantido separado do rendering.

## Estratégia incremental sugerida
1) **Identificar e isolar quais grupos estão “aleatórios” no headset**
   - Sprites de atores/decorações/objetos (via `AddBillboardIfVisible` → `TransformBillboard`).
   - Partículas (`MakeParticleBillboardAndPush`).
   - Spell FX (`SpellFX_Billboard::SpellFXProject`).

2) **Migrar primeiro o grupo mais visível (sprites “comuns”) para world-space (Caminho A/C)**
   - Manter o pipeline antigo para desktop.
   - Criar um caminho VR que não dependa de `Project()` em pixels.

3) **Depois migrar partículas e spell FX**
   - Esses tendem a ser mais numerosos e sensíveis a performance; vale manter culling/sorting aproximado.

## Critérios de validação (o que significa “resolvido”)
- O mesmo sprite/efeito aparece preso ao objeto correto no mundo ao mover a cabeça.
- O sprite mantém paralaxe correta (mudança sutil entre olhos) e não “flutua” em screen-space.
- A oclusão com geometria 3D fica plausível (não atravessa paredes de forma errática).

