# Aula 12 — Formulário de cadastro de produto com modal e validação

## Objetivo

Construir o formulário de cadastro de produto com validação reativa no frontend e integração com o endpoint `POST /produtos` do backend.

O formulário será exibido em um **popup/modal**. Após o cadastro, o modal será fechado e o novo produto deverá aparecer automaticamente na lista sem recarregar a página.

---

# 1. `src/services/produtoService.js`

Substitua o conteúdo do arquivo por:

```js
import { api } from './api.js'

export async function listarProdutos() {
    const response = await api.get('/produtos/todos')
    return response.data
}

export async function cadastrarProduto(produto) {
    const response = await api.post('/produtos', produto)
    return response.data
}
```

---

# 2. Criar o componente `src/components/ProdutoFormulario.vue`


```vue
<script setup>
import { ref, computed, onMounted } from 'vue'
import { cadastrarProduto } from '../services/produtoService.js'
import { listarCategorias } from '../services/categoriaService.js'

const emit = defineEmits([
    'produto-cadastrado',
    'fechar'
])

const produto = ref({
    nome: '',
    descricao: '',
    preco: '',
    estoque: '',
    categoriaId: null
})

const tentouEnviar = ref(false)
const salvando = ref(false)
const erroFormulario = ref('')
const categorias = ref([])

const erros = computed(() => {
    const e = {}

    if (!produto.value.nome.trim()) {
        e.nome = 'Informe o nome do produto.'
    } else if (produto.value.nome.trim().length < 3) {
        e.nome = 'O nome deve ter pelo menos 3 caracteres.'
    }

    if (!produto.value.preco) {
        e.preco = 'Informe o preço.'
    } else if (Number(produto.value.preco) <= 0) {
        e.preco = 'O preço deve ser maior que zero.'
    }

    if (produto.value.estoque === '') {
        e.estoque = 'Informe o estoque.'
    } else if (Number(produto.value.estoque) < 0) {
        e.estoque = 'O estoque não pode ser negativo.'
    }

    if (!produto.value.categoriaId) {
        e.categoriaId = 'Selecione uma categoria.'
    }

    return e
})

const formularioValido = computed(() => {
    return Object.keys(erros.value).length === 0
})

async function salvarProduto() {
    tentouEnviar.value = true

    if (!formularioValido.value) {
        return
    }

    salvando.value = true
    erroFormulario.value = ''

    try {
        const {
            nome,
            descricao,
            preco,
            estoque,
            categoriaId
        } = produto.value

        await cadastrarProduto({
            nome: nome.trim(),
            descricao: descricao.trim(),
            preco: Number(preco),
            estoque: Number(estoque),
            categoriaId: Number(categoriaId)
        })

        produto.value = {
            nome: '',
            descricao: '',
            preco: '',
            estoque: '',
            categoriaId: null
        }

        tentouEnviar.value = false

        emit('produto-cadastrado')

    } catch (error) {
        console.error(error)

        erroFormulario.value =
            'Não foi possível cadastrar o produto. Tente novamente.'

    } finally {
        salvando.value = false
    }
}

onMounted(async () => {
    categorias.value = await listarCategorias()
})
</script>

<template>
    <div
        class="modal-overlay"
        @click.self="emit('fechar')"
    >
        <div class="modal">

            <div class="modal-header">

                <div>
                    <h2>Cadastrar Produto</h2>
                    <p>Informe os dados do novo produto.</p>
                </div>

                <button
                    type="button"
                    class="modal-fechar"
                    @click="emit('fechar')"
                >
                    ×
                </button>

            </div>

            <form @submit.prevent="salvarProduto">

                <p
                    v-if="erroFormulario"
                    class="erro"
                >
                    {{ erroFormulario }}
                </p>

                <div class="campo">

                    <label for="nome">
                        Nome
                    </label>

                    <input
                        id="nome"
                        v-model="produto.nome"
                        type="text"
                        placeholder="Digite o nome do produto"
                        :class="{
                            'input-error':
                                tentouEnviar && erros.nome
                        }"
                    />

                    <span
                        v-if="tentouEnviar && erros.nome"
                        class="field-error"
                    >
                        {{ erros.nome }}
                    </span>

                </div>

                <div class="campo">

                    <label for="descricao">
                        Descrição
                    </label>

                    <textarea
                        id="descricao"
                        v-model="produto.descricao"
                        maxlength="500"
                        placeholder="Descreva o produto..."
                    ></textarea>

                    <span class="field-help">
                        {{ produto.descricao.length }}/500 caracteres
                    </span>

                </div>

                <div class="form-row">

                    <div class="campo">

                        <label for="preco">
                            Preço (R$)
                        </label>

                        <input
                            id="preco"
                            type="number"
                            step="0.01"
                            v-model="produto.preco"
                            placeholder="0,00"
                            :class="{
                                'input-error':
                                    tentouEnviar && erros.preco
                            }"
                        />

                        <span
                            v-if="tentouEnviar && erros.preco"
                            class="field-error"
                        >
                            {{ erros.preco }}
                        </span>

                    </div>

                    <div class="campo">

                        <label for="estoque">
                            Estoque
                        </label>

                        <input
                            id="estoque"
                            type="number"
                            v-model="produto.estoque"
                            placeholder="0"
                            :class="{
                                'input-error':
                                    tentouEnviar && erros.estoque
                            }"
                        />

                        <span
                            v-if="tentouEnviar && erros.estoque"
                            class="field-error"
                        >
                            {{ erros.estoque }}
                        </span>

                    </div>

                </div>

                <div class="campo">

                    <label for="categoria">
                        Categoria
                    </label>

                    <select
                        id="categoria"
                        v-model="produto.categoriaId"
                        :class="{
                            'input-error':
                                tentouEnviar && erros.categoriaId
                        }"
                    >
                        <option :value="null">
                            Selecione uma categoria...
                        </option>

                        <option
                            v-for="c in categorias"
                            :key="c.id"
                            :value="c.id"
                        >
                            {{ c.nome }}
                        </option>
                    </select>

                    <span
                        v-if="tentouEnviar && erros.categoriaId"
                        class="field-error"
                    >
                        {{ erros.categoriaId }}
                    </span>

                </div>

                <div class="modal-footer">

                    <button
                        type="button"
                        class="btn btn-secondary"
                        @click="emit('fechar')"
                    >
                        Cancelar
                    </button>

                    <button
                        type="submit"
                        class="btn btn-primary"
                        :disabled="salvando"
                    >
                        {{ salvando ? 'Salvando...' : 'Cadastrar' }}
                    </button>

                </div>

            </form>

        </div>
    </div>
</template>
```

