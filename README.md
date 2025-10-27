# API de Posts, Comentários e Usuários

Plataforma REST construída com **Spring Boot**, **Spring Web**, **Spring Data MongoDB** e **Maven**, modelando um mini‐blog com **Usuários**, **Posts** e **Comentários**. O projeto demonstra uma arquitetura em camadas (Controller/Resource → Service → Repository) e o uso de **DTOs** para desacoplamento entre a API e o modelo de domínio.

---

## Stack & Requisitos

* **Linguagem:** Java (17+). Testado com versões mais novas do Java.
* **Framework:** Spring Boot (Spring Web, Spring Data MongoDB).
* **Banco de dados:** MongoDB.
* **Build:** Maven.
* **Estilo de API:** REST.

## Arquitetura e Fluxo de Requisições

```
[Cliente] → [Resource/Controller] → [Service] → [Repository] → [MongoDB]
                          ↑               ↓
                        [DTO] ← mapeia → [Domínio]
```

* **Resource (Controller):** recebe a requisição HTTP, valida/parâmetros, orquestra o serviço e devolve `ResponseEntity`.
* **Service:** contém **regras de negócio**, validação de existência, montagem de filtros e chamadas aos repositórios.
* **Repository:** interface com o **Spring Data MongoDB** para persistência e queries.
* **DTOs:** objetos de transporte expostos na API (evitam expor o domínio completo e controlam o payload).
* **Domínio:** classes persistidas (coleções do MongoDB).

---

## Modelagem de Dados

**Entidades principais**

* **User**: id, name, email e **referência** (`@DBRef(lazy = true)`) para seus posts.
* **Post**: id, date (Instant), title, body, **author** (como `AuthorDTO` embutido) e **comments** (lista de `CommentDTO`).

**Pontos-chave da modelagem**

* **Autor embutido no Post:** o `AuthorDTO` guarda apenas **id e name** do autor no momento do post. Isso **desnormaliza** o dado para leitura rápida e evita `JOIN/lookup` frequentes.
* **Comentários embutidos no Post:** `CommentDTO` (texto, data, autor) ficam **dentro do documento** do Post, alinhado ao uso comum no MongoDB.
* **Ligação User → Posts por `@DBRef`:** o `User` mantém uma lista de `Post` como referência. Útil para retornar “posts do usuário” sem duplicar dados do Post no User.

---

## Estrutura de Pacotes

* `config/` – rotinas de inicialização/seed.
* `domain/` – entidades de domínio persistidas no MongoDB.
* `dto/` – **Data Transfer Objects** consumidos/expostos pela API.
* `repository/` – repositórios Spring Data (persistência e consultas).
* `resources/` – controladores REST (camada Web da aplicação).
* `resources/util/` – utilitários usados pela camada de recursos.
* `services/` – regras de negócio e orquestração.
* `services/exception/` – exceções de serviço.

---

## Componentes

### Config/Seed: `Instantiation`

* Classe anotada com `@Configuration` e que implementa `CommandLineRunner`.
* Finalidade: **popular o banco** em cada start para facilitar testes (limpa e recria dados).
* Cria usuários (Maria, Alex, Bob), posts da Maria e comentários do Alex e Bob. Associa os posts à Maria.
* Utiliza `SimpleDateFormat` para strings como `dd/MM/yyyy` e converte para `Instant`.

### Domínio: `User` e `Post`

**User**

* Marcado com `@Document` (coleção do MongoDB).
* Campos básicos: id, name, email.
* `@DBRef(lazy = true) List<Post> posts`: referências de posts do usuário (carregadas sob demanda).

**Post**

* `@Document` com: id, date (`Instant`), title, body.
* **Autor embutido** (`AuthorDTO`) – snapshot do autor.
* **Comentários embutidos** (`List<CommentDTO>`): texto, data, autor do comentário.

### DTOs: `UserDTO`, `AuthorDTO`, `CommentDTO`

* **UserDTO**: expõe `id`, `name`, `email` para a API. Evita expor campos internos do `User` e facilita validação/mapeamento nas entradas de POST/PUT.
* **AuthorDTO**: forma **reduzida** do `User` para embutir no `Post` e em comentários (apenas `id` e `name`).
* **CommentDTO**: comentário embutido no `Post` (texto, data e autor `AuthorDTO`).

**Por que DTO?**

* **Desacoplamento** entre o modelo persistido e o payload externo.
* **Evolução** da API sem quebrar o domínio e vice‐versa.
* **Segurança** (não expor campos sensíveis acidentalmente).

### Repositórios: `UserRepository`, `PostRepository`

* Ambos estendem `MongoRepository<Entidade, String>`.
* **UserRepository**: CRUD padrão gerado pelo Spring Data.
* **PostRepository**: além do CRUD, define **consultas** por título e uma **full search** com filtros combinados:

  * **`findByTitleContainingIgnoreCase`** (derivada de nome – ignora caixa).
  * **`findByTitle`** com `@Query` usando regex case‐insensitive.
  * **`fullSearch`** com `@Query` combinando **intervalo de datas** e regex em **título/corpo/comentários**.

### Serviços: `UserService`, `PostService`

**Responsabilidades comuns**

* **Regras de negócio** e orquestração.
* **Validação** e tratamento de erros de “não encontrado” via `ObjectNotFoundException`.

**UserService**

