# Aula 06 — Consultas avançadas, listagens e geração de relatórios

## Objetivo

Adicionar busca paginada com filtros, uma projeção JPQL para relatório e atualização parcial de estoque via PATCH.

## Novos endpoints

| Método | URL | Descrição |
|---|---|---|
| `GET` | `/produtos?nome=X&categoriaId=1&page=0&size=10&sort=nome,asc` | Busca paginada e filtrada |
| `GET` | `/produtos/relatorio/por-categoria` | Relatório de quantidade por categoria |
| `PATCH` | `/produtos/{id}/estoque` | Atualização parcial do estoque |

---

## Passo 1 — Criar `ProdutoPorCategoriaProjection`

**Pacote:** `br.unisinos.ecommerce.repository`  
**Arquivo:** `ProdutoPorCategoriaProjection.java`

```java
package br.unisinos.ecommerce.repository;

public interface ProdutoPorCategoriaProjection {
    String getCategoria();
    Long getQuantidade();
}
```

> Interfaces de projeção mapeiam o resultado de uma `@Query` JPQL para getters tipados, sem precisar de uma classe concreta.

---

## Passo 2 — Criar `AtualizarEstoqueDTO`

**Pacote:** `br.unisinos.ecommerce.dto`  
**Arquivo:** `AtualizarEstoqueDTO.java`

```java
package br.unisinos.ecommerce.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;

public record AtualizarEstoqueDTO(
    @NotNull
    @Min(value = 0, message = "O estoque não pode ser negativo")
    Integer estoque
) {}
```

---

## Passo 3 — Atualizar `ProdutoRepository`

**Pacote:** `br.unisinos.ecommerce.repository`  
**Arquivo:** `ProdutoRepository.java` — substituir o conteúdo completo

```java
package br.unisinos.ecommerce.repository;

import java.util.List;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;

import br.unisinos.ecommerce.entity.Produto;

@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> {

    @Query("""
        select p from Produto p
        where (:nome is null or lower(p.nome) like lower(concat('%', :nome, '%')))
          and (:categoriaId is null or p.categoria.id = :categoriaId)
    """)
    Page<Produto> buscarComFiltros(String nome, Long categoriaId, Pageable pageable);

    @Query("""
        select c.nome as categoria, count(p) as quantidade
        from Produto p join p.categoria c
        group by c.nome
    """)
    List<ProdutoPorCategoriaProjection> relatorioProdutosPorCategoria();
}
```

> Query 1: Busca produtos no banco de dados usando filtros opcionais (por nome ou categoria) e entrega o resultado dividido em páginas.

> Query 2: Cria um relatório que agrupa os produtos por categoria e mostra a quantidade total de itens em cada uma delas.` 

---

## Passo 4 — Atualizar `ProdutoService`

**Pacote:** `br.unisinos.ecommerce.service`  
**Arquivo:** `ProdutoService.java` — substituir o conteúdo completo

```java
package br.unisinos.ecommerce.service;

import java.util.List;
import java.util.Optional;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;

import br.unisinos.ecommerce.dto.AtualizarEstoqueDTO;
import br.unisinos.ecommerce.dto.ProdutoRequestDTO;
import br.unisinos.ecommerce.dto.ProdutoResponseDTO;
import br.unisinos.ecommerce.entity.Categoria;
import br.unisinos.ecommerce.entity.Produto;
import br.unisinos.ecommerce.exception.RecursoNaoEncontradoException;
import br.unisinos.ecommerce.exception.RegraNegocioException;
import br.unisinos.ecommerce.repository.CategoriaRepository;
import br.unisinos.ecommerce.repository.ProdutoPorCategoriaProjection;
import br.unisinos.ecommerce.repository.ProdutoRepository;

@Service
public class ProdutoService {

    private final CategoriaRepository categoriaRepository;
    private final ProdutoRepository produtoRepository;

    public ProdutoService(CategoriaRepository categoriaRepository, ProdutoRepository produtoRepository) {
        super();
        this.categoriaRepository = categoriaRepository;
        this.produtoRepository = produtoRepository;
    }

    private ProdutoResponseDTO toResponseDTO(Produto p) {
        return new ProdutoResponseDTO(
            p.getId(), p.getNome(), p.getDescricao(), p.getPreco(), p.getEstoque(),
            p.getCategoria() != null ? p.getCategoria().getId() : null,
            p.getCategoria() != null ? p.getCategoria().getNome() : null
        );
    }

    public ProdutoResponseDTO salvar(ProdutoRequestDTO dto) {
        Categoria categoria = categoriaRepository.findById(dto.categoriaId())
            .orElseThrow(() -> new RegraNegocioException("Categoria não encontrada"));
        Produto produto = new Produto(dto.nome(), dto.descricao(), dto.preco(), dto.estoque(), categoria);
        return toResponseDTO(produtoRepository.save(produto));
    }