---

# 3. Ajuste o código do arquivo`src/App.vue`

```vue
<script setup>
import { ref } from 'vue'

import AppHeader from './components/AppHeader.vue'
import ProdutoFormulario from './components/ProdutoFormulario.vue'
import ProdutoLista from './components/ProdutoLista.vue'

const produtoLista = ref(null)
const mostrarFormulario = ref(false)

function atualizarProdutos() {
    produtoLista.value?.carregarProdutos()
}

function produtoCadastrado() {
    mostrarFormulario.value = false
    atualizarProdutos()
}
</script>

<template>
    <AppHeader />

    <main class="pagina">

        <div class="pagina-cabecalho">

            <div>
                <h1>Produtos</h1>
                <p>Gerencie os produtos cadastrados.</p>
            </div>

            <button
                class="btn btn-primary"
                @click="mostrarFormulario = true"
            >
                + Novo Produto
            </button>

        </div>

        <ProdutoLista
            ref="produtoLista"
        />

        <ProdutoFormulario
            v-if="mostrarFormulario"
            @produto-cadastrado="produtoCadastrado"
            @fechar="mostrarFormulario = false"
        />

    </main>
</template>
```

---

# 4. `src/assets/base.css`

Utilise o `base.css` atualizado disponível no Moodle.

---

