---
title: Response
weight: 4
---

## Response Classes Overview

ODY's HTTP component includes several classes for handling responses:

- `Response`: The main PSR-7 compatible HTTP response class with additional convenience methods
- `JsonResponse`: A helper class for creating JSON responses
- `Stream`: Represents response bodies as streams
- `ResponseEmitter`: Handles sending responses to the client
- `SwooleResponseEmitter`: Specialized response emitter for Swoole environments

## Basic Response Handling in Controllers

In ODY controllers, you always return a response object that implements `Psr\Http\Message\ResponseInterface`. Here's a 
basic example:

```php
/**
 * Get a specific resource
 *
 * @param ServerRequestInterface $request
 * @param ResponseInterface $response
 * @param array $params
 * @return ResponseInterface
 */
public function show(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    // Process the request and get data
    $data = ['id' => (int)$id, 'name' => 'Example'];
    
    // Return a JSON response
    return $this->jsonResponse($response, $data);
}
```

## Creating Responses

### Basic Responses

To create a basic response with text content:

```php
// Using the response object passed to your controller
$response = $response->withHeader('Content-Type', 'text/plain');
$response->getBody()->write('Hello, world!');
return $response;
```

With the ODY `Response` class, you can use convenience methods:

```php
// Using the ODY Response fluent interface
return $response
    ->text()  // Sets Content-Type: text/plain
    ->body('Hello, world!');
```

### JSON Responses

JSON responses are common in API development. Here's how to create them:

```php
// Standard PSR-7 way
$response = $response->withHeader('Content-Type', 'application/json');
$response->getBody()->write(json_encode($data));
return $response;
```

ODY provides more convenient methods:

```php
// Using the ODY Response class
return $response->withJson($data);

// Or using the fluent interface
return $response
    ->json()  // Sets Content-Type: application/json
    ->body(json_encode($data));
```

### HTML Responses

For returning HTML content:

```php
// Standard PSR-7 way
$response = $response->withHeader('Content-Type', 'text/html');
$response->getBody()->write('<html><body><h1>Hello</h1></body></html>');
return $response;
```

ODY's fluent interface:

```php
// Using the ODY Response fluent interface
return $response
    ->html()  // Sets Content-Type: text/html
    ->body('<html><body><h1>Hello</h1></body></html>');
```

## Setting HTTP Status Codes

Set the HTTP status code for your response:

```php
// Standard PSR-7 way
$response = $response->withStatus(404);

// Using ODY's fluent interface
$response = $response->status(404);
```

Common HTTP status codes:

- `200`: OK (Success)
- `201`: Created
- `204`: No Content
- `400`: Bad Request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not Found
- `422`: Unprocessable Entity
- `500`: Internal Server Error

## Response Headers

### Setting Headers

Add headers to your response:

```php
// Standard PSR-7 way
$response = $response->withHeader('Content-Type', 'application/json');

// Add a header without removing existing values with the same name
$response = $response->withAddedHeader('Cache-Control', 'no-cache');

// Remove a header
$response = $response->withoutHeader('X-Powered-By');
```

Using ODY's fluent interface:

```php
$response = $response->header('Content-Type', 'application/json');
```

### Common Headers

Here are some commonly used headers:

```php
// CORS headers
$response = $response
    ->withHeader('Access-Control-Allow-Origin', '*')
    ->withHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')
    ->withHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');

// Caching headers
$response = $response
    ->withHeader('Cache-Control', 'no-store, no-cache, must-revalidate')
    ->withHeader('Pragma', 'no-cache')
    ->withHeader('Expires', '0');

// Content security
$response = $response
    ->withHeader('Content-Security-Policy', "default-src 'self'")
    ->withHeader('X-Content-Type-Options', 'nosniff')
    ->withHeader('X-Frame-Options', 'DENY');
```

## Working with Response Body

The response body is represented as a PSR-7 `StreamInterface`. Here are different ways to work with it:

```php
// Write to the body
$response->getBody()->write('Hello, world!');

// Replace the entire body with a new stream
$newStream = $streamFactory->createStream('New content');
$response = $response->withBody($newStream);
```

The ODY `Response` class provides a simpler method:

```php
// Set the body content
$response = $response->body('Hello, world!');
```

## Response Examples for Different Scenarios

### Success Response

```php
public function index(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $users = [
        ['id' => 1, 'name' => 'John Doe'],
        ['id' => 2, 'name' => 'Jane Smith'],
    ];
    
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withStatus(200)
        ->withBody(json_encode([
            'status' => 'success',
            'data' => $users
        ]));
}
```

Using ODY's `Response` class:

```php
public function index(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $users = [
        ['id' => 1, 'name' => 'John Doe'],
        ['id' => 2, 'name' => 'Jane Smith'],
    ];
    
    return $response->withJson([
        'status' => 'success',
        'data' => $users
    ]);
}
```

### Created Resource Response

```php
public function store(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $data = $request->getParsedBody();
    
    // Process and save the data
    $newId = 123; // ID of newly created resource
    
    return $response
        ->withHeader('Content-Type', 'application/json')
        ->withHeader('Location', "/api/resources/{$newId}")
        ->withStatus(201)
        ->withBody(json_encode([
            'status' => 'success',
            'message' => 'Resource created',
            'data' => [
                'id' => $newId
            ]
        ]));
}
```

Using ODY's `Response` class:

```php
public function store(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $data = $request->getParsedBody();
    
    // Process and save the data
    $newId = 123; // ID of newly created resource
    
    return $response
        ->status(201)
        ->header('Location', "/api/resources/{$newId}")
        ->withJson([
            'status' => 'success',
            'message' => 'Resource created',
            'data' => [
                'id' => $newId
            ]
        ]);
}
```

### Error Response

