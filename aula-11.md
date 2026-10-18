# Aula 11 — Composable de listagem e exibição de produtos em cards

## Objetivo

Extrair a lógica de carregamento assíncrono num composable reutilizável (`useListagem`), criar um componente genérico de painel com slots e exibir os produtos do backend em cards numa grade responsiva.

Ao final, ver os produtos carregados do backend exibidos em cards com nome, descrição, preço formatado e quantidade em estoque.

---

### Composable `useListagem`

**Arquivo:** `src/composables/useListagem.js`

Encapsula o estado reativo e a lógica de carregamento para qualquer listagem.

```js
import { ref, computed } from 'vue'

export function useListagem(carregadorPadrao) {
    const itens = ref([])
    const carregando = ref(false)
    const erro = ref('')
    const vazio = computed(() => !carregando.value && !erro.value && itens.value.length === 0)
    const total = computed(() => itens.value.length)

    async function carregar(carregador = carregadorPadrao) {
        carregando.value = true
        erro.value = ''
        try {
            const dados = await carregador()
            itens.value = Array.isArray(dados) ? dados : []
        } catch (e) {
            erro.value = 'Não foi possível carregar os dados. Verifique se o backend está no ar.'
            itens.value = []
        } finally {
            carregando.value = false
        }
    }

    return { itens, carregando, erro, vazio, total, carregar }
}
```

> A lógica que estava espalhada em `CategoriaLista` e `ProdutoLista` agora vive num único lugar. Qualquer componente de listagem usa o mesmo composable.

### `src/services/produtoService.js`

```js
import { api } from './api.js'

export async function listarProdutos() {
    const response = await api.get('/produtos/todos')
    return response.data
}
```

### Componente genérico `PainelLista.vue`

Recebe props de controle e usa **slots** para injetar conteúdo:

```vue
<script setup>
defineProps({
    titulo: String,
    total: { type: Number, default: 0 },
    carregando: Boolean,
    erro: String,
    vazio: Boolean,
    mensagemVazio: { type: String, default: 'Nenhum item encontrado.' },
})
defineEmits(['atualizar'])
</script>

<template>
    <section class="container">
        <div class="painel-header">
            <h2>{{ titulo }} <span v-if="total">({{ total }})</span></h2>
            <div class="painel-acoes">
                <slot name="acoes" />
                <button class="btn" @click="$emit('atualizar')">Atualizar</button>
            </div>
        </div>
        <slot name="filtros" />
        <p v-if="carregando">Carregando...</p>
        <p v-else-if="erro" class="erro">{{ erro }}</p>
        <p v-else-if="vazio">{{ mensagemVazio }}</p>
        <slot v-else />
    </section>
</template>
```

- **slot padrão**: conteúdo da lista (cards, tabela, etc.)
- **slot `#filtros`**: área de filtros acima do conteúdo (usado na aula seguinte)
- **slot `#acoes`**: botões extras na barra do título

### `ProdutoCard.vue`

Componente de apresentação: recebe um `produto` via props e formata os dados.

```vue
<script setup>
import { computed } from 'vue'

const props = defineProps({ produto: { type: Object, required: true } })

const precoFormatado = computed(() => {
    const valor = Number(props.produto.preco)
    if (Number.isNaN(valor)) return 'R$ 0,00'
    return valor.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' })
})
</script>

<template>
    <article class="produto-card">
        <div>
            <h2>{{ produto.nome }}</h2>
            <p>{{ produto.descricao || 'Produto sem descrição cadastrada.' }}</p>
        </div>
        <footer>
            <strong>{{ precoFormatado }}</strong>
            <span>Estoque: {{ produto.estoque }}</span>
        </footer>
    </article>
</template>
```

### `ProdutoLista.vue`

```vue
<script setup>
import { onMounted } from 'vue'
import { listarProdutos } from '../services/produtoService.js'
import { useListagem } from '../composables/useListagem.js'
import PainelLista from './PainelLista.vue'
import ProdutoCard from './ProdutoCard.vue'

const { itens: produtos, carregando, erro, vazio, total, carregar: carregarProdutos } = useListagem(listarProdutos)

defineExpose({ carregarProdutos })
onMounted(() => { carregarProdutos() })
</script>

<template>
    <PainelLista titulo="Catálogo de produtos" :total="total"
        :carregando="carregando" :erro="erro" :vazio="vazio"
        mensagem-vazio="Nenhum produto cadastrado."
        @atualizar="carregarProdutos">
        <section class="produto-grid">
            <ProdutoCard v-for="produto in produtos" :key="produto.id" :produto="produto" />
        </section>
    </PainelLista>
</template>
```

### CSS adicionado em `base.css`

```css
.produto-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
    gap: 16px;
}

.produto-card {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    border: 1px solid #d8dee8;
    border-radius: 8px;
    background: #ffffff;
    padding: 20px;
}
```

### `App.vue` atualizado

```vue
<script setup>
import AppHeader from './components/AppHeader.vue'
import ProdutoLista from './components/ProdutoLista.vue'
</script>

<template>
    <AppHeader />
    <main>
        <ProdutoLista />
    </main>
</template>
```

---

## O que os alunos precisam fazer

1. Criar `src/composables/useListagem.js`
2. Criar `src/services/produtoService.js` com `listarProdutos`
3. Criar `src/components/PainelLista.vue` com props, emits e slots
4. Criar `src/components/ProdutoCard.vue` com preço formatado
5. Criar `src/components/ProdutoLista.vue` usando `useListagem` + `PainelLista`
6. Refatorar `CategoriaLista.vue` para usar `useListagem` (opcional, mas reforça o padrão)
7. Atualizar `App.vue` para renderizar `<ProdutoLista />`
8. Testar: com o backend rodando, confirmar que os cards de produto aparecem na tela

## Conceitos abordados

- **Composables**: funções que encapsulam estado reativo e lógica reutilizável — padrão central do Vue 3
- **`computed()`**: valor derivado que recalcula automaticamente ao mudar dependências
- **Slots**: permitem ao componente pai injetar conteúdo em posições específicas do filho
- **Slots nomeados (`#filtros`, `#acoes`)**: múltiplos pontos de injeção no mesmo componente
- **Componente de apresentação**: recebe dados via `props`, sem estado próprio
- **`defineExpose()`**: torna métodos do filho acessíveis ao pai via `ref` do template
- **`v-for` com `:key`**: renderiza lista de componentes de forma eficiente
- **`toLocaleString('pt-BR')`**: formata número como moeda brasileira
- **CSS Grid com `auto-fit` + `minmax`**: layout responsivo sem media queries