    public List<ProdutoResponseDTO> listarTodas() {
        return produtoRepository.findAll().stream()
            .map(this::toResponseDTO)
            .toList();
    }

    public Optional<ProdutoResponseDTO> buscarPorId(Long id) {
        return produtoRepository.findById(id).map(this::toResponseDTO);
    }

    public ProdutoResponseDTO atualizar(Long id, ProdutoRequestDTO dto) {
        Produto produto = produtoRepository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Produto não encontrado"));
        Categoria categoria = categoriaRepository.findById(dto.categoriaId())
            .orElseThrow(() -> new RecursoNaoEncontradoException("Categoria não encontrada"));
        produto.setNome(dto.nome());
        produto.setDescricao(dto.descricao());
        produto.setPreco(dto.preco());
        produto.setEstoque(dto.estoque());
        produto.setCategoria(categoria);
        return toResponseDTO(produtoRepository.save(produto));
    }

    public void excluir(Long id) {
        produtoRepository.deleteById(id);
    }

    // --- NOVOS MÉTODOS ---

    public Page<ProdutoResponseDTO> buscar(String nome, Long categoriaId, int page, int size, String ordenarPor) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(ordenarPor));
        return produtoRepository.buscarComFiltros(nome, categoriaId, pageable)
            .map(this::toResponseDTO);
    }

    public List<ProdutoPorCategoriaProjection> relatorioProdutosPorCategoria() {
        return produtoRepository.relatorioProdutosPorCategoria();
    }

    public ProdutoResponseDTO atualizarEstoque(Long id, AtualizarEstoqueDTO dto) {
        Produto produto = produtoRepository.findById(id)
            .orElseThrow(() -> new RecursoNaoEncontradoException("Produto não encontrado"));
        produto.setEstoque(dto.estoque());
        return toResponseDTO(produtoRepository.save(produto));
    }
}
```

---

## Passo 5 — Atualizar `ProdutoController`

**Pacote:** `br.unisinos.ecommerce.controller`  
**Arquivo:** `ProdutoController.java` — substituir o conteúdo completo

```java
package br.unisinos.ecommerce.controller;

import java.util.List;
import java.util.Optional;

import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PatchMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import br.unisinos.ecommerce.dto.AtualizarEstoqueDTO;
import br.unisinos.ecommerce.dto.ProdutoRequestDTO;
import br.unisinos.ecommerce.dto.ProdutoResponseDTO;
import br.unisinos.ecommerce.repository.ProdutoPorCategoriaProjection;
import br.unisinos.ecommerce.service.ProdutoService;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/produtos")
public class ProdutoController {

    private final ProdutoService produtoService;

    public ProdutoController(ProdutoService produtoService) {
        super();
        this.produtoService = produtoService;
    }

    @PostMapping
    public ResponseEntity<ProdutoResponseDTO> salvar(@Valid @RequestBody ProdutoRequestDTO dto) {
        ProdutoResponseDTO produto = this.produtoService.salvar(dto);
        return new ResponseEntity<ProdutoResponseDTO>(produto, HttpStatus.ACCEPTED);
    }

