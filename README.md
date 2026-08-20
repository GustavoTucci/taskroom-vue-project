# Taskroom

> Um workspace Kanban leve para transformar ideias em tarefas e acompanhar o
> progresso do projeto com clareza.

O Taskroom é uma aplicação web responsiva construída para organizar o trabalho
em três etapas: **A fazer**, **Em andamento** e **Concluído**. A interface
combina uma visão rápida do progresso com ações essenciais de gerenciamento,
sem depender de backend ou autenticação.

## Visão geral

| Item | Detalhe |
| --- | --- |
| Tipo | Aplicação web de gerenciamento de tarefas |
| Interface | Vue 3 com Composition API |
| Linguagem | TypeScript |
| Bundler | Vite |
| Persistência | `localStorage` do navegador |

## Recursos

- Criar tarefas com título, descrição, etiqueta e prioridade.
- Reordenar o fluxo de trabalho arrastando tarefas entre as colunas.
- Pesquisar por título, descrição ou etiqueta.
- Filtrar por prioridade alta, média ou baixa.
- Excluir tarefas individualmente.
- Alternar entre tema claro e escuro, com preferência persistida.
- Personalizar a cor de destaque do quadro.
- Abrir os detalhes de uma tarefa em modal.
- Arrastar tarefas com mouse, toque e pointer events em dispositivos móveis.
- Exibir carregamento inicial e mensagens de feedback após ações.
- Acompanhar a quantidade de tarefas concluídas.
- Persistir alterações automaticamente no navegador.
- Usar o quadro em telas grandes ou dispositivos móveis.

## Stack

- [Vue 3](https://vuejs.org/) para a camada de interface e reatividade.
- [TypeScript](https://www.typescriptlang.org/) para tipagem estática.
- [Vite](https://vite.dev/) para desenvolvimento e build de produção.
- `vue-tsc` para verificação de tipos em arquivos `.vue`.
- CSS nativo para o layout, responsividade e identidade visual.

## Pré-requisitos

- Node.js `22.18+` ou `24.12+`.
- npm instalado e disponível no terminal.

## Instalação

Clone o repositório, entre na pasta do projeto e instale as dependências:

```sh
git clone <url-do-repositorio>
cd first-vue-project
npm install
```

## Desenvolvimento

Inicie o servidor local com hot reload:

```sh
npm run dev
```

O Vite exibirá no terminal a URL da aplicação, normalmente
[`http://localhost:5173`](http://localhost:5173).

## Scripts

| Comando | Finalidade |
| --- | --- |
| `npm run dev` | Inicia o servidor de desenvolvimento. |
| `npm run type-check` | Executa a verificação de tipos do projeto. |
| `npm run build` | Verifica os tipos e gera o build de produção. |
| `npm run preview` | Serve localmente o build gerado para conferência. |

## Estrutura do projeto

```text
first-vue-project/
├── public/            # Arquivos estáticos públicos
├── src/
│   ├── App.vue        # Interface, estado e regras do quadro
│   └── main.ts        # Inicialização da aplicação Vue
├── index.html         # Documento HTML de entrada
├── package.json       # Dependências e scripts
├── tsconfig*.json     # Configurações do TypeScript
└── vite.config.ts     # Configuração do Vite
```

## Persistência de dados

As tarefas são salvas localmente no navegador sob a chave
`vue-kanban-tasks`. Os dados não são sincronizados entre dispositivos e são
perdidos quando o armazenamento local do site é removido.

Para começar novamente com as tarefas de exemplo, remova essa chave nas
ferramentas de desenvolvimento do navegador e recarregue a página.

## Build de produção

Gere os arquivos otimizados para publicação com:

```sh
npm run build
```

O resultado será criado na pasta `dist/`. Para visualizar esse resultado
localmente antes de publicar:

```sh
npm run preview
```
