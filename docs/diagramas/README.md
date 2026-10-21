# Diagramas do guia

Cada arquivo `.mmd` é uma fonte Mermaid editável; o `.svg` de mesmo nome é a versão vetorial utilizada no Markdown. O HTML incorpora os SVGs como dados, preservando a leitura offline. O botão **Ampliar diagrama** permite explorar figuras maiores, inclusive no celular; Esc fecha a ampliação.

Os diagramas ilustram conceitos do guia, não constituem um diagnóstico pronto da StreamFlow. As legendas descrevem as relações representadas. Árvores de diretórios e exemplos executáveis permanecem como texto.

## Atualização pelo professor

1. Edite a fonte `.mmd` e confira se setas, rótulos e limites de responsabilidade correspondem ao texto.
2. Renderize o SVG com Mermaid. Nesta revisão foi usado Mermaid 11.12.0 já instalado no Antigravity, com Chromium e Playwright, sem dependências adicionadas ao projeto.
3. Regenere o HTML a partir do Markdown e confira as figuras no computador e no celular.
4. Versione `.mmd`, `.svg`, Markdown e HTML juntos.

No workspace de ensino, os comandos compartilhados são:

```bash
node .agent/scripts/render_mermaid_local.cjs <pasta-diagramas> <pacote-playwright> <mermaid.min.js> <executavel-chromium>
node .agent/scripts/md_to_html.js <guia.md> <guia.html> --self-contained
```

Os argumentos apontam para ferramentas já instaladas no ambiente. O script de renderização bloqueia requisições de rede durante a geração. Kroki foi avaliado a partir do experimento existente no workspace, mas a geração final foi local.