```php
public function show(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    // Resource not found
    if (!$id || !$this->resourceExists($id)) {
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus(404)
            ->withBody(json_encode([
                'status' => 'error',
                'message' => 'Resource not found'
            ]));
    }
    
    // ...
}
```

Using ODY's `Response` class:

```php
public function show(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    // Resource not found
    if (!$id || !$this->resourceExists($id)) {
        return $response
            ->status(404)
            ->withJson([
                'status' => 'error',
                'message' => 'Resource not found'
            ]);
    }
    
    // ...
}
```

### Validation Error Response

```php
public function update(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    $data = $request->getParsedBody();
    
    // Validation errors
    $errors = [];
    if (empty($data['name'])) {
        $errors['name'] = 'Name is required';
    }
    if (empty($data['email'])) {
        $errors['email'] = 'Email is required';
    }
    
    if (!empty($errors)) {
        return $response
            ->withHeader('Content-Type', 'application/json')
            ->withStatus(422)
            ->withBody(json_encode([
                'status' => 'error',
                'message' => 'Validation failed',
                'errors' => $errors
            ]));
    }
    
    // ...
}
```

Using ODY's `Response` class:

```php
public function update(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    $data = $request->getParsedBody();
    
    // Validation errors
    $errors = [];
    if (empty($data['name'])) {
        $errors['name'] = 'Name is required';
    }
    if (empty($data['email'])) {
        $errors['email'] = 'Email is required';
    }
    
    if (!empty($errors)) {
        return $response
            ->status(422)
            ->withJson([
                'status' => 'error',
                'message' => 'Validation failed',
                'errors' => $errors
            ]);
    }
    
    // ...
}
```

### File Download Response

```php
public function download(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $filePath = '/path/to/file.pdf';
    $fileName = 'document.pdf';
    
    $fileStream = new \Ody\Foundation\Http\Stream(fopen($filePath, 'r'));
    
    return $response
        ->withHeader('Content-Type', 'application/pdf')
        ->withHeader('Content-Disposition', 'attachment; filename="' . $fileName . '"')
        ->withHeader('Content-Length', (string) filesize($filePath))
        ->withBody($fileStream);
}
```

### No Content Response

```php
public function delete(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    // Delete the resource
    // ...
    
    // Return 204 No Content
    return $response->withStatus(204);
}
```

Using ODY's `Response` class:

```php
public function delete(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    // Delete the resource
    // ...
    
    // Return 204 No Content
    return $response->status(204);
}
```

## Response Emitters

After your controller returns a response, it needs to be sent to the client. In standard PHP environments, the response is automatically emitted. However, in Swoole environments, you need to use a response emitter.

### Standard Response Emitter

```php
use Ody\Foundation\Http\ResponseEmitter;

$emitter = new ResponseEmitter();
$emitter->emit($response);
```

### Swoole Response Emitter

```php
use Ody\Foundation\Http\SwooleResponseEmitter;

$emitter = new SwooleResponseEmitter();
$emitter->emit($response);
```

The framework typically handles this for you, but it's useful to understand how it works.

## Creating a Helper Method for JSON Responses

To standardize your JSON responses, you can create a helper method in your controllers:

```php
/**
 * Helper method to create JSON responses
 *
 * @param ResponseInterface $response
 * @param mixed $data
 * @param int $status
 * @return ResponseInterface
 */
private function jsonResponse(ResponseInterface $response, $data, int $status = 200): ResponseInterface
{
    // Always set JSON content type
    $response = $response->withHeader('Content-Type', 'application/json');
    
    // Set status code
    $response = $response->withStatus($status);
    
    // If using our custom Response class
    if ($response instanceof \Ody\Foundation\Http\Response) {
        return $response->withJson($data);
    }
    
    // For other PSR-7 implementations
    $response->getBody()->write(json_encode($data));
    return $response;
}
```

Now you can use this method consistently in your controllers:

```php
public function show(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    $id = $params['id'] ?? null;
    
    if (!$id || !$this->resourceExists($id)) {
        return $this->jsonResponse($response, [
            'status' => 'error',
            'message' => 'Resource not found'
        ], 404);
    }
    
    $data = $this->fetchResource($id);
    
    return $this->jsonResponse($response, [
        'status' => 'success',
        'data' => $data
    ]);
}
```

## Streaming Responses

For large responses, you can stream the content to avoid memory issues:

```php
public function stream(ServerRequestInterface $request, ResponseInterface $response): ResponseInterface
{
    $response = $response
        ->withHeader('Content-Type', 'text/plain')
        ->withHeader('Transfer-Encoding', 'chunked');
    
    $body = $response->getBody();
    
    // Stream content in chunks
    for ($i = 0; $i < 10; $i++) {
        $body->write("Chunk " . $i . "\n");
        // In a real application, you might flush the output buffer here
    }
    
    return $response;
}
```

In Swoole environments, streaming works differently. The framework handles this for you when you return a response with 
a large body.


## Best Practices

1. **Immutability**: Remember that PSR-7 response objects are immutable. Methods like `withHeader()` return new instances rather than modifying the existing one.

2. **JSON Responses**: Use a consistent format for your JSON responses. Consider standardizing on a structure like `{ status, data, meta }` or `{ status, message, data, errors }`.

3. **Status Codes**: Use appropriate HTTP status codes for different scenarios. Don't just return 200 for everything.

4. **Error Handling**: Provide clear error messages and validation errors in your responses.

5. **Headers**: Set appropriate headers for content type, caching, and security.

6. **Resource URLs**: For created resources, include a `Location` header with the URL of the new resource.

7. **Response Size**: For large responses, consider streaming or pagination.

8. **Response Encoding**: Always specify the character encoding (typically UTF-8) in your content type headers.

9. **Consistency**: Create helper methods to ensure consistent response formats across your application.

