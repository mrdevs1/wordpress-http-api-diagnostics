# WordPress HTTP API Diagnostics

A practical guide to diagnosing, debugging, and troubleshooting outgoing HTTP requests in WordPress using the WordPress HTTP API, PHP, and debugging tools.

## What Is the WordPress HTTP API?

The WordPress HTTP API provides a standardized way for plugins and themes to communicate with external services through HTTP requests.

Common use cases include:

- Connecting to third-party APIs
- Fetching external data
- Sending data to remote servers
- Integrating payment gateways
- Communicating with external applications
- Retrieving remote files
- Building plugin integrations

Common HTTP API functions:

| Function | Purpose |
|---|---|
| `wp_remote_get()` | Send GET requests |
| `wp_remote_post()` | Send POST requests |
| `wp_remote_request()` | Send custom HTTP requests |
| `wp_remote_head()` | Send HEAD requests |
| `wp_safe_remote_get()` | Perform a validated GET request |
| `wp_safe_remote_post()` | Perform a validated POST request |

## Common HTTP API Problems

- External API requests fail.
- Requests return `WP_Error`.
- SSL certificate verification fails.
- Remote connections time out.
- DNS resolution fails.
- External services return HTTP 403 or 500.
- Authentication headers are rejected.
- API responses contain unexpected data.
- Requests work locally but fail on production servers.

## 1. Test an External HTTP Request

Use `wp_remote_get()` to test an external endpoint.

```php
$response = wp_remote_get(
    'https://api.github.com/',
    array(
        'timeout' => 15,
    )
);

if ( is_wp_error( $response ) ) {
    error_log( $response->get_error_message() );
} else {
    error_log(
        'HTTP Status: ' .
        wp_remote_retrieve_response_code( $response )
    );
}
```

This helps determine whether WordPress can communicate with an external service.

Check the PHP error log for diagnostic information.

## 2. Understand WP_Error Responses

WordPress HTTP functions can return either a response array or a `WP_Error` object.

Always check for errors before accessing response information.

```php
$response = wp_remote_get( 'https://example.com/api' );

if ( is_wp_error( $response ) ) {

    $error_code = $response->get_error_code();
    $error_message = $response->get_error_message();

    error_log( $error_code . ': ' . $error_message );

    return;
}

$status_code = wp_remote_retrieve_response_code( $response );
$body = wp_remote_retrieve_body( $response );
```

Useful diagnostic methods:

```php
$response->get_error_codes();
$response->get_error_messages();
```

These methods help identify the underlying request failure.

## 3. Troubleshoot Connection Timeouts

A request may fail because the remote server takes too long to respond.

Inspect the timeout configuration:

```php
$response = wp_remote_get(
    'https://api.example.com/data',
    array(
        'timeout' => 30,
    )
);
```

Investigate:

- Remote server response time
- DNS resolution delays
- Firewall restrictions
- PHP execution limits
- Network connectivity
- External API availability

Increasing the timeout is not always the correct solution. Identify the underlying performance issue first.

## 4. Diagnose SSL Certificate Errors

SSL verification failures can prevent WordPress from establishing secure connections.

Inspect the server's CA certificate configuration and verify that the certificate bundle is current.

Test the external endpoint:

```bash
curl -Iv https://api.example.com
```

Check the PHP configuration:

```bash
php -i | grep -i "curl.cainfo"
```

```bash
php -i | grep -i "openssl.cafile"
```

Investigate:

- Expired SSL certificates
- Missing CA certificate bundles
- Incorrect server certificate configuration
- Outdated PHP or OpenSSL installations
- Incorrect server time

Never disable SSL verification as a permanent workaround.

## 5. Inspect HTTP Response Codes

A successful connection does not necessarily mean the external API request succeeded.

Inspect the HTTP response:

```php
$response = wp_remote_get( 'https://api.example.com/data' );

if ( is_wp_error( $response ) ) {
    return;
}

$status_code = wp_remote_retrieve_response_code( $response );

$body = wp_remote_retrieve_body( $response );

error_log( 'HTTP Status: ' . $status_code );
error_log( 'Response Body: ' . $body );
```

Common HTTP status codes:

| Status | Meaning | Investigation |
|---|---|---|
| 200 | OK | Successful request |
| 201 | Created | Resource created |
| 301 | Redirect | Inspect redirect destination |
| 400 | Bad Request | Validate request parameters |
| 401 | Unauthorized | Check authentication |
| 403 | Forbidden | Check API permissions |
| 404 | Not Found | Verify endpoint URL |
| 429 | Too Many Requests | Investigate rate limits |
| 500 | Server Error | Investigate remote server |
| 502 | Bad Gateway | Check upstream service |
| 503 | Service Unavailable | Check service availability |
| 504 | Gateway Timeout | Investigate response delays |

## 6. Debug POST Requests

Use `wp_remote_post()` when sending data to an external API.

```php
$response = wp_remote_post(
    'https://api.example.com/submit',
    array(
        'timeout' => 20,

        'headers' => array(
            'Content-Type' => 'application/json',
            'Accept' => 'application/json',
        ),

        'body' => wp_json_encode(
            array(
                'name' => 'WordPress',
                'status' => 'active',
            )
        ),
    )
);

if ( is_wp_error( $response ) ) {
    error_log( $response->get_error_message() );
    return;
}

error_log(
    wp_remote_retrieve_body( $response )
);
```