# 5. O que os alunos precisam fazer

1. Adicionar `cadastrarProduto` em `produtoService.js`.
2. Criar `src/components/ProdutoFormulario.vue`.
3. Substituir o conteúdo do `App.vue`.
4. Adicionar as novas classes CSS ao `base.css`.
5. Executar a aplicação.
6. Clicar em **+ Novo Produto**.
7. Testar o formulário vazio.
8. Testar os campos individualmente.
9. Cadastrar um produto válido.
10. Confirmar que o produto aparece na lista sem recarregar a página.

---

## Testes esperados

- Clicar em **+ Novo Produto** → formulário aparece no popup.
- Clicar no `×` → popup fecha.
- Clicar em **Cancelar** → popup fecha.
- Clicar fora do formulário → popup fecha.
- Tentar enviar vazio → erros aparecem em vermelho.
- Preencher apenas alguns campos → erros dos campos faltantes permanecem.
- Nome com menos de 3 caracteres → erro.
- Preço menor ou igual a zero → erro.
- Estoque negativo → erro.
- Preencher tudo corretamente → produto é enviado.
- Durante o envio → botão mostra `Salvando...`.
- Após o sucesso → modal fecha.
- Após o sucesso → novo produto aparece na lista sem recarregar a página.

---

# Outros detalhes

## Por que o formulário foi colocado em um modal?

Antes, o `ProdutoFormulario` fazia parte diretamente da página:

```vue
<ProdutoFormulario />
<ProdutoLista />
```

Agora o formulário só é renderizado quando:

```js
mostrarFormulario
```

é `true`:

```vue
<ProdutoFormulario
    v-if="mostrarFormulario"
/>
```

O botão:

```vue
<button @click="mostrarFormulario = true">
    + Novo Produto
</button>
```

abre o formulário.

O formulário emite:

```js
emit('fechar')
```

quando o usuário deseja fechá-lo.

O `App.vue` recebe o evento:

```vue
@fechar="mostrarFormulario = false"
```

---

## Por que o formulário anterior estava visualmente tão simples?

Porque os componentes do Vue **não possuem automaticamente um estilo visual específico**.

Por exemplo:

```vue
<input>
```

continua sendo um elemento HTML padrão.

O Vue fornece principalmente:

- reatividade;
- componentes;
- `v-model`;
- `computed`;
- `v-if`;
- `v-for`;
- eventos;
- binding.

A aparência é definida pelo CSS.

Por isso adicionamos classes como:

```text
.modal
.campo
.btn
.btn-primary
.input-error
.field-error
.modal-header
.modal-footer
```

Essas classes não são classes especiais do Vue. São simplesmente classes CSS criadas pela aplicação.

---

## Por que o `produto` precisa estar no `<script setup>`?

O template utiliza:

```vue
v-model="produto.nome"
```

e:

```vue
{{ produto.descricao.length }}
```

Portanto, `produto` precisa existir no contexto do componente:

```js
const produto = ref({
    nome: '',
    descricao: '',
    preco: '',
    estoque: '',
    categoriaId: null
})
```

O mesmo vale para:

```text
erroFormulario
tentouEnviar
erros
salvando
categorias
```

Todos são utilizados pelo template e precisam estar declarados no `<script setup>`.

Caso contrário, o Vue apresenta avisos como:

```text
Property "produto" was accessed during render but is not defined
```

ou:

```text
Property "tentouEnviar" was accessed during render but is not defined
```

---

## Por que usamos `computed` para os erros?

A validação depende dos valores de:

```js
produto
```

Por exemplo:

```js
produto.value.nome
```

Quando o usuário altera o nome, o `computed` é automaticamente recalculado.

Por isso:

```js
const erros = computed(() => {
    ...
})
```

é adequado para representar um **valor derivado do estado do formulário**.

---

## Por que os erros não aparecem imediatamente?

A variável:

```js
const tentouEnviar = ref(false)
```

começa como `false`.

