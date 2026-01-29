# Relatório — Plano para corrigir deslocamento no eixo Y dos assets 2D no VR (Solução B)

## Objetivo
Garantir que os assets 2D projetados (sprites/billboards/partículas/FX) permaneçam na posição correta no eixo Y **independente do pitch da cabeça**, com o mínimo de mudanças estruturais no OpenEnroth.

## Sintoma observado
- Com cabeça fixa olhando para frente, o posicionamento fica correto.
- Ao mover a cabeça para cima/baixo, os assets “escorregam” no eixo Y (sobem para o céu/descem para o chão), como se estivessem “presos à tela” parcialmente.
- O problema aparece mesmo com movimento da party funcionando e com a geometria 3D correta.

## Hipótese principal (mais provável)
O pipeline B recalcula o screen-space por olho, mas ainda usa **uma origem/projeção em screen-space que não acompanha o pitch do HMD** para o eixo Y. Isso indica que, em algum ponto do cálculo do `screenPos.y`, os valores:
- estão usando um **view/projection parcial** (ex.: só yaw ou uma câmera “flat”),
- ou usam um **center**/viewport que não está sincronizado com a matriz/projeção do olho,
- ou ainda usam um **viewspace** cujo eixo Y não está sendo afetado pela rotação do HMD.

Como resultado, quando o HMD inclina para cima/baixo, o mundo 3D é reprojetado corretamente, mas os sprites/billboards ficam “presos” em uma referência que não inclina junto.

## Alvo de correção mínima
Manter a Solução B, mas garantir que o cálculo de `screenPos.y` e escala em pixels **use exatamente a mesma view/projection do olho atual** (incluindo pitch), sem introduzir nova arquitetura ou reescrever o sistema em world-space.

## Plano de ação (mínimo de mudanças)

### Etapa 1 — Rastrear a origem do `screenPos.y` usado no VR
Identificar exatamente **qual câmera** e **quais matrizes** estão sendo usadas no cálculo de projeção em screen-space. O alvo é garantir que o cálculo do `y` incorpore o pitch do HMD.

Locais críticos (já conhecidos pelo relatório anterior):
- `BaseRenderer::AddBillboardIfVisible` → chama `ViewClip` e `Project` para obter `screenPos`.
- `Camera3D::Project` → baseia-se no `CreateViewMatrixAndProjectionScale` e em `pViewport`.
- `OpenGLRenderer::_set_3d_projection_matrix` → usa matriz por olho no VR.

Verificação mínima:
Confirmar se, durante o loop de renderização por olho, a câmera/projeção usada por `Camera3D::Project` é realmente atualizada com o pitch do HMD. Se não estiver, o `screenPos.y` vai ficar “ancorado”.

### Etapa 2 — Aplicar o pitch do HMD diretamente no cálculo de projeção usado por `Project`
Se a projeção para screen-space não estiver recebendo o pitch do HMD, aplicar **a rotação do HMD no view** usado por `Camera3D::Project` **somente no caminho VR**.

Intervenção mínima sugerida:
- Introduzir um caminho VR no `Camera3D` (ou um override temporário) que:
  - multiplique o `viewmat` atual pelo quaternion/rotação do HMD **antes do Project**, ou
  - injete o pitch do HMD na construção do view matrix para o olho atual.

Resultado esperado:
`Project(viewspace)` passa a “subir/descer” junto com o HMD, fixando a posição vertical correta dos sprites.

### Etapa 3 — Garantir consistência de `pViewport.center()`/scale com o olho atual
Na solução B, é comum recalcular `pViewport` ou `ViewPlaneDistPixels` para cada olho. Se o Y do `screenPos` ainda deslocar após Etapa 2, a causa provável é **desalinhamento do centro do viewport** com a projeção do olho.

Intervenção mínima:
- No início do render por olho, setar explicitamente o **viewport de projeção** que `Camera3D::Project` usa:
  - `pViewport` e `ViewPlaneDistPixels` devem ser atualizados com base no **tamanho e FOV** do eye, não no desktop.

Resultado esperado:
`screenPos.y` volta a ser consistente quando o HMD inclina, pois a projeção passa a usar o mesmo centro/scale da cena 3D.

## Checagens rápidas (para confirmar a causa sem refatorar)
1. Fixar o pitch do HMD em 0 (ignorar rotação vertical) e observar:
   - Se o problema some, então o erro está no uso do pitch no cálculo de `Project`.
2. Forçar o mesmo pitch da party (flat) na matriz de view usada por `Project`:
   - Se os assets “grudam” na posição correta, confirma que o `Project` não estava sincronizado com HMD.

## Recomendações de implementação mínima (sem mudanças grandes)
- Não migrar para world-space ainda; manter pipeline atual.
- Atualizar somente o cálculo da view/projection usado por `Camera3D::Project` no VR, garantindo que o eixo Y reflita o pitch do HMD.
- Manter o culling e sorting já existente, apenas corrigindo a base de projeção.

## Critério de sucesso (somente eixo Y)
- Ao mover a cabeça para cima/baixo, os sprites permanecem fixos no objeto correto do mundo.
- Não há deslocamento vertical “em bloco” dos assets.
- A geometria 3D continua alinhada e o efeito não altera o eixo X.

## Observação final
Esse plano mantém a solução B e reduz o risco de mudanças estruturais. Se, após isso, o problema persistir, o diagnóstico indica que o pipeline screen-space está intrinsicamente incompatível com VR e será necessário migrar para world-space (Caminho A/C). Porém, antes disso, a sincronização de pitch no `Project` é a correção mínima e mais promissora.
