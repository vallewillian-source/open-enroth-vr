# Análise da Implementação de Diálogos com NPCs (MM7 VR)

Este documento detalha como os diálogos com NPCs e camponeses estão implementados no motor OpenEnroth, com o objetivo de planejar a melhoria da experiência em Realidade Virtual.

## 1. Componentes Visuais do Diálogo

A interação com NPCs é composta por dois elementos principais que aparecem na tela 2D:

### A. Painel de Texto Inferior (Subtítulos/Mensagens)
*   **Classe Responsável:** `GUIWindow` (método `DrawDialoguePanel`).
*   **Comportamento:** Exibe a fala do NPC ou descrições de eventos.
*   **Renderização:**
    *   Usa uma textura de "couro" (`ui_leather_mm7`) como fundo.
    *   A altura é dinâmica, baseada no tamanho do texto.
    *   **Posicionamento:** É ancorado em `y=352` e cresce para cima. Começa em `x=8`.
    *   **Fontes:** Usa `pFontArrus` por padrão, mas alterna para `pFontCreate` (menor) se o texto for muito grande para o espaço disponível.
*   **Arquivos Relacionados:** [GUIWindow.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/GUIWindow.cpp#L337-L354).

### B. Menu Lateral Direito (Opções de Diálogo)
*   **Classes Responsáveis:** `GUIWindow_Dialogue` e `GUIWindow_House`.
*   **Comportamento:** Exibe o nome do NPC, retrato e uma lista de tópicos clicáveis (Quest, Hire, Bye, etc.).
*   **Renderização:**
    *   Desenha um frame de fundo (`game_ui_right_panel_frame`) e um fundo sólido (`game_ui_dialogue_background`).
    *   O retrato do NPC é desenhado no topo do painel direito.
    *   **Posicionamento:** O painel começa em `x=468` e vai até o fim da tela (`x=640`). Os botões de opção começam geralmente em `x=480`.
*   **Arquivos Relacionados:** [UIDialogue.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIDialogue.cpp) e [UIHouses.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIHouses.cpp).

## 2. Fluxo de Execução e Estados

O sistema alterna entre diferentes estados de tela (`SCREEN_TYPE`) dependendo do tipo de diálogo:

1.  **`SCREEN_NPC_DIALOGUE`**: Diálogo completo com opções (usado em cidades/mundo aberto).
2.  **`SCREEN_BRANCHLESS_NPC_DIALOG`**: Diálogos sem escolhas (saudações rápidas, frases de efeito de camponeses). Usa `GUIWindow_BranchlessDialogue`.
3.  **`SCREEN_HOUSE`**: Diálogos dentro de casas/lojas. A lógica é gerenciada por `GUIWindow_House`.

### Inicialização
*   A função `initializeNPCDialogue` em [UIDialogue.cpp](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIDialogue.cpp#L43) configura o ID do NPC, pausa os timers do jogo e prepara as texturas.
*   Os botões são criados dinamicamente com base nas "topics" disponíveis para aquele NPC.

## 3. Desafios e Oportunidades para o VR

Atualmente, o VR renderiza a tela 2D inteira como um overlay plano. Para uma experiência imersiva, podemos considerar:

*   **Isolamento do Texto:** Em vez de mostrar o "couro" 2D no fundo da visão, poderíamos extrair apenas a string do texto e renderizá-la em um painel flutuante (diegeticamente) próximo ao NPC ou fixo no campo de visão inferior do jogador (HUD VR).
*   **Menu de Opções:** O menu da direita poderia ser transformado em um menu flutuante 3D que o jogador aponta com o laser do controle.
*   **Retrato do NPC:** Em VR, o retrato é menos necessário se o jogador estiver olhando diretamente para o modelo 3D do NPC, mas pode ser útil para ver detalhes de NPCs que não possuem modelos únicos.

## 4. Referências de Código Úteis

*   **`GUIWindow::DrawDialoguePanel`**: [GUIWindow.cpp:L337](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/GUIWindow.cpp#L337) - Onde o fundo de couro e o texto inferior são desenhados.
*   **`GUIWindow_Dialogue::Update`**: [UIDialogue.cpp:L173](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIDialogue.cpp#L173) - Onde o painel lateral direito é montado.
*   **`GUIWindow_House::houseDialogManager`**: [UIHouses.cpp:L879](file:///c:/Users/Usuário/www/mm7_vr/src/GUI/UI/UIHouses.cpp#L879) - Gerenciamento de diálogo dentro de casas.
