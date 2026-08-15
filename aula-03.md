# Aula 03 — Implementação da lógica de negócio e primeiras operações do sistema

## Objetivo

Implementar a camada de serviço e o controller REST para `Categoria`, expondo um CRUD completo via HTTP.

## Código Oficial

Criar o pacote `service` e a classe `CategoriaService` com todos os métodos. Criar o pacote `controller` e a classe `CategoriaController` com todos os endpoints.

### `CategoriaService`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/service/CategoriaService.java`

```java
package br.unisinos.ecommerce.service;

import java.util.List;
import java.util.Optional;

import org.springframework.stereotype.Service;

import br.unisinos.ecommerce.entity.Categoria;
import br.unisinos.ecommerce.repository.CategoriaRepository;

@Service
public class CategoriaService {

    public CategoriaService(CategoriaRepository categoriaRepository) {
		super();
		this.categoriaRepository = categoriaRepository;
	}

	private final CategoriaRepository categoriaRepository;

    public Categoria salvar(Categoria categoria) {
        return categoriaRepository.save(categoria);
    }

    public List<Categoria> listarTodas() {
        return categoriaRepository.findAll();
    }

    public Optional<Categoria> buscarPorId(Long id) {
        return categoriaRepository.findById(id);
    }

    public Categoria atualizar(Long id, Categoria categoriaAtualizada) {
        Categoria categoria = categoriaRepository.findById(id)
            .orElseThrow(() -> new RuntimeException("Categoria não encontrada"));
        categoria.setNome(categoriaAtualizada.getNome());
        return categoriaRepository.save(categoria);
    }

    public void excluir(Long id) {
        categoriaRepository.deleteById(id);
    }
}
```

> Nota: nesta aula ainda usamos `RuntimeException` genérica — na Aula 05 isso será substituído por exceções específicas.

### `CategoriaController`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/controller/CategoriaController.java`

```java
package br.unisinos.ecommerce.controller;

import java.util.List;
import java.util.Optional;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import br.unisinos.ecommerce.entity.Categoria;
import br.unisinos.ecommerce.service.CategoriaService;
import jakarta.validation.Valid;

@RestController
@RequestMapping("/categorias")
public class CategoriaController {

    private final CategoriaService categoriaService;

    public CategoriaController(CategoriaService categoriaService) {
		super();
		this.categoriaService = categoriaService;
	}

	@PostMapping
    public ResponseEntity<Categoria> salvar(@Valid @RequestBody Categoria categoria) {
		this.categoriaService.salvar(categoria);
		return new ResponseEntity<Categoria>(categoria, HttpStatus.ACCEPTED);
	}
	
    @GetMapping
    public ResponseEntity<List<Categoria>> listarTodos() {
    	List<Categoria> listaCategoria = this.categoriaService.listarTodas();
    	return new ResponseEntity<List<Categoria>>(listaCategoria, HttpStatus.OK);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Categoria> buscarPorId(@PathVariable Long id) {
        
        Optional<Categoria> categoria = categoriaService.buscarPorId(id);

        if (categoria.isPresent()) {
            return ResponseEntity.ok(categoria.get());
        }

        return ResponseEntity.notFound().build();
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<Categoria> atualizar(@PathVariable Long id, @Valid @RequestBody Categoria categoria) {
        Categoria categoriaAtualizado = categoriaService.atualizar(id, categoria);
        return ResponseEntity.ok(categoriaAtualizado);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> excluir(@PathVariable Long id) {
        
        Optional<Categoria> categoria = categoriaService.buscarPorId(id);

        if (categoria.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        categoriaService.excluir(id);

        return ResponseEntity.noContent().build();
    }
}
```

### Testes

Criar categoria (POST)
```bash
  curl -X POST http://localhost:8080/categorias \
    -H "Content-Type: application/json" \
    -d '{"nome": "Eletrônicos"}'
```    

Listar todas (GET)
```bash
  curl http://localhost:8080/categorias
```

Buscar por ID (GET)
```bash
  curl http://localhost:8080/categorias/1
```

Atualizar (PUT)
```bash
  curl -X PUT http://localhost:8080/categorias/1 \
    -H "Content-Type: application/json" \
    -d '{"nome": "Eletrônicos e Informática"}'
```

Excluir (DELETE)
```bash
  curl -X DELETE http://localhost:8080/categorias/1
```

## Endpoints disponíveis após esta aula

| Método | URL | Ação |
|---|---|---|
| `POST` | `/categorias` | Cria categoria |
| `GET` | `/categorias` | Lista todas |
| `GET` | `/categorias/{id}` | Busca por ID |
| `PUT` | `/categorias/{id}` | Atualiza |
| `DELETE` | `/categorias/{id}` | Remove |

## O que os alunos precisam fazer

1. Criar o pacote `service` e a classe `CategoriaService` com todos os métodos
2. Criar o pacote `controller` e a classe `CategoriaController` com todos os endpoints

## Conceitos abordados

- `@Service`: registra a classe como bean gerenciado pelo Spring
- `@RestController`: combina `@Controller` + `@ResponseBody`
- `@RequestMapping`: define o prefixo de URL do controller
- `@Valid`: ativa Bean Validation no objeto recebido
- `ResponseEntity<T>`: permite controlar o status HTTP e o corpo da resposta
- `@RequiredArgsConstructor` do Lombok: injeta dependências via construtor automaticamente
