# HTTP Client: Inventory Sync Client

## Concept

When communicating with external REST APIs, handling JSON encoding/decoding and HTTP status codes correctly is essential. Just like we built the server previously, now we are building the client that talks to it.

**The Golden Rule of Go HTTP Clients:** You should **never use the default HTTP client** (`http.Get`, `http.Post`) in production. The default client has _no timeout_, meaning if the server hangs, your application will hang forever waiting for a response! Instead, you must always instantiate a custom `http.Client`.

Finally, dealing with networks often means dealing with latency. If you need to fetch 10 items, doing it one by one is slow. Go makes it incredibly easy to issue multiple HTTP requests simultaneously using **Goroutines**. Think of a `sync.WaitGroup` like a school field-trip chaperone—they wait at the exit until every single student (Goroutine) has returned before letting the buses leave.

- **`client := &http.Client{Timeout: 5 * time.Second}`**: Creates a safe HTTP client.
- **`client.Get(url)`**: Issues a `GET` request using the custom client.
- **`json.NewDecoder(resp.Body).Decode(&obj)`**: Decodes a JSON response back into a Go struct.
- **`sync.WaitGroup`**: Wait for a collection of Goroutines to finish their work.
- **`sync.Mutex`**: Safely lock shared data (like a Map) so two Goroutines don't try to write to it at the exact same microsecond, which would cause a panic.

## References

- [Go by Example: HTTP Clients](https://gobyexample.com/http-clients)
- [Go by Example: WaitGroups](https://gobyexample.com/waitgroups)
- [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)

## Quest

### Objective

Build an **Inventory Sync Client** that communicates with a Product Catalog API across four functions.

### Requirements

**All functions** use a custom client with a 5-second timeout: `&http.Client{Timeout: 5 * time.Second}`

---

**`FetchProducts(apiURL string) ([]Product, error)`**

| Step | Action                                  |
| ---- | --------------------------------------- |
| 1    | `GET` → `apiURL + "/products"`          |
| 2    | Return error if status ≠ 200 OK         |
| 3    | Decode JSON into `[]Product` and return |

---

**`CreateProduct(apiURL string, p Product) error`**

| Step | Action                                                                 |
| ---- | ---------------------------------------------------------------------- |
| 1    | Encode `p` to JSON using `bytes.Buffer` + `json.NewEncoder()`          |
| 2    | `POST` → `apiURL + "/products"` with `"application/json"` content type |
| 3    | Return error if status ≠ 201 Created (or 200 OK)                       |

---

**`FetchProduct(apiURL string, id string) (Product, bool, error)`**

| Status Code                   | Return                           |
| ----------------------------- | -------------------------------- |
| 200 OK                        | decoded `Product`, `true`, `nil` |
| 404 Not Found                 | empty `Product`, `false`, `nil`  |
| anything else / network error | zero value, `false`, `error`     |

---

**`FetchMultipleProducts(apiURL string, ids []string) (map[string]Product, error)`**

| Step | Action                                                                       |
| ---- | ---------------------------------------------------------------------------- |
| 1    | Launch one Goroutine per ID using a `sync.WaitGroup`                         |
| 2    | Inside each, call `FetchProduct` — skip 404s and errors                      |
| 3    | Use a `sync.Mutex` to safely insert found products into `map[string]Product` |

### Inputs

- `apiURL`: The full URL to the Product Catalog API (e.g., `"http://localhost:8080"`).
- `id` (for `FetchProduct`): The ID of the product to fetch.
- `ids` (for `FetchMultipleProducts`): A slice of IDs to fetch concurrently.
- `p` (for `CreateProduct`): The `Product` object to create.

### Outputs

- `FetchProducts`: Returns `[]Product` and `error`.
- `CreateProduct`: Returns an `error` if the creation failed or the API returned non-success.
- `FetchProduct`: Returns `(Product, bool, error)`, where the boolean indicates if the product was found (200 OK vs 404 Not Found).
- `FetchMultipleProducts`: Returns a `map[string]Product` of the successful results.

### Models

```go
type Product struct {
	ID    string  `json:"id"`
	Name  string  `json:"name"`
	Price float64 `json:"price"`
}
```

### Testing

```bash
go test -v ./quests/028.http_client
```

Or from the quest directory:

```bash
go test -v
```

Expected output:

```text
=== RUN   TestInventorySyncClient
=== RUN   TestInventorySyncClient/FetchProducts
=== RUN   TestInventorySyncClient/CreateProduct_Success
=== RUN   TestInventorySyncClient/CreateProduct_Failure
=== RUN   TestInventorySyncClient/FetchProduct_Found
=== RUN   TestInventorySyncClient/FetchProduct_NotFound
=== RUN   TestInventorySyncClient/FetchMultipleProducts
--- PASS: TestInventorySyncClient (0.00s)
    --- PASS: TestInventorySyncClient/FetchProducts (0.00s)
    --- PASS: TestInventorySyncClient/CreateProduct_Success (0.00s)
    --- PASS: TestInventorySyncClient/CreateProduct_Failure (0.00s)
    --- PASS: TestInventorySyncClient/FetchProduct_Found (0.00s)
    --- PASS: TestInventorySyncClient/FetchProduct_NotFound (0.00s)
    --- PASS: TestInventorySyncClient/FetchMultipleProducts (0.00s)
PASS
```
