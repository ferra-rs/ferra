# Ferra

**The API framework forged in Rust.**

Ferra is in active development (0.x). The source opens with the 1.0 release candidates.
To be notified, click **Watch → Custom → Releases** at the top of this page.

[Site](https://ferra.rs) · [Documentation](https://ferra.rs/docs/)

---

Ferra is a Rust framework for building HTTP APIs from a single source of truth:
a Sea-ORM entity with one derive, `#[derive(FerraModel)]`. From it Ferra derives,
at compile time, the routes, the query plans, the OpenAPI document, the
documentation page, the hypermedia links and the authorization checks. Change
the struct and everything downstream follows. If the generated code would be inconsistent,
the build fails.

The priorities, in order, are **security, developer experience, and the
experience of the AI agents that write code against Ferra**. Where they pull
apart, security wins: a model declares who may do what, and a model that
declares nothing does not compile.

## The problem Ferra solves

Building a conventional REST API means describing the same data structure
multiple times:

- once in a migration file (the database schema),
- once as a Rust struct (the model),
- once in each handler (request body, response type, SQL query),
- once in OpenAPI annotations (`#[utoipa::path(...)]`, response schemas),
- once in validation rules.

Every copy can drift from the others, and the compiler catches none of these
mismatches. Rename a field in the struct but not in the OpenAPI annotation, and
the documentation lies without a warning. Forget a validation rule in one
handler, and you open a security gap that nothing reports.

Ferra collapses the copies into one struct:

```rust
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel, FerraModel, Serialize, Deserialize)]
#[sea_orm(table_name = "films")]
#[ferra(auth = "bearer")] // every operation requires a valid token
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    #[field(min_length = 1, max_length = 255, filterable, sortable)]
    pub title: String,
    #[field(min = 1888, max = 2100)]
    pub year: i32,
}
```

## What you get

Mount the model on a `Foundry` and give it a JWT configuration
([Getting started](https://ferra.rs/docs/getting-started.html) has the complete
program, from the database connection to `axum::serve`):

```rust
let app = Foundry::new(conn)
    .with_auth::<MyClaims>(jwt)
    .mount::<film::Model>()
    .with_docs()
    .build();
```

You get all of the following without writing more code.

### CRUD routes

| Method   | Path          | Description |
|----------|---------------|-------------|
| `GET`    | `/films`      | Paginated collection, `?page=1&per_page=20` (`filter[...]`, `sort` and `cursor` for the fields that opt in) |
| `GET`    | `/films/{id}` | One resource by key |
| `POST`   | `/films`      | Create: `201` and a `Location` header; the server chooses the key |
| `PUT`    | `/films/{id}` | Update the row the path names: `200`, or `404` |
| `DELETE` | `/films/{id}` | Delete: `204`, or `404` |

### Authorization that refuses by default

Ferra checks authorization on every generated route. A model with no declared
posture does not compile, and a model that needs an identity does not mount on
a chain without `.with_auth()`. `#[authorize(...)]` states a rule per operation
(`IsOwner`, `HasRole(...)`, `Authenticated`, `Anyone`), and Ferra sets the
owner of a new row from the token.

### Hypermedia `_links` in the HAL representation

HAL is the default representation: every item response includes a `_links`
block, so clients and AI agents can navigate the API without knowing its URL
structure in advance. `Accept: application/json` gives the plain body.

```json
{
  "id":    1,
  "title": "Amélie",
  "year":  2001,
  "_links": {
    "self":       { "href": "/films/1" },
    "collection": { "href": "/films" }
  }
}
```

### An OpenAPI document that cannot drift

The OpenAPI 3.2 document is served at `/docs/openapi.json` and the interactive
Scalar page at `/docs`. Ferra derives both from the struct that also produces
the routes, so they cannot disagree with the running code, and you have no
annotations to forget.

### Typed SQL, bound by construction

Generated routes parameterize every query they run and have no mechanism to
put request data in SQL text. Every framework crate carries
`#![forbid(unsafe_code)]`.

### RFC 7807 errors, typed and machine-readable

Every error response follows RFC 7807 problem details. The `type` field is a URI
from a **closed** set that the framework keeps as a constant.

```json
{
  "type":   "https://ferra.rs/errors/not_found",
  "title":  "Resource Not Found",
  "status": 404,
  "detail": "films/42 not found"
}
```

### Validation, behaviors and security defaults

- `#[field(min_length, max_length, min, max, pattern, email, url, required)]`
  validates request bodies before the query runs; a violation answers `422`
  with every failing field.
- `#[behavior(timestampable, soft_delete)]` manages `created_at`, `updated_at`
  and `deleted_at`.
- Rate limiting on mutation routes, a 1 MiB body cap and restrictive CORS come
  out of the box.

## What Ferra is not

**Not an ORM.** Ferra generates the routes and the queries; you manage the
database schema with Sea-ORM.

**Not a general-purpose web framework.** No template engines, no session
handling, no WebSockets. To go further, add axum handlers next to the
Ferra routes.

**Not an identity provider.** Ferra verifies tokens (a shared HMAC secret, or an
EdDSA key set published as a JWKS); it does not issue them.

## License

Announced with the source.