A mensagem:

```vue
<span
    v-if="tentouEnviar && erros.nome"
>
```

só aparece quando:

```text
tentouEnviar = true
```

e existe um erro para aquele campo.

Ao clicar em cadastrar:

```js
tentouEnviar.value = true
```

Assim, o usuário pode preencher o formulário sem receber mensagens de erro enquanto ainda está digitando.

---

## Por que usamos `@submit.prevent`?

O HTML tradicional faria o navegador recarregar a página ao enviar um formulário.

Utilizamos:

```vue
<form @submit.prevent="salvarProduto">
```

O:

```text
@submit
```

captura o evento de envio.

O:

```text
.prevent
```

impede o comportamento padrão do navegador.

Assim, quem controla o envio é o Vue:

```js
salvarProduto()
```

que posteriormente chama:

```js
cadastrarProduto(...)
```

---

## Por que usamos `Number()`?

Os valores dos campos HTML normalmente chegam como texto.

Por exemplo:

```js
produto.value.preco
```

pode conter:

```text
"4500"
```

mas queremos enviar ao backend:

```js
4500
```

Por isso:

```js
preco: Number(preco)
```

O mesmo acontece com:

```js
estoque: Number(estoque)
```

e:

```js
categoriaId: Number(categoriaId)
```

---

## Por que usamos `defineEmits`?

O formulário é um componente filho de `App.vue`.

Ele não deve decidir diretamente como o `App.vue` controla o modal.

Em vez disso, ele informa ao pai que algo aconteceu:

```js
emit('fechar')
```

ou:

```js
emit('produto-cadastrado')
```

O pai decide o que fazer.

Por exemplo:

```vue
@produto-cadastrado="produtoCadastrado"
```

executa:

```js
function produtoCadastrado() {
    mostrarFormulario.value = false
    atualizarProdutos()
}
```

---

## Por que usamos `ref="produtoLista"`?

O `ProdutoLista` possui o método:

```js
carregarProdutos()
```

Depois de cadastrar um produto, precisamos executar esse método novamente.

No template:

```vue
<ProdutoLista
    ref="produtoLista"
/>
```

e no script:

```js
const produtoLista = ref(null)
```

permitem acessar o componente:

```js
produtoLista.value?.carregarProdutos()
```

Assim, a lista é atualizada sem recarregar a página.

---

## Por que o modal fecha depois do cadastro?

Após o backend confirmar o cadastro:

```js
emit('produto-cadastrado')
```

O `App.vue` recebe o evento e executa:

```js
function produtoCadastrado() {
    mostrarFormulario.value = false
    atualizarProdutos()
}
```

Portanto, duas coisas acontecem:

```text
1. fechar o modal
2. atualizar a lista
```

---

## Por que `@click.self`?

O overlay possui:

```vue
@click.self="emit('fechar')"
```

O `.self` faz com que o evento seja executado somente quando o usuário clicar no próprio overlay.

Assim:

```text
clicar fora do modal → fecha

clicar dentro do modal → não fecha
```

---

## Conceitos abordados

- **`ref`**: estado reativo do formulário e controle da abertura do modal.
- **`v-model`**: two-way binding entre inputs e estado.
- **`computed`**: cálculo automático dos erros e da validade do formulário.
- **Binding de classe**: aplicação condicional da classe `input-error`.
- **`v-if`**: renderização condicional do modal e mensagens de erro.
- **`v-for`**: criação das opções de categoria.
- **`defineEmits`**: comunicação do componente filho com o pai.
- **`@submit.prevent`**: interceptação do envio do formulário.
- **`:disabled`**: bloqueio do botão durante o envio.
- **`onMounted`**: carregamento das categorias.
- **`ref` em componente**: acesso ao método `carregarProdutos()` do `ProdutoLista`.
- **CSS**: criação de uma interface mais próxima de uma aplicação frontend moderna.
- **Modal**: exibição do formulário sobre a página.
- **Integração com API**: envio do produto através de `POST /produtos`.