    @GetMapping("/todos")
    public ResponseEntity<List<ProdutoResponseDTO>> listarTodos() {
        List<ProdutoResponseDTO> listaProdutos = this.produtoService.listarTodas();
        return new ResponseEntity<List<ProdutoResponseDTO>>(listaProdutos, HttpStatus.OK);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProdutoResponseDTO> buscarPorId(@PathVariable Long id) {
        Optional<ProdutoResponseDTO> produto = produtoService.buscarPorId(id);
        if (produto.isPresent()) {
            return ResponseEntity.ok(produto.get());
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProdutoResponseDTO> atualizar(@PathVariable Long id,
            @Valid @RequestBody ProdutoRequestDTO dto) {
        ProdutoResponseDTO produtoAtualizado = produtoService.atualizar(id, dto);
        return ResponseEntity.ok(produtoAtualizado);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> excluir(@PathVariable Long id) {
        Optional<ProdutoResponseDTO> produto = produtoService.buscarPorId(id);
        if (produto.isEmpty()) {
            return ResponseEntity.notFound().build();
        }
        produtoService.excluir(id);
        return ResponseEntity.noContent().build();
    }

    // --- NOVOS ENDPOINTS ---

    @GetMapping
    public ResponseEntity<Page<ProdutoResponseDTO>> buscar(
            @RequestParam(required = false) String nome,
            @RequestParam(required = false) Long categoriaId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "nome") String ordenarPor) {
        Page<ProdutoResponseDTO> pagina = produtoService.buscar(nome, categoriaId, page, size, ordenarPor);
        return ResponseEntity.ok(pagina);
    }

    @GetMapping("/relatorio/por-categoria")
    public ResponseEntity<List<ProdutoPorCategoriaProjection>> relatorioPorCategoria() {
        List<ProdutoPorCategoriaProjection> relatorio = produtoService.relatorioProdutosPorCategoria();
        return ResponseEntity.ok(relatorio);
    }

    @PatchMapping("/{id}/estoque")
    public ResponseEntity<ProdutoResponseDTO> atualizarEstoque(@PathVariable Long id,
            @Valid @RequestBody AtualizarEstoqueDTO dto) {
        ProdutoResponseDTO produto = produtoService.atualizarEstoque(id, dto);
        return ResponseEntity.ok(produto);
    }
}
```

> **Atenção:** o endpoint `GET /produtos` (busca paginada) agora substitui o antigo `GET /produtos` (listar todos). O listar todos foi movido para `GET /produtos/todos`.

---

## Exemplo de uso da busca paginada

```
GET /produtos?nome=mouse&categoriaId=1&page=0&size=5&ordenarPor=nome
```

A resposta é um objeto `Page<ProdutoResponseDTO>` com:
```json
{
  "content": [...],
  "totalElements": 12,
  "totalPages": 3,
  "first": true,
  "last": false,
  "size": 5,
  "number": 0
}
```

Sem parâmetros, retorna a primeira página com 10 itens ordenados por nome:
```
GET /produtos
```

---

## Testes via terminal

### Windows (cmd / PowerShell)

Busca paginada sem filtros (primeira página, 10 itens)
```bash
curl http://localhost:8080/produtos
```

Busca com filtro de nome
```bash
curl "http://localhost:8080/produtos?nome=notebook"
```

Busca com filtro de nome + categoria + paginação + ordenação
```bash
curl "http://localhost:8080/produtos?nome=notebook&categoriaId=1&page=0&size=5&ordenarPor=nome"
```

Relatório de produtos por categoria
```bash
curl http://localhost:8080/produtos/relatorio/por-categoria
```

Atualizar estoque (PATCH)
```bash
curl -X PATCH http://localhost:8080/produtos/1/estoque -H "Content-Type: application/json" -d "{\"estoque\": 50}"
```

---

### Mac / Linux

Busca paginada sem filtros (primeira página, 10 itens)
```bash
curl http://localhost:8080/produtos
```

Busca com filtro de nome
```bash
curl "http://localhost:8080/produtos?nome=notebook"
```

Busca com filtro de nome + categoria + paginação + ordenação
```bash
curl "http://localhost:8080/produtos?nome=notebook&categoriaId=1&page=0&size=5&ordenarPor=nome"
```

Relatório de produtos por categoria
```bash
curl http://localhost:8080/produtos/relatorio/por-categoria
```

Atualizar estoque (PATCH)
```bash
curl -X PATCH http://localhost:8080/produtos/1/estoque \
  -H "Content-Type: application/json" \
  -d '{"estoque": 50}'
```

---

## Resumo dos arquivos alterados

| Ação | Arquivo |
|---|---|
| **Criar** | `repository/ProdutoPorCategoriaProjection.java` |
| **Criar** | `dto/AtualizarEstoqueDTO.java` |
| **Criar** | `resources/import.sql` |
| **Substituir** | `resources/application.properties` |
| **Substituir** | `repository/ProdutoRepository.java` |
| **Substituir** | `service/ProdutoService.java` |
| **Substituir** | `controller/ProdutoController.java` |

---

## O que os alunos precisam fazer

1. Criar `ProdutoPorCategoriaProjection` no pacote `repository` (Passo 1)
2. Criar `AtualizarEstoqueDTO` no pacote `dto` (Passo 2)
3. Substituir `ProdutoRepository` com as duas `@Query` novas (Passo 3)
4. Substituir `ProdutoService` com os três novos métodos (Passo 4)
5. Substituir `ProdutoController` com os três novos endpoints (Passo 5)
6. Testar no Swagger com diferentes combinações de filtros e páginas

---

## Passo 6 — Dados de teste no banco

### Por que mudar o `ddl-auto`?

Com `create-drop` o banco é destruído e recriado a cada restart. Para testar paginação você precisa de pelo menos 11 produtos (para ter 2 páginas com `size=10`), e para o relatório precisa de produtos em categorias diferentes. Recriar tudo manualmente no Swagger a cada restart é inviável.

A solução é:
- Trocar `create-drop` por `create` → as tabelas são criadas uma vez e os dados persistem enquanto a aplicação estiver rodando
- Adicionar um arquivo `import.sql` → o **Hibernate** executa esse script logo após criar as tabelas, garantindo a ordem correta

> **Por que `import.sql` e não `data.sql`?** O projeto tem `postgresql` e `h2` no classpath ao mesmo tempo. Com dois drivers, o Spring Boot não consegue detectar automaticamente qual usar para executar o `data.sql`, rodando-o antes do Hibernate criar as tabelas. O `import.sql` é executado pelo próprio Hibernate — sempre após o DDL, independente do datasource.

> **Atenção:** com H2 em memória (`jdbc:h2:mem:...`) os dados ainda somem quando a aplicação é encerrada. A vantagem é não precisar reinserir nada durante a sessão de testes — só perde ao parar o servidor.

---

### 6.1 — Alterar `application.properties`

**Arquivo:** `src/main/resources/application.properties` — substituir o conteúdo completo

```properties
spring.application.name=ecommerce

# H2
spring.datasource.url=jdbc:h2:mem:ecommerce
spring.datasource.username=sa
spring.datasource.password=
spring.datasource.driver-class-name=org.h2.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=create
spring.jpa.show-sql=true

# H2 Console
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

---

### 6.2 — Criar `import.sql`

**Arquivo:** `src/main/resources/import.sql` — criar arquivo novo

```sql
INSERT INTO categoria (nome) VALUES ('Informática');
INSERT INTO categoria (nome) VALUES ('Periféricos');
INSERT INTO categoria (nome) VALUES ('Smartphones');

INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Notebook Dell i7', 'Notebook 16GB RAM 512GB SSD', 4999.99, 10, 1);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Notebook Lenovo i5', 'Notebook 8GB RAM 256GB SSD', 3299.99, 15, 1);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Notebook Apple M3', 'MacBook Air 8GB 256GB', 8999.99, 5, 1);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('SSD 1TB Kingston', 'SSD SATA 2.5 polegadas', 399.99, 30, 1);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Memória RAM 16GB', 'DDR4 3200MHz', 249.99, 20, 1);

INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Mouse Logitech MX', 'Mouse sem fio ergonômico', 299.99, 25, 2);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Teclado Mecânico Redragon', 'Switch Red, RGB', 349.99, 18, 2);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Monitor LG 27', 'Full HD 144Hz', 1499.99, 8, 2);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Headset HyperX', 'Surround 7.1 USB', 499.99, 12, 2);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Webcam Logitech C920', 'Full HD 1080p', 599.99, 7, 2);

INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('iPhone 15', '128GB Preto', 5999.99, 6, 3);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Samsung Galaxy S24', '256GB Azul', 4499.99, 9, 3);
INSERT INTO produto (nome, descricao, preco, estoque, id_categoria) VALUES ('Xiaomi Redmi Note 13', '128GB Branco', 1299.99, 22, 3);
```

> **Atenção:** o `import.sql` não suporta comentários com `--` em algumas versões do Hibernate. Mantenha o arquivo sem comentários como acima.

---

### O que testar após iniciar

**Paginação** — primeira página (5 itens):
```
GET /produtos?page=0&size=5
```

**Paginação** — segunda página:
```
GET /produtos?page=1&size=5
```

**Filtro por nome** — deve retornar notebooks:
```
GET /produtos?nome=notebook
```

**Filtro por categoria** — apenas periféricos (id=2):
```
GET /produtos?categoriaId=2
```

**Paginação com ordenação** — segunda página, ordenada por preço:
```
GET /produtos?page=1&size=5&ordenarPor=preco
```

**Relatório** — deve mostrar 3 categorias com 5, 5 e 3 produtos respectivamente:
```
GET /produtos/relatorio/por-categoria
```

**PATCH estoque** — atualizar estoque do produto 1:

Windows:
```bash
curl -X PATCH http://localhost:8080/produtos/1/estoque -H "Content-Type: application/json" -d "{\"estoque\": 99}"
```

Mac/Linux:
```bash
curl -X PATCH http://localhost:8080/produtos/1/estoque \
  -H "Content-Type: application/json" \
  -d '{"estoque": 99}'
```

---

## Conceitos abordados

- `Pageable` do Spring Data: encapsula `page`, `size` e `sort` automaticamente via parâmetros de URL
- `Page<T>`: objeto de resultado com `content`, `totalElements`, `totalPages`, `first`, `last`
- `@PageableDefault`: define valores padrão de paginação quando os parâmetros não são informados
- Projeções JPQL: como mapear resultados de queries agregadas sem criar novas entidades
- `PATCH` vs `PUT`: PATCH atualiza apenas um subconjunto dos campos do recurso
- `@RequestParam(required = false)`: parâmetro de URL opcional, recebido como `null` quando ausente
