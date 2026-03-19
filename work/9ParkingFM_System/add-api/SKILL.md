---
name: add-api
description: "Add a new API endpoint to an existing Controller. Use this skill whenever the user says /add-api, or asks to add an endpoint, create an API route, add a controller action, wire up a command/query to a controller, or expose a use case as an HTTP endpoint. Also trigger when the user describes an API they want (e.g., 'I need a POST endpoint for creating orders', 'add a delete route for products') — even without using the exact command name."
---

# Add API Endpoint

Add a new endpoint to an existing WebApi Controller, wiring it to an Application-layer Command or Query via MassTransit Mediator.

## What the user provides

The user will describe (orally):
1. Which **Command or Query** to wire up (e.g., `CreateOrderCommand`)
2. The **HTTP method** (GET, POST, PUT, DELETE)
3. The **route path** (e.g., `order/{orderId}`)

If the user forgets any of these, prompt them:
- Missing Command/Query → "Which Command or Query should this endpoint dispatch?"
- Missing HTTP method → "What HTTP method? (GET / POST / PUT / DELETE)"
- Missing route → "What should the route path be? (e.g., `order/{orderId}`)"

## Step 1: Read context

1. **The Command/Query record** — to know the input properties and Response type
2. **The target Controller** — find the existing controller file for this feature area. Look in `WebApi/Controllers/`
3. **Existing endpoints** — read the controller to match its style and avoid route conflicts

## Step 2: Generate the endpoint

### Controller pattern

The project uses MassTransit Mediator (not MediatR):

```csharp
[Http{Method}("{route}")]
public async Task<IActionResult> {ActionName}({parameters})
{
    var client = _mediator.CreateRequestClient<{CommandOrQuery}>();
    var response = await client.GetResponse<{ResponseType}>({message});
    return Ok(response.Message);
}
```

### Parameter binding rules

| Source | Attribute | When to use |
|---|---|---|
| Route | `[FromRoute]` | ID parameters in the URL path (e.g., `{orderId}`) |
| Body | No attribute (default) or `[FromBody]` | POST/PUT command objects |
| Query string | `[FromQuery]` | GET filter/search parameters |

### Pattern by HTTP method

**POST** (Create) — body binding:
```csharp
[HttpPost("{route}")]
public async Task<IActionResult> {Action}({CommandType} command)
{
    var client = _mediator.CreateRequestClient<{CommandType}>();
    var response = await client.GetResponse<{ResponseType}>(command);
    return Ok(response.Message);
}
```

**PUT** (Update) — body binding:
```csharp
[HttpPut("{route}")]
public async Task<IActionResult> {Action}([FromBody] {CommandType} command)
{
    var client = _mediator.CreateRequestClient<{CommandType}>();
    var response = await client.GetResponse<{ResponseType}>(command);
    return Ok(response.Message);
}
```

**DELETE** — route parameter, construct command:
```csharp
[HttpDelete("{route}/{id}")]
public async Task<IActionResult> {Action}([FromRoute] int {id})
{
    var client = _mediator.CreateRequestClient<{CommandType}>();
    var response = await client.GetResponse<{ResponseType}>(new {CommandType}({id}));
    return Ok(response.Message);
}
```

**GET** (no params) — construct empty query:
```csharp
[HttpGet("{route}")]
public async Task<IActionResult> {Action}()
{
    var client = _mediator.CreateRequestClient<{QueryType}>();
    var response = await client.GetResponse<{ResponseType}>(new {QueryType}());
    return Ok(response.Message);
}
```

**GET** (with route param) — route parameter, construct query:
```csharp
[HttpGet("{route}/{id}")]
public async Task<IActionResult> {Action}([FromRoute] int {id})
{
    var client = _mediator.CreateRequestClient<{QueryType}>();
    var response = await client.GetResponse<{ResponseType}>(new {QueryType}({id}));
    return Ok(response.Message);
}
```

## Step 3: Add using statements

If the Command/Query namespace isn't already imported in the controller, add it.

## Step 4: Confirm and write

Show the user:
- The new endpoint code
- Where it will be inserted in the controller
- Any new `using` statements

Wait for confirmation before modifying the file.
