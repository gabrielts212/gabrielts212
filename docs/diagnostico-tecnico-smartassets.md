# Diagnóstico Técnico — SmartAssets / Smart Check

| Campo        | Valor                                              |
|--------------|----------------------------------------------------|
| **Projeto**  | SmartAssets / Smart Check                          |
| **Data**     | 2026-03-06                                         |
| **Analista** | Diagnóstico Técnico Automatizado                   |
| **Escopo**   | Frontend (Vite + React + TypeScript + Tailwind + ESLint) + Referência de integração Backend (.NET) |
| **Versão**   | 1.0                                                |

---

## Sumário

1. [Escopo e Premissas](#1-escopo-e-premissas)
2. [Visão Geral da Arquitetura](#2-visão-geral-da-arquitetura)
3. [Frontend — Estrutura e Organização](#3-frontend--estrutura-e-organização)
4. [Frontend — Dependências e Tooling](#4-frontend--dependências-e-tooling)
5. [Frontend — Variáveis de Ambiente](#5-frontend--variáveis-de-ambiente)
6. [Frontend — Configuração de Ferramentas](#6-frontend--configuração-de-ferramentas)
7. [Frontend — Padrões de Código e Arquitetura](#7-frontend--padrões-de-código-e-arquitetura)
8. [Integração com Backend e Assets (Azure Files + SAS)](#8-integração-com-backend-e-assets-azure-files--sas)
9. [CI/CD e Deploy](#9-cicd-e-deploy)
10. [Riscos e Pontos de Atenção](#10-riscos-e-pontos-de-atenção)
11. [Recomendações Priorizadas](#11-recomendações-priorizadas)
12. [Referência de Endpoints do Backend](#12-referência-de-endpoints-do-backend)
13. [Execução Local](#13-execução-local)
14. [Exportação para PDF](#14-exportação-para-pdf)

---

## 1. Escopo e Premissas

### 1.1 Objetivo
Este documento é um **diagnóstico técnico consultivo** do projeto SmartAssets / Smart Check. Tem caráter analítico e não prevê alterações de funcionalidades existentes.

### 1.2 Premissas
- O **frontend** está implementado com Vite, React 18, TypeScript, React Router, Tailwind CSS e ESLint.
- O **backend** é uma API .NET hospedada separadamente; este documento documenta a integração a partir do que o frontend consome.
- Os assets de imagem são armazenados no **Azure Files** e acessados via URL com token SAS.
- O deploy do frontend é realizado em ambiente Windows (IIS ou Azure App Service Windows) com `web.config`.
- O pipeline de CI/CD usa **Azure Pipelines** (`azure-pipelines.frontend.yml`).

### 1.3 Fora do Escopo
- Alterações de código de produção.
- Análise interna do código-fonte do backend .NET.
- Configuração de infraestrutura de banco de dados (SQL Server).

---

## 2. Visão Geral da Arquitetura

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USUÁRIO (Navegador)                             │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ HTTPS
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              FRONTEND  (React 18 + Vite + TypeScript)                    │
│                                                                          │
│  ┌──────────────┐  ┌─────────────────┐  ┌───────────────────────────┐  │
│  │  React Router│  │  Componentes UI │  │  Tailwind CSS (estilos)   │  │
│  │  (rotas SPA) │  │  (features +    │  │  + globals.css            │  │
│  └──────────────┘  │   shared)       │  └───────────────────────────┘  │
│                    └─────────────────┘                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              src/shared/api/apiClient.ts                        │   │
│  │   axios/fetch wrapper → /api/* (proxy Vite em DEV)             │   │
│  │                       → VITE_API_BASE_URL (PROD)               │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
│                                 │                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              src/shared/lib/assets.ts                           │   │
│  │   URL = VITE_ASSETS_BASE_URL + caminho + ? + VITE_ASSETS_SAS   │   │
│  └──────────────────────────────┬──────────────────────────────────┘   │
└────────────────────────────────┬┴──────────────────────────────────────┘
                                 │                       │
                    REST / JSON  │                       │ HTTPS (SAS URL)
                                 ▼                       ▼
┌───────────────────────────┐       ┌─────────────────────────────────────┐
│   Backend API (.NET)      │       │  Azure Files (storage de imagens)   │
│   /api/*                  │       │  /images/*  (containers de assets)  │
│   (SQL Server, regras)    │       └─────────────────────────────────────┘
└───────────────────────────┘

CI/CD: Azure Pipelines → Build Vite → Artefato dist/ → Deploy IIS/App Service (Windows)
```

### 2.1 Fluxos Principais

| Fluxo | Caminho |
|-------|---------|
| Autenticação | Browser → apiClient → `POST /api/auth/*` → Backend .NET |
| Listagem de ativos | Browser → apiClient → `GET /api/assets` → Backend .NET → SQL Server |
| Visualização de imagem | Browser → assets.ts → Azure Files (URL + SAS token) |
| Geocodificação / Mapas | Browser → Google Maps JS API (chave `VITE_GOOGLE_MAPS_API_KEY`) |

---

## 3. Frontend — Estrutura e Organização

### 3.1 Estrutura de Pastas (convenção)

```
/                               ← raiz do projeto
├── index.html                  ← entrypoint HTML do Vite
├── vite.config.ts              ← configuração do Vite (alias, proxy, plugins)
├── tsconfig.json               ← configuração TypeScript
├── tsconfig.node.json          ← TS para scripts do Vite/Node
├── eslint.config.js / .eslintrc← configuração ESLint
├── tailwind.config.ts/js       ← configuração Tailwind CSS
├── postcss.config.js           ← PostCSS (autoprefixer + tailwind)
├── package.json                ← dependências e scripts NPM
├── azure-pipelines.frontend.yml← pipeline CI/CD (Azure DevOps)
├── web.config                  ← reescrita de rotas SPA (IIS)
├── public/                     ← assets estáticos públicos (favicon, etc.)
└── src/
    ├── main.tsx                ← entrypoint React (ReactDOM.createRoot)
    ├── App.tsx                 ← componente raiz (Router, providers globais)
    ├── shared/                 ← código compartilhado entre features
    │   ├── api/
    │   │   └── apiClient.ts    ← cliente HTTP centralizado
    │   ├── lib/
    │   │   └── assets.ts       ← helper de URLs de assets (Azure Files + SAS)
    │   ├── styles/
    │   │   └── globals.css     ← estilos globais + diretivas Tailwind
    │   ├── components/         ← componentes UI reutilizáveis (Button, Modal…)
    │   ├── hooks/              ← hooks customizados reutilizáveis
    │   └── types/              ← tipos/interfaces TypeScript globais
    └── features/               ← módulos de domínio (feature-based)
        ├── auth/               ← autenticação (login, logout, guards)
        ├── assets/             ← gestão de ativos (listagem, detalhes, forms)
        ├── dashboard/          ← painel principal
        ├── maps/               ← integração Google Maps
        └── …/                 ← outras features
```

### 3.2 Entrypoints

| Arquivo | Papel |
|---------|-------|
| `index.html` | Único HTML servido pelo Vite; contém `<div id="root">` e o script `src/main.tsx` |
| `src/main.tsx` | Inicializa React (`ReactDOM.createRoot`), monta `<App />` |
| `src/App.tsx` | Define `<BrowserRouter>`, providers de contexto globais e as rotas de alto nível |

### 3.3 Roteamento (React Router v6)

O roteamento é declarativo com `<Routes>` e `<Route>` no `App.tsx` ou em arquivos de rotas das features. Padrões esperados:

```tsx
// Exemplo de estrutura de rotas em App.tsx
<BrowserRouter>
  <Routes>
    <Route path="/login"    element={<LoginPage />} />
    <Route element={<PrivateRoute />}>          {/* guard de autenticação */}
      <Route path="/"         element={<Dashboard />} />
      <Route path="/assets"   element={<AssetsPage />} />
      <Route path="/assets/:id" element={<AssetDetail />} />
      <Route path="/maps"     element={<MapsPage />} />
    </Route>
    <Route path="*"         element={<NotFound />} />
  </Routes>
</BrowserRouter>
```

O alias `@` aponta para `src/` (`"@/*": ["src/*"]` em `tsconfig.json` e `resolve.alias` em `vite.config.ts`), permitindo imports como `import { apiClient } from '@/shared/api/apiClient'`.

---

## 4. Frontend — Dependências e Tooling

### 4.1 Stack Principal

| Pacote | Versão estimada | Papel |
|--------|----------------|-------|
| `react` | ^18.x | Biblioteca de UI |
| `react-dom` | ^18.x | Renderização no DOM |
| `react-router-dom` | ^6.x | Roteamento SPA |
| `typescript` | ^5.x | Tipagem estática |
| `vite` | ^5.x | Bundler/dev server |
| `tailwindcss` | ^3.x | Framework CSS utilitário |
| `@vitejs/plugin-react` | ^4.x | Plugin React para Vite (Fast Refresh) |
| `autoprefixer` | ^10.x | PostCSS: prefixos CSS automáticos |
| `postcss` | ^8.x | Processador CSS (pipeline Tailwind) |
| `eslint` | ^8-9.x | Linter de código |
| `@typescript-eslint/*` | ^6-7.x | Regras ESLint para TypeScript |
| `eslint-plugin-react-hooks` | ^4.x | Regras para hooks React |

### 4.2 Scripts NPM Esperados

```jsonc
// package.json (scripts típicos)
{
  "scripts": {
    "dev":     "vite",                    // servidor de desenvolvimento (HMR)
    "build":   "tsc && vite build",       // checagem de tipos + build de produção
    "preview": "vite preview",            // pré-visualização do build
    "lint":    "eslint src --ext ts,tsx"  // execução do linter
  }
}
```

> **Nota sobre versão de Node:** Recomenda-se **Node 18 LTS** (ou 20 LTS) para compatibilidade com Vite 5 e as dependências TypeScript.

### 4.3 Verificação de Compatibilidade

- `vite@5` requer Node ≥ 18.
- Pacotes `@types/react` e `@types/react-dom` devem ser compatíveis com a versão de React instalada.
- A versão do `typescript` no `package.json` deve corresponder à do `tsconfig.json` (`"target"` e `"lib"`).

---

## 5. Frontend — Variáveis de Ambiente

As variáveis de ambiente do Vite são prefixadas com `VITE_` para serem expostas ao bundle do browser. São definidas em arquivos `.env.*` na raiz do projeto.

### 5.1 Tabela de Variáveis

| Variável | Uso | Ambiente |
|----------|-----|----------|
| `VITE_API_PROXY_TARGET` | URL alvo do proxy do Vite em desenvolvimento (ex.: `http://localhost:5000`) | Desenvolvimento |
| `VITE_DOTNET_API_BASE` | Base URL da API .NET usada internamente (pode ser diferente da URL pública) | Todos |
| `VITE_API_BASE_URL` | URL base da API consumida pelo `apiClient.ts` em produção | Produção |
| `VITE_ASSETS_BASE_URL` | URL base do container Azure Files para assets/imagens | Todos |
| `VITE_ASSETS_SAS` | Token SAS (Shared Access Signature) para autenticação no Azure Files | Todos |
| `VITE_GOOGLE_MAPS_API_KEY` | Chave da Google Maps JavaScript API | Todos |
| `VITE_GOOGLE_MAPS_LIBRARIES` | Bibliotecas do Google Maps a carregar (ex.: `"places,geometry"`) | Todos |

### 5.2 Arquivos `.env.*`

```
.env                  ← valores padrão (sem segredos)
.env.development      ← overrides locais (não commitado)
.env.production       ← valores de produção (sem segredos; use pipeline para secrets)
.env.local            ← overrides pessoais (não commitado, ignorado no git)
```

> **⚠️ Segurança:** Qualquer valor em `VITE_*` é **incorporado estaticamente ao bundle JS** gerado e fica exposto no código-fonte do browser. **Nunca** coloque secrets de acesso a banco de dados, chaves de API com permissões amplas ou tokens de longa duração em variáveis `VITE_*`.

### 5.3 Riscos de Exposição

| Variável | Risco | Mitigação |
|----------|-------|-----------|
| `VITE_ASSETS_SAS` | Token SAS exposto no bundle → acesso não autorizado ao storage | Usar tokens SAS de curta duração + gerar via backend on-demand |
| `VITE_GOOGLE_MAPS_API_KEY` | Chave exposta → uso indevido com custos financeiros | Restringir a chave por domínio/referrer no Google Cloud Console |
| `VITE_API_BASE_URL` | Revelar URL interna da API | Usar URL pública; proteger endpoints no backend com autenticação |

### 5.4 Acesso no Código

```typescript
// Forma correta de acessar variáveis Vite no TypeScript
const apiBase = import.meta.env.VITE_API_BASE_URL;
const sasToken = import.meta.env.VITE_ASSETS_SAS;
```

Para tipagem correta, declarar em `src/vite-env.d.ts`:

```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_PROXY_TARGET: string;
  readonly VITE_DOTNET_API_BASE: string;
  readonly VITE_API_BASE_URL: string;
  readonly VITE_ASSETS_BASE_URL: string;
  readonly VITE_ASSETS_SAS: string;
  readonly VITE_GOOGLE_MAPS_API_KEY: string;
  readonly VITE_GOOGLE_MAPS_LIBRARIES: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## 6. Frontend — Configuração de Ferramentas

### 6.1 Vite (`vite.config.ts`)

```typescript
// vite.config.ts — estrutura esperada
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [react()],

    resolve: {
      alias: {
        '@': path.resolve(__dirname, './src'), // alias @ → src/
      },
    },

    server: {
      proxy: {
        '/api': {
          target: env.VITE_API_PROXY_TARGET,  // ex.: http://localhost:5000
          changeOrigin: true,
          secure: false,
        },
      },
    },

    build: {
      outDir: 'dist',
      sourcemap: false,  // desabilitar em produção para não expor código
    },
  };
});
```

**Pontos de atenção:**
- O proxy `/api` só funciona em desenvolvimento (`npm run dev`). Em produção, `VITE_API_BASE_URL` deve apontar para o backend diretamente ou via reverse proxy no IIS/App Service.
- `sourcemap: true` em produção expõe código-fonte. Recomenda-se `false` ou `hidden`.

### 6.2 TypeScript (`tsconfig.json`)

```jsonc
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["./src/*"]   // alias para imports absolutos
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### 6.3 Tailwind CSS (`tailwind.config.ts`)

```typescript
// tailwind.config.ts — estrutura esperada
import type { Config } from 'tailwindcss';

export default {
  content: [
    './index.html',
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      // Customizações de cores, fontes, espaçamentos do SmartAssets
    },
  },
  plugins: [],
} satisfies Config;
```

**Globals CSS (`src/shared/styles/globals.css`):**

```css
/* src/shared/styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Estilos globais customizados */
:root {
  /* variáveis CSS customizadas */
}
```

Este arquivo deve ser importado uma única vez em `src/main.tsx`:

```typescript
import './shared/styles/globals.css';
```

### 6.4 ESLint

Configuração típica para React 18 + TypeScript:

```jsonc
// .eslintrc.cjs ou eslint.config.js
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react-hooks/recommended"
  ],
  "parser": "@typescript-eslint/parser",
  "rules": {
    "react-refresh/only-export-components": ["warn", { "allowConstantExport": true }],
    "@typescript-eslint/no-explicit-any": "warn"
  }
}
```

---

## 7. Frontend — Padrões de Código e Arquitetura

### 7.1 Arquitetura Feature-Based

O projeto adota uma organização por **features/domínios**, o que melhora a coesão e facilita escalabilidade:

```
src/features/
  auth/
    components/    ← componentes específicos de auth (LoginForm, etc.)
    hooks/         ← useAuth, useAuthGuard
    services/      ← authService.ts (chamadas à API)
    types/         ← User, AuthToken, etc.
    index.ts       ← barrel export
  assets/
    components/
    hooks/
    services/      ← assetsService.ts
    types/
    index.ts
```

### 7.2 Cliente HTTP (`src/shared/api/apiClient.ts`)

O arquivo centraliza todas as chamadas HTTP. Padrão esperado com Axios ou fetch nativo:

```typescript
// src/shared/api/apiClient.ts — padrão esperado
import axios from 'axios';

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor de request: adicionar token de autenticação
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token'); // ou cookie/sessionStorage
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Interceptor de response: tratamento de erros globais
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // redirecionar para login
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

**Pontos de atenção:**
- O `baseURL` usa `VITE_API_BASE_URL`. Em desenvolvimento, as chamadas `/api/*` são interceptadas pelo proxy do Vite.
- Armazenar tokens no `localStorage` é conveniente mas vulnerável a XSS. Considerar `httpOnly cookies` para maior segurança.

### 7.3 Helper de Assets (`src/shared/lib/assets.ts`)

```typescript
// src/shared/lib/assets.ts — padrão esperado
const ASSETS_BASE_URL = import.meta.env.VITE_ASSETS_BASE_URL;
const ASSETS_SAS = import.meta.env.VITE_ASSETS_SAS;

/**
 * Gera a URL completa de um asset no Azure Files com token SAS.
 * @param filePath  caminho relativo do arquivo no container (ex.: "images/asset123.jpg")
 * @returns URL completa com token SAS
 */
export function getAssetUrl(filePath: string): string {
  const base = ASSETS_BASE_URL.endsWith('/')
    ? ASSETS_BASE_URL
    : `${ASSETS_BASE_URL}/`;
  const sas = ASSETS_SAS.startsWith('?') ? ASSETS_SAS : `?${ASSETS_SAS}`;
  return `${base}${filePath}${sas}`;
}
```

### 7.4 Componentes

Convenções recomendadas (e verificar se o projeto as segue):

- Componentes como **function components** com TypeScript (sem class components).
- Props tipadas com `interface` ou `type`.
- Componentes de UI reutilizáveis em `src/shared/components/`.
- Componentes de domínio em `src/features/<feature>/components/`.
- Exportações nomeadas preferencialmente (`export function Button`, não `export default`).

---

## 8. Integração com Backend e Assets (Azure Files + SAS)

### 8.1 Proxy de Desenvolvimento (Vite)

Em desenvolvimento (`npm run dev`), o Vite redireciona requisições `/api/*` para o backend local:

```
Browser → localhost:5173/api/assets → Vite Proxy → http://localhost:5000/api/assets
```

Isso evita problemas de CORS durante o desenvolvimento local.

### 8.2 Produção

Em produção, o frontend é um bundle estático servido pelo IIS/App Service. O `apiClient.ts` usa `VITE_API_BASE_URL` como base, que deve ser configurada para a URL pública do backend .NET.

**CORS no Backend:** O backend .NET deve configurar CORS para permitir requisições do domínio do frontend:

```csharp
// Program.cs / Startup.cs — .NET (referência)
builder.Services.AddCors(options =>
{
    options.AddPolicy("FrontendPolicy", policy =>
    {
        policy.WithOrigins("https://smartassets.dominio.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});
```

### 8.3 Azure Files — SAS Token

O `VITE_ASSETS_SAS` é um token SAS do Azure Storage que autoriza acesso de leitura aos arquivos. A URL completa segue o padrão:

```
https://<storage-account>.file.core.windows.net/<share>/<path>?sv=2022-11-02&ss=f&srt=o&sp=r&se=<expiry>&st=<start>&spr=https&sig=<signature>
```

**Fluxo de acesso a imagens:**

```
React Component
  → getAssetUrl("photos/asset-123.jpg")
  → "https://myaccount.file.core.windows.net/smartassets/photos/asset-123.jpg?sv=...&sig=..."
  → Azure Files (validação SAS)
  → Retorno da imagem
```

**⚠️ Risco de segurança:** O token SAS fica embutido no bundle JS. Recomenda-se:
1. Token SAS com **validade curta** (horas, não meses).
2. Permissões **mínimas** (somente leitura, somente o container necessário).
3. Idealmente, gerar tokens SAS **on-demand** via endpoint autenticado do backend (`GET /api/assets/sas-token`), evitando expor o token no bundle.

### 8.4 Google Maps

```typescript
// Uso típico no componente de mapas
const { isLoaded } = useJsApiLoader({
  googleMapsApiKey: import.meta.env.VITE_GOOGLE_MAPS_API_KEY,
  libraries: import.meta.env.VITE_GOOGLE_MAPS_LIBRARIES.split(','),
});
```

---

## 9. CI/CD e Deploy

### 9.1 Azure Pipelines (`azure-pipelines.frontend.yml`)

Estrutura esperada do pipeline:

```yaml
# azure-pipelines.frontend.yml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'   # ou windows-latest

variables:
  nodeVersion: '18.x'
  buildDir: 'dist'

stages:
  - stage: Build
    jobs:
      - job: BuildFrontend
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - script: npm ci
            displayName: 'Instalar dependências'

          - script: npm run build
            displayName: 'Build de produção'
            env:
              VITE_API_BASE_URL:          $(VITE_API_BASE_URL)
              VITE_ASSETS_BASE_URL:       $(VITE_ASSETS_BASE_URL)
              VITE_ASSETS_SAS:            $(VITE_ASSETS_SAS)
              VITE_GOOGLE_MAPS_API_KEY:   $(VITE_GOOGLE_MAPS_API_KEY)
              VITE_GOOGLE_MAPS_LIBRARIES: $(VITE_GOOGLE_MAPS_LIBRARIES)
              VITE_DOTNET_API_BASE:       $(VITE_DOTNET_API_BASE)

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: $(buildDir)
              ArtifactName: 'frontend-dist'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - deployment: DeployToIIS
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                # Deploy para IIS via Web Deploy ou cópia de arquivos
                - task: IISWebAppDeploymentOnMachineGroup@0
                  inputs:
                    WebSiteName: 'SmartAssets'
                    Package: '$(Pipeline.Workspace)/frontend-dist'
```

### 9.2 Variáveis do Pipeline

As variáveis `VITE_*` são definidas como **variáveis de pipeline secretas** no Azure DevOps (não em arquivos commitados), garantindo que secrets não sejam expostos no repositório.

No momento do `npm run build`, o Vite as recebe via `process.env` e as incorpora ao bundle gerado.

### 9.3 `web.config` (IIS — SPA Fallback)

Para que o React Router funcione corretamente em produção (SPA), o IIS precisa redirecionar todas as rotas para `index.html`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="React Router" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>
    <staticContent>
      <mimeMap fileExtension=".woff2" mimeType="font/woff2" />
      <mimeMap fileExtension=".webp" mimeType="image/webp" />
    </staticContent>
    <httpProtocol>
      <customHeaders>
        <add name="X-Content-Type-Options" value="nosniff" />
        <add name="X-Frame-Options" value="SAMEORIGIN" />
        <add name="X-XSS-Protection" value="1; mode=block" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
```

### 9.4 Fluxo Completo de Deploy

```
Desenvolvedor → git push → Azure Repos
    ↓
Azure Pipelines (trigger automático)
    ↓
npm ci → npm run build (com variáveis de ambiente injetadas)
    ↓
Artefato: dist/ (HTML + JS + CSS otimizados)
    ↓
Deploy → IIS / Azure App Service Windows
    ↓
Usuário acessa https://smartassets.dominio.com
```

---

## 10. Riscos e Pontos de Atenção

### 10.1 Segurança

| # | Risco | Impacto | Probabilidade |
|---|-------|---------|---------------|
| R1 | Token SAS exposto no bundle JS | Alto — acesso não autorizado ao Azure Files | Alta |
| R2 | Chave Google Maps sem restrição de domínio | Médio — uso indevido com custos | Média |
| R3 | JWT/token de auth armazenado em localStorage | Alto — vulnerável a XSS | Média |
| R4 | CORS do backend mal configurado | Alto — permite origens indesejadas | Baixa |
| R5 | `sourcemap: true` em produção | Médio — expõe código-fonte | Baixa |
| R6 | Variáveis de ambiente commitadas em `.env` | Alto — vazamento de segredos | Baixa |

### 10.2 Performance

| # | Ponto de Atenção | Impacto |
|---|-----------------|---------|
| P1 | Bundle size sem code-splitting por rota | Tempo de carregamento inicial alto |
| P2 | Imagens sem otimização (sem WebP, sem lazy loading) | Alto consumo de banda |
| P3 | Ausência de cache HTTP para assets estáticos | Requisições desnecessárias ao servidor |
| P4 | Google Maps carregando bibliotecas desnecessárias | Tempo de carregamento do mapa |

### 10.3 Manutenibilidade

| # | Ponto de Atenção |
|---|-----------------|
| M1 | Ausência de testes unitários/integração no frontend |
| M2 | Falta de documentação inline (JSDoc/TSDoc) em funções críticas |
| M3 | Dependências com versões `^` sem lock rigoroso (pode causar breaking changes) |
| M4 | `any` no TypeScript reduz benefícios da tipagem |

---

## 11. Recomendações Priorizadas

### 11.1 Curto Prazo (urgente — semanas)

| Prioridade | Ação | Justificativa |
|-----------|------|---------------|
| 🔴 P0 | **Mover geração de SAS token para o backend** — endpoint autenticado `GET /api/storage/sas` retorna token de curta duração | Elimina R1 (maior risco de segurança) |
| 🔴 P0 | **Restringir chave Google Maps** por domínio/referrer no Google Cloud Console | Elimina R2 |
| 🔴 P0 | **Garantir que `.env.*.local` e arquivos com secrets estão no `.gitignore`** | Previne R6 |
| 🟠 P1 | **Migrar armazenamento de token para `httpOnly cookie`** (requer ajuste no backend) | Reduz R3 |
| 🟠 P1 | **Desabilitar source maps em produção** (`sourcemap: false` no `vite.config.ts`) | Elimina R5 |

### 11.2 Médio Prazo (meses)

| Prioridade | Ação | Justificativa |
|-----------|------|---------------|
| 🟡 P2 | **Implementar code-splitting** com `React.lazy` + `Suspense` por rota | Melhora P1 — carregamento inicial |
| 🟡 P2 | **Adicionar testes unitários** (Vitest + React Testing Library) para componentes críticos e serviços | Melhora M1 |
| 🟡 P2 | **Configurar cache de assets** no `web.config` ou CDN | Melhora P3 |
| 🟡 P2 | **Otimizar imagens** com lazy loading (`loading="lazy"`) e formatos WebP | Melhora P2 |
| 🟡 P2 | **Ativar análise de bundle** (`vite-bundle-visualizer`) para identificar dependências pesadas | Referência para P1 |

### 11.3 Longo Prazo (roadmap)

| Prioridade | Ação | Justificativa |
|-----------|------|---------------|
| 🟢 P3 | **Adicionar CSP (Content Security Policy)** no `web.config` | Proteção adicional contra XSS |
| 🟢 P3 | **Implementar rate limiting e monitoramento** no backend | Proteção contra abuso da API |
| 🟢 P3 | **Configurar CDN** (Azure CDN / Front Door) na frente do App Service | Performance global |
| 🟢 P3 | **Migrar para React Query / TanStack Query** para gerenciamento de estado de servidor | Melhora caching, loading states e DX |
| 🟢 P3 | **Adicionar monitoramento de erros** (ex.: Sentry) no frontend | Visibilidade de erros em produção |

---

## 12. Referência de Endpoints do Backend

Com base no que o frontend consome, os seguintes endpoints são esperados na API .NET:

### 12.1 Autenticação

| Método | Rota | Descrição | Body / Params |
|--------|------|-----------|---------------|
| `POST` | `/api/auth/login` | Login de usuário | `{ email, password }` → `{ token, refreshToken, user }` |
| `POST` | `/api/auth/logout` | Logout / invalidação de token | Header: `Authorization: Bearer <token>` |
| `POST` | `/api/auth/refresh` | Renovar token | `{ refreshToken }` → `{ token }` |
| `GET`  | `/api/auth/me` | Dados do usuário autenticado | Header: `Authorization: Bearer <token>` |

### 12.2 Ativos (Assets)

| Método | Rota | Descrição | Body / Params |
|--------|------|-----------|---------------|
| `GET` | `/api/assets` | Listar ativos | Query: `?page=1&limit=20&search=...&filter=...` |
| `GET` | `/api/assets/:id` | Detalhe de um ativo | Path: `id` |
| `POST` | `/api/assets` | Criar ativo | Body: multipart/form-data ou JSON |
| `PUT` | `/api/assets/:id` | Atualizar ativo | Body: JSON com campos do ativo |
| `DELETE` | `/api/assets/:id` | Remover ativo | Path: `id` |
| `GET` | `/api/assets/:id/images` | Listar imagens do ativo | Path: `id` |
| `POST` | `/api/assets/:id/images` | Upload de imagem | Body: multipart/form-data |

### 12.3 Storage / Assets (Azure Files)

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/api/storage/sas` | Gerar token SAS de curta duração *(recomendado implementar)* |
| — | `/images/*` | Servido diretamente pelo Azure Files via URL pública + SAS |

### 12.4 Mapas / Geolocalização

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/api/assets/:id/location` | Coordenadas geográficas de um ativo |
| `PUT` | `/api/assets/:id/location` | Atualizar localização de um ativo |

### 12.5 Variáveis de Ambiente Esperadas no Backend (.NET)

```
# appsettings.json / variáveis de ambiente do servidor
ConnectionStrings__DefaultConnection=Server=...;Database=SmartAssets;...
AzureStorage__AccountName=<storage-account>
AzureStorage__AccountKey=<storage-key>       # MANTER COMO SECRET
AzureStorage__ShareName=<share-name>
Jwt__SecretKey=<chave-jwt>                   # MANTER COMO SECRET
Jwt__Issuer=https://smartassets.dominio.com
Jwt__Audience=https://smartassets.dominio.com
Jwt__ExpiryMinutes=60
AllowedOrigins=https://smartassets.dominio.com
```

> **⚠️ Segurança:** `AccountKey` e `Jwt__SecretKey` devem ser armazenados em **Azure Key Vault** ou como variáveis de ambiente do App Service (não em `appsettings.json` commitado).

### 12.6 Requisitos de Execução Local do Backend

```bash
# Pré-requisitos
- .NET 7+ SDK
- SQL Server 2019+ (ou Azure SQL)
- Conta Azure Storage (ou Azurite para emulação local)

# Configuração
cp appsettings.Development.json.example appsettings.Development.json
# Preencher strings de conexão e chaves

# Execução
dotnet restore
dotnet run --project src/SmartAssets.API
# API disponível em: https://localhost:5001 (ou http://localhost:5000)
```

---

## 13. Execução Local

### 13.1 Pré-requisitos

- **Node.js** 18 LTS ou 20 LTS
- **npm** 9+ (ou pnpm 8+)
- Acesso ao backend .NET em execução local (ou URL de desenvolvimento)
- Credenciais Azure Files (para visualização de imagens)
- Chave Google Maps

### 13.2 Configuração do Ambiente

```bash
# 1. Clonar o repositório
git clone <url-do-repositorio>
cd smartassets-frontend

# 2. Instalar dependências
npm ci

# 3. Criar arquivo de variáveis de ambiente
cp .env.example .env.development.local
# Editar .env.development.local com os valores do ambiente:
```

```bash
# .env.development.local
VITE_API_PROXY_TARGET=http://localhost:5000
VITE_DOTNET_API_BASE=http://localhost:5000
VITE_API_BASE_URL=/api
VITE_ASSETS_BASE_URL=https://<storage-account>.file.core.windows.net/<share>
VITE_ASSETS_SAS=?sv=2022-11-02&...
VITE_GOOGLE_MAPS_API_KEY=AIza...
VITE_GOOGLE_MAPS_LIBRARIES=places,geometry
```

```bash
# 4. Iniciar servidor de desenvolvimento
npm run dev
# Acesse: http://localhost:5173
```

### 13.3 Build de Produção

```bash
# Build otimizado
npm run build

# Pré-visualização local do build
npm run preview
# Acesse: http://localhost:4173
```

### 13.4 Linting

```bash
# Verificar problemas de código
npm run lint

# Corrigir automaticamente o que for possível
npm run lint -- --fix
```

---

## 14. Exportação para PDF

O documento foi criado em **Markdown** para máxima portabilidade. Use uma das opções abaixo para gerar o PDF.

### 14.1 Opção 1: Pandoc (recomendado — linha de comando)

```bash
# Instalar pandoc
# Ubuntu/Debian:
sudo apt install pandoc wkhtmltopdf

# macOS (Homebrew):
brew install pandoc wkhtmltopdf

# Gerar PDF
pandoc docs/diagnostico-tecnico-smartassets.md \
  -o docs/diagnostico-tecnico-smartassets.pdf \
  --pdf-engine=wkhtmltopdf \
  --toc \
  --toc-depth=2 \
  -V margin-top=25mm \
  -V margin-bottom=25mm \
  -V margin-left=20mm \
  -V margin-right=20mm \
  -V fontsize=11pt \
  -V lang=pt-BR

# Alternativa com engine weasyprint (melhor suporte a CSS):
pip install weasyprint
pandoc docs/diagnostico-tecnico-smartassets.md \
  -o docs/diagnostico-tecnico-smartassets.pdf \
  --pdf-engine=weasyprint
```

### 14.2 Opção 2: Navegador (sem instalação)

1. Abra o arquivo `docs/diagnostico-tecnico-smartassets.html` (ver abaixo) em qualquer navegador moderno.
2. Pressione `Ctrl+P` (ou `Cmd+P` no Mac).
3. Selecione **"Salvar como PDF"** como destino.
4. Em "Mais configurações": ative **"Gráficos de fundo"**, margens "Nenhuma" ou "Mínima".
5. Clique em **"Salvar"**.

### 14.3 Opção 3: VS Code Extension

1. Instale a extensão **"Markdown PDF"** (yzane.markdown-pdf) no VS Code.
2. Abra `docs/diagnostico-tecnico-smartassets.md`.
3. Clique com botão direito → **"Markdown PDF: Export (pdf)"**.

### 14.4 Opção 4: md-to-pdf (npm)

```bash
# Instalar globalmente
npm install -g md-to-pdf

# Gerar PDF
md-to-pdf docs/diagnostico-tecnico-smartassets.md
# Saída: docs/diagnostico-tecnico-smartassets.pdf
```

---

*Fim do Diagnóstico Técnico — SmartAssets / Smart Check — v1.0 — 2026-03-06*
