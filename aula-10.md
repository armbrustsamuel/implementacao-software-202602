# Aula 10 — Setup do projeto e primeira integração com o backend

## Objetivo

Criar o projeto Vue 3 + Vite, configurar o cliente HTTP Axios, construir o cabeçalho da aplicação e exibir a listagem de categorias carregada do backend.

Ao final, acessar `http://localhost:5173` e ver a lista de categorias vinda do backend.

---

### Estrutura do projeto

```
ecommerce-frontend/
├── index.html
├── vite.config.js
├── package.json
├── .env
└── src/
    ├── main.js
    ├── App.vue
    ├── assets/
    │   └── base.css
    ├── components/
    │   └── AppHeader.vue
    │   └── CategoriaLista.vue
    └── services/
        └── api.js
        └── categoriaService.js
```

### Criação via Vite

```bash
npm create vite@latest ecommerce-frontend --template vue
cd ecommerce-frontend
npm install
npm install axios
```

### `main.js`

```js
import { createApp } from 'vue'
import './assets/base.css'
import App from './App.vue'

createApp(App).mount('#app')
```

### `src/services/api.js` — cliente Axios

```js
import axios from "axios"

export const api = axios.create({
    baseURL: import.meta.env?.VITE_API_URL || 'http://localhost:8080',
})
```

O arquivo `.env` pode definir `VITE_API_URL` para apontar para outro servidor.

### `src/services/categoriaService.js`

```js
import { api } from './api.js'

export async function listarCategorias() {
    const response = await api.get('/categorias')
    return response.data
}
```

### `src/components/AppHeader.vue`

```vue
<template>
    <header class="app-header">
        <div class="container header-content">
            <strong>E-commerce</strong>
            <span>Frontend em Vue.js</span>
        </div>
    </header>
</template>
```

### `src/components/CategoriaLista.vue` — integração com o backend

```vue
<script setup>
import { ref, onMounted } from 'vue'
import { listarCategorias } from '../services/categoriaService.js'

const categorias = ref([])
const carregando = ref(false)
const erro = ref('')

async function carregar() {
    carregando.value = true
    erro.value = ''
    try {
        categorias.value = await listarCategorias()
    } catch (e) {
        erro.value = 'Não foi possível carregar as categorias.'
    } finally {
        carregando.value = false
    }
}

onMounted(() => { carregar() })
</script>

<template>
    <section class="container">
        <h2>Categorias</h2>
        <p v-if="carregando">Carregando...</p>
        <p v-else-if="erro" class="erro">{{ erro }}</p>
        <ul v-else-if="categorias.length">
            <li v-for="c in categorias" :key="c.id">{{ c.nome }}</li>
        </ul>
        <p v-else>Nenhuma categoria cadastrada.</p>
    </section>
</template>
```

### `src/App.vue`

```vue
<script setup>
import AppHeader from './components/AppHeader.vue'
import CategoriaLista from './components/CategoriaLista.vue'
</script>

<template>
    <AppHeader />
    <main class="container">
        <CategoriaLista />
    </main>
</template>
```

### `src/assets/base.css` — estilos globais mínimos

Define reset (`* { box-sizing: border-box }`), tipografia, `.container`, `.app-header` e `.erro`.

### Ajuste erro CORS

Para ajustar o erro de CORS, é necessário configurar o backend para permitir requisições do frontend.

---

## O que os alunos precisam fazer

1. Criar o projeto com `npm create vite@latest --template vue`
2. Instalar dependências: `npm install` e `npm install axios`
3. Limpar arquivos de exemplo do Vite (`HelloWorld.vue`, `TheWelcome.vue`, etc.)
4. Criar `src/services/api.js` com a instância Axios
5. Criar `src/services/categoriaService.js` com `listarCategorias`
6. Criar `AppHeader.vue` e `CategoriaLista.vue`
7. Atualizar `App.vue` para renderizar ambos
8. Rodar `npm run dev`, iniciar o backend e confirmar que as categorias aparecem

## Conceitos abordados

- **Vite**: bundler moderno com hot reload muito mais rápido que webpack
- **Single File Component (SFC)**: arquivos `.vue` com `<script setup>`, `<template>` e `<style>`
- **`<script setup>`**: sintaxe da Composition API — sem `export default`, mais conciso
- **Variáveis de ambiente Vite**: prefixo `VITE_` expõe variáveis no código via `import.meta.env`
- **Axios**: cliente HTTP com API mais amigável que `fetch` nativo
- **`ref()`**: torna um valor primitivo reativo
- **`onMounted()`**: hook executado após o componente ser inserido no DOM
- **`async/await` com `try/catch/finally`**: fluxo assíncrono com tratamento de erro e limpeza de estado
- **`v-if` / `v-else-if` / `v-else`**: renderização condicional para estados de loading/erro/vazio/dados
