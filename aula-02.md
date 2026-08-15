# Aula 02 — Modelagem de dados e persistência em banco

## Objetivo

Modelar as entidades do sistema (`Categoria` e `Produto`), mapear o relacionamento entre elas e criar os repositórios JPA.

![](./aula-02-diagrama-classes.png)

## Banco de dados em memória

H2 é um banco de dados em memória que será utilizado para persistir os dados da aplicação. Ele é útil para testes e desenvolvimento, pois não requer configuração de um banco externo.

Maven dependency, alterar o `pom.xml`:

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

**Arquivo:** `src/main/resources/application.properties`:

```bash
# H2
spring.datasource.url=jdbc:h2:mem:ecommerce
spring.datasource.username=sa
spring.datasource.password=
spring.datasource.driver-class-name=org.h2.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true

# H2 Console
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

## Primeiro experimento(realizado na aula 02):

#### PARTE 1 - Criando entidade e camada de repository

**Arquivo:** `src/main/java/br/unisinos/ecommerce/entity/Produto.java`

``` java
package br.unisinos.ecommerce.entity;

import java.math.BigDecimal;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;

@Entity
public class Produto {
	
	@Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nome;

    private BigDecimal preco;
	
    public Produto() {
		super();
	}

	public Produto(String nome, BigDecimal preco) {
		super();
		this.nome = nome;
		this.preco = preco;
	}

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	public String getNome() {
		return nome;
	}

	public void setNome(String nome) {
		this.nome = nome;
	}

	public BigDecimal getPreco() {
		return preco;
	}

	public void setPreco(BigDecimal preco) {
		this.preco = preco;
	}

}
```

**Arquivo:** `repository/ProdutoRepository.java`

``` java
package br.unisinos.ecommerce.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import br.unisinos.ecommerce.entity.Produto;

@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> { }
```

#### PARTE 2 - executando

Executando código depois que o Spring inicia

Para realizar uma demonstração de persistência, podemos utilizar
`CommandLineRunner`.

``` java
package br.unisinos.ecommerce;

import java.math.BigDecimal;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import br.unisinos.ecommerce.entity.Produto;
import br.unisinos.ecommerce.repository.ProdutoRepository;

@SpringBootApplication
public class EcommerceApplication {

	public static void main(String[] args) {
		SpringApplication.run(EcommerceApplication.class, args);
	}
	
	@Bean
    CommandLineRunner initDatabase(ProdutoRepository produtoRepository) {
        return args -> {

        	Produto produto = new Produto("Notebook", "computador", BigDecimal.valueOf(144.00), 10, null); 
           
            produtoRepository.save(produto);
        };
    }
}
```

## Código Oficial

Vamos criar as entidades `CATEGORIA` e `PRODUTO` com relacionamento bidirecional e depois criar os repositórios JPA para cada uma delas.

### Entidade `Categoria`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/entity/Categoria.java`

```java
package br.unisinos.ecommerce.entity;

import java.util.List;

import com.fasterxml.jackson.annotation.JsonManagedReference;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import jakarta.validation.constraints.NotBlank;

@Entity
@Table(name = "categoria")
public class Categoria {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "O nome da categoria é obrigatório")
    @Column(nullable = false, length = 100)
    private String nome;

    @OneToMany(mappedBy = "categoria")
    @JsonManagedReference
    private List<Produto> produtos;

	public Categoria(@NotBlank(message = "O nome da categoria é obrigatório") String nome,
			List<Produto> produtos) {
		super();
		this.nome = nome;
		this.produtos = produtos;
	}

	public Categoria() {
		super();
	}

	public String getNome() {
		return nome;
	}

	public void setNome(String nome) {
		this.nome = nome;
	}

	public List<Produto> getProdutos() {
		return produtos;
	}

	public void setProdutos(List<Produto> produtos) {
		this.produtos = produtos;
	}    
    
}
```

### Entidade `Produto`

**Arquivo:** `src/main/java/br/unisinos/ecommerce/entity/Produto.java`

