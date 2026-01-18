# ASP.NET Core Web API Core Development Notes

## 1. Project Startup and Pipeline (Program.cs)
ASP.NET Core uses the **Builder Pattern** to configure applications.
*   **WebApplication.CreateBuilder**: Initializes configuration, logging, and the DI container.
*   **Services**: Registers services (e.g., `AddOpenApi`, `AddDbContext`) before `Build()`.
*   **Middleware**: Configures the request processing pipeline on `app` (e.g., `UseHttpsRedirection`).
*   **app.Run()**: Starts the Kestrel server and begins listening for HTTP requests.

---

## 2. Database Core Concepts (EF Core)
### 2.1 DbContext & DbSet
*   **DbContext**: The "steward" of the database, responsible for connection management and transaction commits.
*   **DbSet<'T'>**: Represents **a single table** in the database. It's not just a container but also the starting point for **IQueryable** queries.
*   **Attribute**: Used to precisely control how C# properties map to database columns (e.g., changing column names, specifying `decimal(18,2)` precision).

### 2.2 Migrations
*   **Essence**: A "version control system" for the database.
*   **Process**: Modify Model -> `add migration` (generates code) -> `update database` (synchronizes SQL Server).
*   **Enterprise Practice**: Direct updates in development environments; export **SQL scripts** for auditing and deployment in production.

---

## 3. Controllers and Routing (Controllers)
*   **[ApiController]**: Enables automatic model validation and binding source inference (automatically finds data from Body/Route).
*   **[Route]**: Defines the controller's access "address," commonly `[Route("api/[controller]")]`.
*   **IActionResult**: A common return type for API methods, used with `Ok()` (200), `NotFound()` (404), `CreatedAtAction()` (201).

---

## 4. Data Transfer Objects (DTO)
*   **Purpose**:
    1.  **Security**: Hides sensitive database fields (e.g., passwords).
    2.  **Performance**: Only transfers fields required by the frontend.
    3.  **Decoupling**: Database schema changes do not directly break the API contract.
*   **Process**: Client -> `RequestDTO` -> `Model` -> Database -> `Model` -> `ResponseDTO` -> Client.

---

## 5. Asynchronous Programming (Async/Await)
*   **Principle**: When encountering `await`, the thread is released back to the thread pool to handle other requests until the IO operation (e.g., querying the database) completes.
*   **Execution Order**:
    ```csharp
    var data = await _context.Stock.ToListAsync(); // 1. Only after this step is fully completed
    var dto = data.Select(...);                    // 2. will this step be executed
    ```
