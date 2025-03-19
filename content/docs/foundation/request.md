---
title: Request
weight: 3
---

## Introduction

The ODY Framework provides an HTTP layer that follows PSR-7 standards for HTTP message interfaces and PSR-15 
standards for HTTP server request handlers. This document explains how to work with HTTP requests in your controllers 
and applications.

## Request Classes Overview

ODY's HTTP component includes several classes for handling requests:

- `Request`: The main PSR-7 compatible HTTP request class
- `ServerRequest`: Server-side HTTP request implementation
- `UploadedFile`: Handles file uploads according to PSR-7
- `Stream`: Represents request and response bodies as streams

## Basic Request Handling in Controllers

Controllers in ODY receive request objects that implement the `Psr\Http\Message\ServerRequestInterface`. Here's a basic 
example of a controller method:

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
    
    // Process the request
    $data = ['id' => (int)$id, 'name' => 'Example'];
    
    // Return a JSON response
    return $this->jsonResponse($response, $data);
}
```

## Accessing Request Data

### Route Parameters

Route parameters are passed to your controller methods in the `$params` array:

```php
public function show(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
{
    // Access a route parameter (e.g., /users/{id})
    $id = $params['id'] ?? null;
    
    // ...
}
```

### Query Parameters

Query parameters (from the URL's query string) can be accessed using the `getQueryParams()` method:

```php
// For a request to /users?sort=name&order=asc
$queryParams = $request->getQueryParams();
$sort = $queryParams['sort'] ?? 'id';
$order = $queryParams['order'] ?? 'asc';
```

### Form Data

For traditional form submissions (with `application/x-www-form-urlencoded` or `multipart/form-data` content types), use 
the `getParsedBody()` method:

```php
// For a POST/PUT request with form data
$formData = $request->getParsedBody();
$name = $formData['name'] ?? '';
$email = $formData['email'] ?? '';
```

### JSON Request Body

For JSON requests, you can get the raw content and decode it:

```php
// For a request with application/json content type
$jsonData = json_decode((string)$request->getBody(), true);
```

Alternatively, the ODY's `Request` class provides a more convenient `json()` method:

```php
// Using ODY's Request extension
$jsonData = $request->json();
$name = $jsonData['name'] ?? '';
```

### Input Helper Method

The ODY `Request` class provides a convenient `input()` method that checks for a parameter in both the request body and 
query parameters:

```php
// Will look in both query string and request body
$searchTerm = $request->input('search', 'default value');
```

### Headers

Access request headers:

```php
// Check if a header exists
if ($request->hasHeader('Content-Type')) {
    // Get a header (returns array of values)
    $contentType = $request->getHeader('Content-Type');
    
    // Get a header as a string
    $contentTypeString = $request->getHeaderLine('Content-Type');
}
```

### Cookies

Get cookies from the request:

```php
$cookies = $request->getCookieParams();
$sessionId = $cookies['session_id'] ?? null;
```

## File Uploads

For file uploads, use the `getUploadedFiles()` method which returns an array of `UploadedFileInterface` objects:

```php
$files = $request->getUploadedFiles();

// Access a file by its input name
if (isset($files['profile_image'])) {
    $file = $files['profile_image'];
    
    // Check if upload was successful
    if ($file->getError() === UPLOAD_ERR_OK) {
        // Get file details
        $clientFilename = $file->getClientFilename();
        $fileSize = $file->getSize();
        $mediaType = $file->getClientMediaType();
        
        // Move the uploaded file to a permanent location
        $file->moveTo('/path/to/uploads/' . $clientFilename);
        
        // Or get the file content as a stream
        $stream = $file->getStream();
        $content = (string)$stream;
    }
}
```

### Handling Multiple File Uploads

For multiple file inputs (with the same name), uploaded files will be in an array:

```php
// For <input type="file" name="documents[]" multiple>
$files = $request->getUploadedFiles();
if (isset($files['documents']) && is_array($files['documents'])) {
    foreach ($files['documents'] as $file) {
        if ($file->getError() === UPLOAD_ERR_OK) {
            // Process each file
            $file->moveTo('/path/to/uploads/' . $file->getClientFilename());
        }
    }
}
```

## Request Attributes

Request attributes are used to store arbitrary data associated with the request, often set by middleware:

```php
// Get an attribute
$user = $request->getAttribute('user');

// With a default value if the attribute doesn't exist
$locale = $request->getAttribute('locale', 'en');

// Create a new request with an added attribute
$requestWithUser = $request->withAttribute('user', $userObject);
```

## Request Methods and Properties

### Request Method

Get the HTTP method:

```php
$method = $request->getMethod();

// Check if it's a specific method
if ($request->getMethod() === 'POST') {
    // Handle POST request
}
```

### Request URI and Path

Get information about the request URI:

```php
// Get the full URI
$uri = $request->getUri();

// Get just the path
$path = $request->getUri()->getPath();

// Get the query string
$query = $request->getUri()->getQuery();

// Get the host
$host = $request->getUri()->getHost();
```

The ODY `Request` class also provides convenient shortcuts:

```php
// Get the path
$path = $request->getPath();

// Get the URI as a string
$uriString = $request->getUriString();
```

### Server Parameters

Access server parameters (equivalent to `$_SERVER`):

```php
$serverParams = $request->getServerParams();
$ipAddress = $serverParams['REMOTE_ADDR'] ?? '';
$userAgent = $serverParams['HTTP_USER_AGENT'] ?? '';
```

## Complete Controller Example

Here's a comprehensive example of a controller that handles various request scenarios:

```php
<?php

namespace App\Controllers;

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use Ody\Foundation\Http\Response;

class UserController
{
    /**
     * Display a listing of users
     */
    public function index(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
    {
        // Get query parameters for filtering and pagination
        $queryParams = $request->getQueryParams();
        $page = (int)($queryParams['page'] ?? 1);
        $limit = (int)($queryParams['limit'] ?? 20);
        $sort = $queryParams['sort'] ?? 'id';
        
        // Fetch users from database (in a real app)
        $users = [
            ['id' => 1, 'name' => 'John Doe'],
            ['id' => 2, 'name' => 'Jane Smith'],
        ];
        
        // Return JSON response
        return $this->jsonResponse($response, [
            'data' => $users,
            'meta' => [
                'page' => $page,
                'limit' => $limit,
                'total' => count($users),
            ]
        ]);
    }
    
    /**
     * Store a new user
     */
    public function store(ServerRequestInterface $request, ResponseInterface $response, array $params): ResponseInterface
    {
        // Get request data (works with both JSON and form data)
        $data = $request->getParsedBody() ?? $request->json() ?? [];
        
        // Validate input
        if (empty($data['name']) || empty($data['email'])) {
            return $this->jsonResponse($response->withStatus(422), [
                'error' => 'Validation failed',
                'messages' => [
                    'name' => empty($data['name']) ? 'Name is required' : null,
                    'email' => empty($data['email']) ? 'Email is required' : null,
                ]
            ]);
        }
        
        // Process file upload if present
        $profileImage = null;
        $files = $request->getUploadedFiles();
        if (isset($files['profile_image']) && $files['profile_image']->getError() === UPLOAD_ERR_OK) {
            $file = $files['profile_image'];
            $profileImage = uniqid() . '_' . $file->getClientFilename();
            $file->moveTo('/path/to/uploads/' . $profileImage);
        }
        
        // Create user in database (in a real app)
        // ...
        
        // Return success response
        return $this->jsonResponse($response->withStatus(201), [
            'id' => 3, // The new user ID
            'name' => $data['name'],
            'email' => $data['email'],
            'profile_image' => $profileImage,
        ]);
    }
    
    /**
     * Helper method to create JSON responses
     */
    private function jsonResponse(ResponseInterface $response, $data): ResponseInterface
    {
        // Set JSON content type
        $response = $response->withHeader('Content-Type', 'application/json');
        
        // For ODY Response class
        if ($response instanceof Response) {
            return $response->withJson($data);
        }
        
        // For other PSR-7 responses
        $response->getBody()->write(json_encode($data));
        return $response;
    }
}
```

## Best Practices

1. **Immutability**: Remember that PSR-7 request and response objects are immutable. Methods like `withHeader()` return new instances rather than modifying the existing one.
2. **Type Hinting**: Always use the PSR interfaces (`ServerRequestInterface`, `ResponseInterface`) for type hints in your controller methods, not the concrete classes.
3. **Validation**: Validate all input data before using it. Don't trust any data coming from the client.
4. **Response Format**: Be consistent with your response format. Consider standardizing on JSON for API responses.
5. **Error Handling**: Use appropriate HTTP status codes for errors and provide meaningful error messages.
6. **File Uploads**: Always check the error status of uploaded files before processing them.
7. **Resource Cleanup**: In Swoole environments, be mindful of cleaning up resources after request handling.
