# Aula 04 — Construção das operações completas de cadastro e manutenção de dados

## Objetivo

Implementar o CRUD completo de `Produto` usando o padrão DTO (Data Transfer Object) para separar a entrada e saída da API da entidade JPA.

### Por que usar DTOs?

A entidade `Produto` tem um relacionamento com `Categoria`. Se expusermos a entidade diretamente na API, o cliente precisaria enviar o objeto `Categoria` inteiro. Com DTOs:
- O cliente envia apenas o `categoriaId` (número)
- A API retorna os campos necessários com o nome da categoria embutido

## Código Oficial

### `ProdutoRequestDTO` — entrada

**Arquivo:** `src/main/java/br/unisinos/ecommerce/dto/ProdutoRequestDTO.java`

```java
public record ProdutoRequestDTO(
    @NotBlank String nome,
    @Size(max = 500) String descricao,
    @NotNull @DecimalMin(value = "0.0", inclusive = false) BigDecimal preco,
    @NotNull Integer estoque,
    @NotNull Long categoriaId
) {}
```

### `ProdutoResponseDTO` — saída

**Arquivo:** `src/main/java/br/unisinos/ecommerce/dto/ProdutoResponseDTO.java`

```java
public record ProdutoResponseDTO(
    Long id, String nome, String descricao,
    BigDecimal preco, Integer estoque,
    Long categoriaId, String nomeCategoria
) {}
```

### `ProdutoService` (métodos principais)

```java
package br.unisinos.ecommerce.service;

import java.util.List;
import java.util.Optional;

import org.springframework.stereotype.Service;

import br.unisinos.ecommerce.dto.ProdutoRequestDTO;
import br.unisinos.ecommerce.dto.ProdutoResponseDTO;
import br.unisinos.ecommerce.entity.Categoria;
import br.unisinos.ecommerce.entity.Produto;
import br.unisinos.ecommerce.repository.CategoriaRepository;
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
            .orElseThrow(() -> new RuntimeException("Categoria não encontrada"));
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
            .orElseThrow(() -> new RuntimeException("Produto não encontrado"));
        Categoria categoria = categoriaRepository.findById(dto.categoriaId())
            .orElseThrow(() -> new RuntimeException("Categoria não encontrada"));
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

}
```

### `ProdutoController`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/controller/ProdutoController.java`

```java
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
	
    @GetMapping
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
    public ResponseEntity<ProdutoResponseDTO> atualizar(@PathVariable Long id, @Valid @RequestBody ProdutoRequestDTO dto) {
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
}
```

### Testes: 

Criar produto (POST)
```bash
  curl -X POST http://localhost:8080/produtos \
    -H "Content-Type: application/json" \
    -d '{"nome": "Notebook Dell", "descricao": "Notebook i7 16GB RAM", "preco":
  4999.99, "estoque": 10, "categoriaId": 1}'
```

Listar todos (GET)
```bash
  curl http://localhost:8080/produtos
```

Buscar por ID (GET)
```bash
  curl http://localhost:8080/produtos/1
```  

Atualizar (PUT)
```bash
  curl -X PUT http://localhost:8080/produtos/1 \
    -H "Content-Type: application/json" \
    -d '{"nome": "Notebook Dell XPS", "descricao": "Notebook i9 32GB RAM",
  "preco": 7999.99, "estoque": 5, "categoriaId": 1}'
```

Excluir (DELETE)
```bash  
  curl -X DELETE http://localhost:8080/produtos/1
```

## Endpoints disponíveis após esta aula

| Método | URL | Corpo | Resposta |
|---|---|---|---|
| `POST` | `/produtos` | `ProdutoRequestDTO` | `ProdutoResponseDTO` |
| `GET` | `/produtos` | — | `List<ProdutoResponseDTO>` |
| `GET` | `/produtos/{id}` | — | `ProdutoResponseDTO` |
| `PUT` | `/produtos/{id}` | `ProdutoRequestDTO` | `ProdutoResponseDTO` |
| `DELETE` | `/produtos/{id}` | — | `204 No Content` |

## O que os alunos precisam fazer

1. Criar o pacote `dto` com `ProdutoRequestDTO` e `ProdutoResponseDTO` como **records**
2. Criar `ProdutoService` com os métodos `salvar`, `atualizar`, `listarTodos`, `buscarPorId`, `excluir`
3. Criar `ProdutoController` com todos os endpoints
4. Testar criando um produto via Swagger (é necessário ter uma categoria criada antes)

## Conceitos abordados

- **Java Records**: classes imutáveis e compactas para DTOs
- Padrão **Request/Response DTO**: separação entre o que a API recebe e o que ela retorna
- Método `toResponseDTO()`: mapeamento manual entidade → DTO dentro do service
- `@Builder` do Lombok: construção de `Produto` a partir dos campos do DTO
