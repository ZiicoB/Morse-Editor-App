# Morsejs Editor para Windows (Aplicativo)

Esta pasta transforma o [Editor-Offline](../Editor-Offline/README.md) num programa de Windows com instalador. Quem instala recebe um atalho na área de trabalho e no menu Iniciar e usa o editor numa janela própria, sem navegador, sem terminal e sem internet.

O Aplicativo leva sua própria cópia do editor (`dist/`). Depois de instalado, ele não depende da pasta Editor-Offline nem de nenhum servidor. O editor é carregado pelo endereço interno `app://editor/`, e o rascunho fica salvo no próprio app, em `%APPDATA%\Morsejs Editor`.

## Gerar o instalador

Na sua máquina, com Node.js 20 ou mais recente:

```
cd Aplicativo
npm install
npm run dist
```

`npm install` precisa de internet uma única vez, para baixar o Electron e o electron-builder. `npm run dist` copia a versão atual do Editor-Offline e gera o instalador em `Aplicativo/release/`, com um nome parecido com `Morse-Js-Editor-Setup-1.0.6.exe`. Esse arquivo é o que você entrega: a pessoa abre, escolhe a pasta de instalação e usa o atalho criado.

Para publicar a atualização manualmente no GitHub, anexe o instalador, seu arquivo `.exe.blockmap` e o `latest.yml` da mesma compilação. Mantenha os nomes dos arquivos: o `latest.yml` usa o nome com hífens para localizar o instalador.

Se a compilação falhar depois de aumentar a versão, repita apenas `npm run dist`. Os comandos `release:patch`, `release:minor` e `release:major` aumentam a versão novamente a cada execução.

No Windows, o hook `build/nsis-spawn.cjs` preserva as variáveis de ambiente e faz até quatro novas tentativas se o gerador do desinstalador falhar com `spawn UNKNOWN` antes de iniciar. Outros erros continuam interrompendo a compilação normalmente.

Para testar sem gerar o instalador, use `npm start`, que abre a janela direto. `npm run pack` gera só a pasta do programa (`release/win-unpacked`), útil para conferir o resultado antes do instalador completo.

Sempre que você alterar o Editor-Offline, os comandos `npm start`, `npm run pack` e `npm run dist` copiam a versão nova automaticamente. Para copiar sem abrir nada, use `npm run sync`.

Os ícones vêm de `Icones/` na raiz do repositório: `assets/icon.ico` e `assets/titlebar-icon.png` são gerados a partir de `Icones/morsejs.png` e aparecem no instalador, no atalho que abre o aplicativo e na barra de título; `assets/taskbar-icon.png` vem de `Icones/MorseJS_0.png` e aparece só na barra de tarefas do Windows enquanto o app está aberto. A barra de título tem 36 px, a mesma altura da do Codex: a barra nativa fica escondida, o Windows desenha só os botões de minimizar, maximizar e fechar, e [electron-titlebar.cjs](electron-titlebar.cjs) desenha o ícone e o nome na mesma faixa, acompanhando o tema claro ou escuro. Como o instalador não é assinado digitalmente, o Windows pode mostrar o aviso do SmartScreen na primeira execução ("Mais informações" → "Executar assim mesmo").

## Corretor ortográfico com e sem internet

No Windows, o corretor do Electron usa dicionários Hunspell (arquivos `.bdic`) e mostra as sugestões no menu "Ortografia" do próprio editor, ao clicar com o botão direito numa palavra sublinhada.

- **Com internet**, o corretor funciona sem nenhum ajuste. Na primeira vez, o Electron baixa o dicionário pt-BR e o guarda para as próximas.
- **Sem internet**, ele só funciona se o dicionário já estiver no computador. Para isso, coloque o arquivo pt-BR `.bdic` em `resources/dictionaries/` antes de rodar `npm run dist`. O instalador leva o arquivo junto, e o app o entrega ao Electron por um servidor interno (`127.0.0.1`), sem acessar a internet. O passo a passo está em [resources/dictionaries/README.md](resources/dictionaries/README.md).

Se a pasta de dicionários estiver vazia, o app volta ao comportamento padrão: baixa o dicionário quando houver internet.

## Como o app é montado

- [main.js](main.js) cria a janela, registra o endereço `app://`, remove a barra de menus do Windows, abre links externos no navegador padrão e mantém uma única janela aberta (abrir o atalho de novo traz a janela existente para frente). A janela liga o leitor de PDF embutido do Chromium (`plugins: true`), usado para abrir PDFs somente para leitura; links clicados dentro de um PDF abrem no navegador padrão. O atalho Ctrl+G é capturado aqui (`before-input-event` em todos os conteúdos da janela) e enviado ao editor, para abrir a pesquisa mesmo com o foco dentro do PDF.
- [app-protocol.cjs](app-protocol.cjs) entrega os arquivos de `dist/` e bloqueia qualquer caminho fora dela.
- [dictionary-server.cjs](dictionary-server.cjs) entrega somente os `.bdic` de `resources/dictionaries/`.
- [electron-spelling.cjs](electron-spelling.cjs) e [electron-preload.cjs](electron-preload.cjs) levam a palavra errada e as sugestões do Chromium até o menu do editor e aplicam a sugestão escolhida.
- [sync-editor.mjs](sync-editor.mjs) copia `Editor-Offline/dist` e as licenças de terceiros.
- [update-install.cjs](update-install.cjs) aguarda a confirmação de início do instalador antes de fechar o editor, tanto ao confirmar a atualização quanto ao fechar normalmente. Usa instalação silenciosa na pasta do aplicativo atual e solicita reabertura somente quando a pessoa clica em atualizar. Se o Windows não conseguir iniciar o instalador, mantém o editor aberto e informa o erro.

Quem estiver numa versão anterior à 1.0.7 com falha ao aplicar atualizações deve instalar o pacote 1.0.7 manualmente uma vez para receber essa correção.

## Testes

```
cd Aplicativo
npm test
```

Os testes cobrem o endereço `app://` (tipos de arquivo, arquivos ausentes, tentativa de sair da pasta), o servidor de dicionários, a ordem de inicialização de `main.js` com um Electron simulado (o endereço do dicionário é definido antes de ativar o pt-BR), o bloqueio de navegação para fora do editor e a ponte do corretor. Eles não abrem uma janela Electron de verdade: o teste final é rodar `npm start` e o instalador na sua máquina.

Para os PDFs, planilhas e apresentações há um teste de ponta a ponta que abre o editor de verdade numa janela Electron oculta:

```
cd Aplicativo
npm run test:office
```

Ele cria, edita, salva e reabre planilhas e apresentações (inclusive com fórmulas compartilhadas, tabelas do Excel, gráficos e grupos), confere que cada pacote salvo tem XML válido, tipos de conteúdo e relações coerentes, que as partes não editadas ficam idênticas byte a byte, o desfazer, a volta do rascunho depois de recarregar e a abertura de um PDF, e que o .docx continua igual. Para testar arquivos seus, informe `MORSE_E2E_FILES="caminho1;caminho2"`; `MORSE_E2E_SHOTS=pasta` salva capturas de tela.