```java
package br.unisinos.ecommerce.entity;

import java.math.BigDecimal;

import com.fasterxml.jackson.annotation.JsonBackReference;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.ForeignKey;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

@Entity
@Table(name = "produto")
public class Produto {
	
	@Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(nullable = false, length = 100)
    private String nome;

    @Size(max = 500)
    private String descricao;

    @NotNull
    @DecimalMin(value = "0.0", inclusive = false)
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal preco;

    @NotNull
    @Column(nullable = false)
    private Integer estoque;

    @ManyToOne
    @JoinColumn(name = "id_categoria", foreignKey = @ForeignKey(name = "fk_produto_categoria"))
    @JsonBackReference
    private Categoria categoria;
    
    
    public Produto() {
		super();
	}

	public Produto(@NotBlank String nome, @Size(max = 500) String descricao,
			@NotNull @DecimalMin(value = "0.0", inclusive = false) BigDecimal preco, @NotNull Integer estoque,
			Categoria categoria) {
		super();
		this.nome = nome;
		this.descricao = descricao;
		this.preco = preco;
		this.estoque = estoque;
		this.categoria = categoria;
	}

	public String getNome() {
		return nome;
	}

	public void setNome(String nome) {
		this.nome = nome;
	}

	public String getDescricao() {
		return descricao;
	}

	public void setDescricao(String descricao) {
		this.descricao = descricao;
	}

	public BigDecimal getPreco() {
		return preco;
	}

	public void setPreco(BigDecimal preco) {
		this.preco = preco;
	}

	public Integer getEstoque() {
		return estoque;
	}

	public void setEstoque(Integer estoque) {
		this.estoque = estoque;
	}

	public Categoria getCategoria() {
		return categoria;
	}

	public void setCategoria(Categoria categoria) {
		this.categoria = categoria;
	}
}
```

### Repositórios

**Arquivo:** `src/main/java/br/unisinos/ecommerce/repository/CategoriaRepository.java`

```java
package br.unisinos.ecommerce.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import br.unisinos.ecommerce.entity.Categoria;

@Repository
public interface CategoriaRepository extends JpaRepository<Categoria, Long> { }
```

**Arquivo:** `src/main/java/br/unisinos/ecommerce/repository/ProdutoRepository.java`

``` java
package br.unisinos.ecommerce.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import br.unisinos.ecommerce.entity.Produto;

@Repository
public interface ProdutoRepository extends JpaRepository<Produto, Long> { }
```

`JpaRepository<T, ID>` já fornece: `save()`, `findById()`, `findAll()`, `deleteById()`, entre outros.

### Testando o produto

```java
package br.unisinos.ecommerce;

import java.math.BigDecimal;
import java.util.List;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

import br.unisinos.ecommerce.entity.Produto;
import br.unisinos.ecommerce.repository.ProdutoRepository;

@SpringBootApplication
public class EcommerceApplication {

	public static void main(String[] args) {
		SpringApplication.run(EcommerceApplication.class, args);
	}
	
	@Bean
    CommandLineRunner initDatabase(ProdutoRepository produtoRepository) {
        return args -> {

        	Produto produto = new Produto("Notebook", "computador", BigDecimal.valueOf(144.00), 10, null); 
           
            produtoRepository.save(produto);
            
            List<Produto> listaProdutos = produtoRepository.findAll();
            
            for(Produto p : listaProdutos) {
            	System.out.println(p.getNome());
            }
        };
    }
}
```

## O que os alunos precisam fazer

1. Criar o pacote `entity` dentro de `br.unisinos.ecommerce`
2. Criar a classe `Categoria` com os campos e anotações mostrados
3. Criar a classe `Produto` com relacionamento `@ManyToOne` para `Categoria`
4. Criar o pacote `repository` e as interfaces `CategoriaRepository` e `ProdutoRepository`
5. Subir a aplicação e verificar no banco que as tabelas `categoria` e `produto` foram criadas

## Conceitos abordados

| Anotação | Significado |
|---|---|
| `@Entity` | Marca a classe como tabela JPA |
| `@Table(name=)` | Define o nome da tabela |
| `@Id` + `@GeneratedValue` | Chave primária auto-incrementada |
| `@Column` | Customiza a coluna (nullable, length) |
| `@ManyToOne` / `@OneToMany` | Relacionamento bidirecional |
| `@JsonManagedReference` / `@JsonBackReference` | Evita recursão infinita na serialização JSON |
| Lombok `@Builder` | Permite construção fluente de objetos |

### Código desenvolvido em aula 

> Disponibilizado no Moodle da disciplina