Verify that:

- The endpoint accepts POST requests.
- The content type matches the API requirements.
- Required parameters are included.
- Authentication credentials are valid.
- The remote service accepts the submitted payload.

## 7. Investigate HTTP API Authentication

External APIs may require authentication headers.

Example using a bearer token:

```php
$response = wp_remote_get(
    'https://api.example.com/private',
    array(
        'headers' => array(
            'Authorization' => 'Bearer YOUR_API_TOKEN',
            'Accept' => 'application/json',
        ),
    )
);
```

For production implementations:

- Store credentials outside publicly accessible source code.
- Use environment variables or secure configuration.
- Never log authentication tokens.
- Use HTTPS.
- Handle expired credentials appropriately.

## 8. Monitor HTTP API Requests

WordPress provides the `http_api_debug` action for monitoring HTTP API requests.

Example diagnostic logger:

```php
add_action(
    'http_api_debug',
    function ( $response, $context, $transport, $args, $url ) {

        if ( 'response' !== $context ) {
            return;
        }

        $status = is_wp_error( $response )
            ? $response->get_error_message()
            : wp_remote_retrieve_response_code( $response );

        error_log(
            sprintf(
                'HTTP API: %s | Transport: %s | Status: %s',
                $url,
                $transport,
                $status
            )
        );

    },
    10,
    5
);
```

This can help identify failing requests and the transport involved.

**Security note:** Avoid logging request headers, authentication credentials, private response bodies, or sensitive query parameters.

## 9. Inspect HTTP API Request Arguments

WordPress supports several arguments for controlling HTTP requests.

```php
$args = array(
    'method'      => 'GET',
    'timeout'     => 15,
    'redirection' => 5,
    'httpversion' => '1.1',
    'blocking'    => true,
    'headers'     => array(),
    'body'        => null,
    'sslverify'   => true,
);

$response = wp_remote_request(
    'https://api.example.com/data',
    $args
);
```

Important arguments:

| Argument | Purpose |
|---|---|
| `method` | HTTP request method |
| `timeout` | Maximum request duration |
| `redirection` | Maximum allowed redirects |
| `headers` | Request headers |
| `body` | Request payload |
| `sslverify` | SSL certificate verification |
| `blocking` | Whether to wait for the response |

Keep SSL verification enabled unless there is a carefully controlled testing reason to change it.

## 10. Investigate HTTP API Performance

External HTTP requests can increase WordPress page load times.

Consider these optimization techniques:

- Cache external API responses.
- Avoid repeated requests on every page load.
- Use appropriate request timeouts.
- Schedule background synchronization.
- Implement retry strategies with backoff.
- Respect external API rate limits.
- Avoid unnecessary blocking requests.

Example using WordPress Transients:

```php
$cached_data = get_transient( 'external_api_data' );

if ( false === $cached_data ) {

    $response = wp_remote_get(
        'https://api.example.com/data',
        array(
            'timeout' => 10,
        )
    );

    if ( is_wp_error( $response ) ) {
        return;
    }

    if ( 200 !== wp_remote_retrieve_response_code( $response ) ) {
        return;
    }

    $cached_data = wp_remote_retrieve_body( $response );

    set_transient(
        'external_api_data',
        $cached_data,
        HOUR_IN_SECONDS
    );
}
```

For production integrations, also validate the response structure before caching or consuming the data.

## 11. Troubleshoot HTTP Requests Using cURL

Test the same external endpoint outside WordPress.

```bash
curl -i https://api.example.com/data
```

Inspect connection details:

```bash
curl -v https://api.example.com/data
```

Test DNS resolution:

```bash
nslookup api.example.com
```

Test SSL certificates:

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com
```

If cURL works but WordPress fails, investigate PHP configuration, WordPress HTTP filters, request arguments, and server-level restrictions.

## 12. Essential Troubleshooting Checklist

- [ ] Verify the external API URL.
- [ ] Test connectivity outside WordPress.
- [ ] Check DNS resolution.
- [ ] Verify SSL certificates.
- [ ] Inspect `WP_Error` messages.
- [ ] Check HTTP response codes.
- [ ] Validate request headers.
- [ ] Verify authentication credentials.
- [ ] Inspect request timeout settings.
- [ ] Monitor HTTP API debug logs.
- [ ] Check firewall and hosting restrictions.
- [ ] Review caching and retry strategies.

## Important Security Guidelines

1. Prefer `wp_safe_remote_get()` or `wp_safe_remote_request()` when working with user-controlled URLs.
2. Never disable SSL verification permanently.
3. Validate and sanitize external data before using it.
4. Protect API credentials and authentication tokens.
5. Avoid exposing private response data in logs.
6. Respect API rate limits.
7. Use appropriate timeouts.
8. Cache external responses when appropriate.

## Conclusion

The WordPress HTTP API is essential for plugin integrations, external service communication, and modern WordPress development.

Understanding request handling, `WP_Error`, HTTP status codes, SSL verification, authentication, and debugging hooks makes diagnosing external connection problems much easier.

This repository serves as a practical technical reference for WordPress developers troubleshooting HTTP API integrations.