* `findAll`, `findById`, `insert`, `delete`, `update`.
* `fromDTO(UserDTO)`: mapeia o DTO para a entidade de domínio.
* `updateData`: aplica mudanças controladas (name/email) no objeto persistido.

**PostService**

* `findById` com exceção quando ausente.
* Consultas: `findByTitle` (regex) e `fullSearch` (texto + data).
* **Observação de janela de datas**: há soma de 1 dia no `PostService` **e** no `PostResource` (ver abaixo). Isso resulta em uma **janela máxima +2 dias**. Padronize em **apenas um lugar** (Service **ou** Resource) para evitar sobre‐alcance no filtro.

### Recursos (Controllers): `UserResource`, `PostResource`

**UserResource** (`/users`)

* **GET /**: lista usuários como **`UserDTO`**.
* **GET /{id}**: retorna um usuário por id como **`UserDTO`**.
* **POST /**: cria usuário a partir de **`UserDTO`** e retorna **201 Created** com `Location` do recurso.
* **DELETE /{id}**: remove usuário por id (**204 No Content**).
* **PUT /{id}**: atualiza (name/email) a partir de **`UserDTO`** (**204 No Content**).
* **GET /{id}/posts**: retorna a lista de **posts** referenciados pelo usuário (via `@DBRef`).

**PostResource** (`/posts`)

* **GET /{id}**: busca post por id.
* **GET /titlesearch?text=...**: busca por título (regex, ignore case).
* **GET /fullsearch?text=...&minDate=...&maxDate=...**:

  * Decodifica `text` e converte `minDate`/`maxDate` (padrão `yyyy-MM-dd`).
  * **Soma 1 dia ao `maxDate` no controller** para incluir o dia final.
  * Chama o service para execução da consulta combinada.

### Utilitários: `resources.util.URL`

* `decodeParam`: decodifica parâmetros URL (UTF-8).
* `convertDate`: recebe texto `yyyy-MM-dd` e retorna **`Instant` no início do dia (UTC)**; se inválido, retorna valor padrão fornecido.

**Impacto de fuso e início do dia**

* O início do dia é calculado em **UTC**. Em sistemas com usuários em outros fusos, isso pode **incluir/excluir** registros nas bordas. Planeje conforme o público/servidor.

### Exceções: `ObjectNotFoundException`

* Exceção de serviço lançada quando um recurso não é encontrado.
* **Sugestão**: mapear para **HTTP 404** via um `@ControllerAdvice` (Handler global). Se não houver esse handler no projeto, a exceção pode virar **500** – considere adicionar um.

---

## Endpoints da API

### Users

* **GET** `/users` → Lista de usuários (DTO).
* **GET** `/users/{id}` → Usuário por id (DTO).
* **POST** `/users` → Cria usuário a partir de DTO. Retorna 201 com `Location`.
* **PUT** `/users/{id}` → Atualiza name/email via DTO. Retorna 204.
* **DELETE** `/users/{id}` → Remove usuário. Retorna 204.
* **GET** `/users/{id}/posts` → Posts associados ao usuário.

### Posts

* **GET** `/posts/{id}` → Post por id.
* **GET** `/posts/titlesearch?text=...` → Busca por título (regex case‐insensitive).
* **GET** `/posts/fullsearch?text=...&minDate=YYYY-MM-DD&maxDate=YYYY-MM-DD` → Busca combinada por texto (título, corpo e comentários) e intervalo de datas.

**Formato de datas**: `YYYY-MM-DD` (interpreta como **início do dia em UTC**).

---

## Busca por Título e Full Search

**Título**

* Implementada de duas formas no Repository: derivada por nome (`ContainingIgnoreCase`) e por `@Query` com **regex** e opção **case‐insensitive**.

**Full Search**

* Combina **intervalo de datas** (inclusive) com regex aplicada a **título**, **corpo** e **`comments.text`**.
* Observação: há **soma de 1 dia no Resource** e **no Service**. Padronize para somar **apenas uma vez** (recomenda‐se deixar **no Service**).

---

## Execução & Configuração

### Pré‐requisitos

* Java 17+ (LTS recomendado)
* Maven
* MongoDB em execução (local ou remoto)

### Configurar o MongoDB

* Defina a URI do banco na configuração da aplicação (por exemplo, variável `spring.data.mongodb.uri`).
* Se usar MongoDB local padrão (porta 27017), a URI padrão costuma funcionar.

### Rodando a aplicação

* Via Maven: executar a aplicação Spring Boot (por exemplo, alvo de execução do Maven/Spring Boot Run no IDE ou linha de comando).
* Ao subir, a classe **`Instantiation`** limpa e popula o banco para testes.

---

## Decisões de Design e Boas Práticas

1. **DTOs para fronteira da API**: mantêm a API estável, reduzem vazamento de detalhes do domínio e facilitam validação.
2. **Autor/Comentários embutidos** em `Post`: leitura rápida e alinhada ao modelo de documento do MongoDB. Ideal para **feed** e cenários de leitura majoritária.
3. **Referências do User para Posts**: útil para endpoint “posts por usuário” sem duplicar dados.
4. **Queries declarativas** com Spring Data: menos boilerplate e maior legibilidade.
5. **Conversão consistente de datas**: padronize **UTC** e **janela inclusiva** (somar 1 dia **em um único lugar**).
6. **Exceções significativas**: lance `ObjectNotFoundException` no Service e mapeie para 404 com handler global.
7. **Seed de dados**: acelera testes; em produção, desabilite ou condicione por perfil.

---
