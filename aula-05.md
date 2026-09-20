# Aula 05 — Tratamento de exceções e organização das respostas da aplicação

## Objetivo

Substituir exceções genéricas por exceções de domínio tipadas, centralizar o tratamento de erros e configurar o CORS para permitir chamadas do frontend.

## Código Oficial

### Exceções de domínio

**Arquivo:** `src/main/java/br/unisinos/ecommerce/exception/RecursoNaoEncontradoException.java`
```java
package br.unisinos.ecommerce.exception;

public class RecursoNaoEncontradoException extends RuntimeException { 
	
    // 1. Construtor padrão sem argumentos
    public RecursoNaoEncontradoException() {
        super();
    }

    // 2. Construtor apenas com a mensagem de erro
    public RecursoNaoEncontradoException(String message) {
        super(message);
    }

    // 3. Construtor com a mensagem e a causa raiz (outra exceção)
    public RecursoNaoEncontradoException(String message, Throwable cause) {
        super(message, cause);
    }

    // 4. Construtor apenas com a causa raiz
    public RecursoNaoEncontradoException(Throwable cause) {
        super(cause);
    }
}
```

**Arquivo:** `src/main/java/br/unisinos/ecommerce/exception/RegraNegocioException.java`
```java
package br.unisinos.ecommerce.exception;

public class RegraNegocioException extends RuntimeException { 
	
    // 1. Construtor padrão sem argumentos
    public RegraNegocioException() {
        super();
    }

    // 2. Construtor apenas com a mensagem de erro
    public RegraNegocioException(String message) {
        super(message);
    }

    // 3. Construtor com a mensagem e a causa raiz (outra exceção)
    public RegraNegocioException(String message, Throwable cause) {
        super(message, cause);
    }

    // 4. Construtor apenas com a causa raiz
    public RegraNegocioException(Throwable cause) {
        super(cause);
    }
}
```

> `@StandardException` do Lombok gera automaticamente construtores com `String message` e `Throwable cause`.

### `GlobalExceptionHandler`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/exception/GlobalExceptionHandler.java`

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RecursoNaoEncontradoException.class)
    public ResponseEntity<String> tratarRecursoNaoEncontrado(RecursoNaoEncontradoException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }

    @ExceptionHandler(RegraNegocioException.class)
    public ResponseEntity<String> tratarRegraNegocio(RegraNegocioException ex) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> tratarValidacao(MethodArgumentNotValidException ex) {
        Map<String, String> erros = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
            .forEach(erro -> erros.put(erro.getField(), erro.getDefaultMessage()));
        return ResponseEntity.badRequest().body(erros);
    }
}
```

### Respostas HTTP padronizadas

| Exceção | Status HTTP |
|---|---|
| `RecursoNaoEncontradoException` | `404 Not Found` |
| `RegraNegocioException` | `400 Bad Request` |
| `MethodArgumentNotValidException` | `400 Bad Request` + mapa de erros por campo |

### `CorsConfig`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/config/CorsConfig.java`

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("http://localhost:5173")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*");
    }
}
```

> Permite que o frontend Vue (porta 5173) acesse a API sem bloqueio de CORS.

### Atualização nos services

Nos métodos `atualizar` de `CategoriaService` e `ProdutoService`, `RuntimeException` trocar a exceção genérica por `RecursoNaoEncontradoException`. 

Em `ProdutoService.salvar()`, adicionar validação explícita de `categoriaId` lançando `RegraNegocioException`.

## O que os alunos precisam fazer

1. Criar o pacote `exception` com `RecursoNaoEncontradoException` e `RegraNegocioException`
2. Criar `GlobalExceptionHandler` com `@RestControllerAdvice`
3. Criar `CorsConfig` no pacote `config`
4. **Atualizar** `CategoriaService` e `ProdutoService` para lançar as exceções novas
5. Testar no Swagger:
   - `GET /categorias/999` → deve retornar `404` com mensagem
   - `POST /produtos` com `categoriaId` inválido → deve retornar `400`
   - `POST /categorias` sem nome → deve retornar `400` com mapa de erros

## Conceitos abordados

- `@RestControllerAdvice`: intercepta exceções de todos os controllers
- `@ExceptionHandler(Tipo.class)`: mapeia cada tipo de exceção para uma resposta HTTP
- Hierarquia de exceções de domínio: separação entre "recurso não encontrado" e "regra de negócio violada"
- CORS: por que navegadores bloqueiam requisições cross-origin e como liberar
