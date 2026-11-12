# NGINX Configuration Mastery
## A Practical Reference Guide to Reading, Understanding, and Troubleshooting NGINX Configuration Files

**Purpose:** Develop the ability to read unfamiliar NGINX `.conf` files, understand their syntax and relationships, interpret directives, and confidently troubleshoot configuration issues.

---

# Table of Contents

1. [Introduction](#1-introduction)
2. [Understanding NGINX Configuration Syntax](#2-understanding-nginx-configuration-syntax)
3. [Understanding NGINX Contexts](#3-understanding-nginx-contexts)
4. [Understanding the NGINX Directory Structure](#4-understanding-the-nginx-directory-structure)
5. [Understanding Include Directives](#5-understanding-include-directives)
6. [Understanding Symbolic Links](#6-understanding-symbolic-links)
7. [Understanding Directive Inheritance](#7-understanding-directive-inheritance)
8. [Understanding Directive Arguments](#8-understanding-directive-arguments)
9. [Understanding NGINX Variables](#9-understanding-nginx-variables)
10. [Understanding Regular Expressions](#10-understanding-regular-expressions)
11. [Understanding Location Matching](#11-understanding-location-matching)
12. [Understanding Upstream Blocks](#12-understanding-upstream-blocks)
13. [Understanding Map Blocks](#13-understanding-map-blocks)
14. [Understanding Proxy Directives](#14-understanding-proxy-directives)
15. [Understanding Rewrite and Try Files](#15-understanding-rewrite-and-try-files)
16. [Understanding SSL Configuration](#16-understanding-ssl-configuration)
17. [Understanding Logging Configuration](#17-understanding-logging-configuration)
18. [Understanding Common NGINX Directives](#18-understanding-common-nginx-directives)
19. [How to Read an Unfamiliar NGINX Configuration](#19-how-to-read-an-unfamiliar-nginx-configuration)
20. [NGINX Inspection and Troubleshooting Commands](#20-nginx-inspection-and-troubleshooting-commands)
21. [Common NGINX Errors](#21-common-nginx-errors)
22. [Safe Configuration Change Workflow](#22-safe-configuration-change-workflow)
23. [Practical Exercises](#23-practical-exercises)
24. [Official Documentation](#24-official-documentation)

---

# 1. Introduction

NGINX configurations are written using a declarative configuration language.

Unlike programming languages such as Go, Python, or JavaScript, NGINX configuration does not primarily describe a sequence of instructions to execute.

Instead, it describes how NGINX should behave when processing connections and requests.

An NGINX configuration consists of:

- Contexts (configuration blocks).
- Directives (configuration instructions).
- Directive arguments.
- Variables.
- Regular expressions.
- Nested configuration structures.
- Included configuration files.

Understanding these elements makes it considerably easier to navigate large production configurations.

The objective of this guide is not simply to memorize directives but to understand the grammar and structure that allow you to interpret unfamiliar configurations.

# 2. Understanding NGINX Configuration Syntax

There are two fundamental patterns in NGINX configuration syntax.

## 2.1 Simple directives

A simple directive follows this structure:

```nginx
directive argument1 argument2;
```

Examples:

```nginx
worker_processes auto;

client_max_body_size 50M;

proxy_read_timeout 60s;

server_name example.com;
```

A simple directive:

- Begins with a directive name.
- Can contain one or more arguments.
- Ends with a semicolon (`;`).

The semicolon is mandatory for ordinary directives.

For example:

```nginx
worker_processes auto
```

Is invalid because it lacks a terminating semicolon.

## 2.2 Block directives

A block directive follows this structure:

```nginx
block_directive arguments {
    configuration;
    configuration;
}
```

Examples:

```nginx
http {
    ...
}

server {
    ...
}

location /api/ {
    ...
}

upstream backend {
    ...
}
```

A block directive opens a configuration context using `{` and closes it using `}`.

A block can contain:

- Simple directives.
- Other block directives.
- Nested configuration structures.

## 2.3 Understanding braces

Consider:

```nginx
http {
    server {
        listen 80;

        location /api/ {
            proxy_pass http://localhost:8080;
        }
    }
}
```

The braces establish the hierarchy.

```text
http
 └── server
      └── location
           └── proxy_pass
```

When reading a complex configuration, always identify where a block begins and ends.

A misplaced closing brace can change the context of subsequent directives or cause a syntax error.

## 2.4 Comments

Comments begin with `#`.

```nginx
# Global configuration

worker_processes auto;

http {
    # Website configuration

    server {
        listen 80; # HTTP listener
    }
}
```

Comments are ignored by the NGINX configuration parser.

They are useful for documenting configuration decisions and temporarily disabling directives.

## 2.5 Quoted arguments

NGINX supports single and double quotes.

```nginx
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent';
```

Quoted strings allow spaces and special characters to be included in an argument.

The example above defines a log format containing multiple NGINX variables.

---

# 3. Understanding NGINX Contexts

Contexts are the foundation of NGINX configuration organization.

A context determines where directives can be placed and what scope they affect.

The most important contexts are:

| Context | Purpose |
|---|---|
| Main | Global NGINX configuration |
| events | Connection processing |
| http | HTTP configuration |
| server | Virtual server configuration |
| location | URI-specific configuration |
| upstream | Backend server groups |
| map | Variable mapping |
| if | Conditional configuration |

## 3.1 Main context

The main context contains global configuration directives.

Example:

```nginx
user www-data;

worker_processes auto;

error_log /var/log/nginx/error.log;

pid /run/nginx.pid;
```

Common directives:

| Directive | Purpose |
|---|---|
| user | Defines the user and optionally group used by worker processes |
| worker_processes | Number of worker processes |
| error_log | Defines the error-log destination and optional severity |
| pid | Defines the PID file location |
| worker_rlimit_nofile | Changes the maximum open file descriptor limit for worker processes |

These directives are generally placed outside other blocks.

## 3.2 Events context

The events context controls connection-processing behaviour.

Example:

```nginx
events {
    worker_connections 4096;
    multi_accept on;
}
```

Important directives:

**worker_connections**

Sets the maximum number of simultaneous connections a worker process can open.

For example:

```nginx
worker_processes 4;

events {
    worker_connections 4096;
}
```

A theoretical connection limit is:

```text
4 workers × 4096 connections = 16384 connections
```

This is not a guaranteed number of simultaneous client requests.

Reverse-proxy connections, file descriptor limits, operating-system constraints and available resources must also be considered.

**multi_accept**

Controls whether a worker accepts all available new connections at once or accepts one connection at a time.

## 3.3 HTTP context

The HTTP context contains configuration related to HTTP traffic.

Example:

```nginx
http {
    include /etc/nginx/mime.types;

    default_type application/octet-stream;

    sendfile on;

    keepalive_timeout 65;

    server {
        listen 80;
        server_name example.com;
    }
}
```

Directives defined here can apply to multiple virtual servers.

Common HTTP-level directives include:

- include
- log_format
- access_log
- client_max_body_size
- sendfile
- keepalive_timeout
- gzip
- upstream
- map

## 3.4 Server context

The server context defines a virtual server.

Example:

```nginx
server {
    listen 80;

    server_name api.example.com;

    root /var/www/api;
}
```

A server block can contain:

- listen
- server_name
- root
- index
- access_log
- error_log
- location
- SSL directives
- proxy-related directives

Multiple server blocks can listen on the same port and serve different hostnames.

NGINX uses the requested hostname and listening address/port to select the appropriate server configuration.

## 3.5 Location context

The location context defines how requests matching particular URI patterns should be handled.

Example:

```nginx
location /api/ {
    proxy_pass http://localhost:8080;
}
```

A server can contain multiple location blocks.

Different locations can:

- Serve static files.
- Proxy requests to applications.
- Return custom responses.
- Restrict access.
- Apply rate limits.
- Configure caching.

## 3.6 Upstream context

The upstream context defines a named group of backend servers.

Example:

```nginx
upstream backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
```

It is normally defined inside the HTTP context.

## 3.7 Map context

The map context defines a variable based on another variable.

Example:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

It is also defined inside the HTTP context.

## 3.8 Context hierarchy

A simplified hierarchy:

```text
Main
│
├── events
│
└── http
    │
    ├── upstream
    │
    ├── map
    │
    └── server
        │
        ├── location
        │
        └── location
```

Not every context can be nested arbitrarily.

Always check the documentation for the permitted contexts of an unfamiliar directive.

---

# 4. Understanding the NGINX Directory Structure

On Debian and Ubuntu installations, NGINX commonly uses:

```text
/etc/nginx/
│
├── nginx.conf
│
├── mime.types
│
├── conf.d/
│   ├── application.conf
│   └── security.conf
│
├── sites-available/
│   ├── api.conf
│   ├── admin.conf
│   └── website.conf
│
├── sites-enabled/
│   ├── api.conf
│   └── website.conf
│
├── snippets/
│   ├── ssl-params.conf
│   └── fastcgi-php.conf
│
└── modules-enabled/
```

These directories serve different organizational purposes.

| Directory/file | Purpose |
|---|---|
| nginx.conf | Main NGINX configuration |
| mime.types | Maps file extensions to MIME types |
| conf.d | Commonly used for additional configuration files |
| sites-available | Stores available virtual-server configurations |
| sites-enabled | Contains configurations intended to be loaded |
| snippets | Stores reusable configuration fragments |
| modules-enabled | Commonly contains enabled module configuration files |

Important:

These directory names are not special NGINX language constructs.

Their behaviour depends on the `include` directives in the main configuration and any other included files.

---

# 5. Understanding Include Directives

The `include` directive allows NGINX to load configuration from another file.

Syntax:

```nginx
include file;
```

Example:

```nginx
http {
    include /etc/nginx/mime.types;

    include /etc/nginx/conf.d/*.conf;

    include /etc/nginx/sites-enabled/*;
}
```

## 5.1 Include as file composition

Suppose `nginx.conf` contains:

```nginx
http {
    include /etc/nginx/conf.d/api.conf;
}
```

And `api.conf` contains:

```nginx
server {
    listen 80;
    server_name api.example.com;
}
```

Conceptually, NGINX processes the configuration as if the contents of `api.conf` were inserted at the location of the include directive.

The resulting structure is:

```nginx
http {
    server {
        listen 80;
        server_name api.example.com;
    }
}
```

The included file does not automatically create a new context.

It is parsed in the context where it is included.

Therefore:

```nginx
http {
    include /etc/nginx/conf.d/*.conf;
}
```

Normally expects the included files to contain valid HTTP-context configuration.

A `server` block is valid there.

Another `http` block would not be valid.

## 5.2 Wildcards

NGINX supports wildcard patterns in include paths.

```nginx
include /etc/nginx/conf.d/*.conf;
```

Loads files matching the `.conf` pattern.

```nginx
include /etc/nginx/sites-enabled/*;
```

Loads matching entries regardless of their filename extension.

A file that exists somewhere under `/etc/nginx` is not automatically loaded.

It must be included directly or indirectly by the active configuration.

## 5.3 Nested includes

An included file can include another file.

Example:

`nginx.conf`:

```nginx
http {
    include /etc/nginx/conf.d/*.conf;
}
```

`conf.d/api.conf`:

```nginx
server {
    listen 443 ssl;

    include /etc/nginx/snippets/ssl-params.conf;
}
```

`snippets/ssl-params.conf`:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_session_cache shared:SSL:10m;
```

This creates a chain of configuration dependencies.

When investigating unfamiliar NGINX installations, follow the include chain rather than assuming each file operates independently.

---

# 6. Understanding Symbolic Links

A symbolic link (symlink) is a filesystem reference that points to another file or directory.

NGINX installations commonly use symlinks to connect `sites-available` and `sites-enabled`.

Example:

```text
/etc/nginx/
│
├── sites-available/
│   ├── api.conf
│   ├── admin.conf
│   └── website.conf
│
└── sites-enabled/
    ├── api.conf -> ../sites-available/api.conf
    └── website.conf -> ../sites-available/website.conf
```

Here:

- `sites-available` contains the actual configuration files.
- `sites-enabled` contains symbolic links to selected configurations.

## 6.1 Creating a symbolic link

```bash
sudo ln -s /etc/nginx/sites-available/api.conf \
           /etc/nginx/sites-enabled/api.conf
```

Syntax:

```bash
ln -s TARGET LINK_NAME
```

| Argument | Meaning |
|---|---|
| ln | Create a link |
| -s | Create a symbolic link |
| TARGET | Original file |
| LINK_NAME | Path and name of the symlink |

The command does not create another independent copy of the configuration.

## 6.2 How NGINX loads the symlink

Suppose the main configuration contains:

```nginx
http {
    include /etc/nginx/sites-enabled/*;
}
```

NGINX encounters:

```text
/etc/nginx/sites-enabled/api.conf
```

The operating system resolves the symlink and provides the contents of the original file.

The configuration is therefore loaded.

## 6.3 Disabling a configuration

Remove the symlink:

```bash
sudo rm /etc/nginx/sites-enabled/api.conf
```

This does not delete:

```text
/etc/nginx/sites-available/api.conf
```

The original configuration remains available.

## 6.4 Useful symlink commands

Inspect symlinks:

```bash
ls -l /etc/nginx/sites-enabled/
```

Resolve a symlink:

```bash
readlink -f /etc/nginx/sites-enabled/api.conf
```

Inspect file type:

```bash
file /etc/nginx/sites-enabled/api.conf
```

Find symlinks:

```bash
find /etc/nginx/sites-enabled -maxdepth 1 -type l -ls
```

Important distinction:

`sites-available` and `sites-enabled` are administrative conventions, not mandatory NGINX directories.

The actual activation mechanism is the combination of symbolic links and include directives.

---

# 7. Understanding Directive Inheritance

Inheritance determines whether configuration defined in a parent context applies to a child context.

Example:

```nginx
http {
    client_max_body_size 20M;

    server {
        server_name api.example.com;

        client_max_body_size 50M;

        location /uploads/ {
            # Effective value: 50M
        }
    }
}
```

The server-level value overrides the HTTP-level value.

However, inheritance rules differ between directives.

For example, `proxy_set_header` has its own inheritance behaviour: parent-level proxy headers are inherited only when no `proxy_set_header` directives are defined at the current level.

This is important when a location block defines only one proxy header and you expect all parent-level headers to remain active.

Always check the directive's documentation rather than assuming universal inheritance.

---

# 8. Understanding Directive Arguments

NGINX directives can accept different types of arguments.

## 8.1 Literal strings

```nginx
server_name api.example.com;
```

The argument is a hostname.

## 8.2 Numeric arguments

```nginx
worker_connections 1024;
```

The argument is a number.

## 8.3 Size arguments

```nginx
client_max_body_size 50M;
```

Common size suffixes:

| Suffix | Meaning |
|---|---|
| k / K | Kilobytes |
| m / M | Megabytes |
| g / G | Gigabytes |

## 8.4 Time arguments

```nginx
proxy_read_timeout 60s;

keepalive_timeout 65;

client_body_timeout 30s;
```

Common time units include:

| Suffix | Meaning |
|---|---|
| ms | Milliseconds |
| s | Seconds |
| m | Minutes |
| h | Hours |
| d | Days |

Some directives also accept compound time values.

Always consult the specific directive documentation.

## 8.5 Quoted arguments

```nginx
return 200 "Application is healthy";
```

Quotes allow spaces within an argument.

## 8.6 Multiple arguments

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
```

This contains three tokens:

```text
proxy_set_header
X-Forwarded-Proto
$scheme
```

The directive name is followed by two arguments.

---

# 9. Understanding NGINX Variables

NGINX variables begin with `$`.

Examples:

```nginx
$remote_addr
$host
$request_uri
$request_method
$status
$scheme
$server_name
$http_user_agent
$http_upgrade
$proxy_add_x_forwarded_for
```

Variables allow configuration directives to use request-specific and connection-specific information.

## 9.1 Common variables

| Variable | Meaning |
|---|---|
| `$remote_addr` | Client address as seen by NGINX |
| `$host` | Hostname from the request or applicable server configuration |
| `$request_uri` | Original request URI, including arguments |
| `$uri` | Current normalized URI |
| `$request_method` | HTTP method |
| `$scheme` | Request scheme, such as HTTP or HTTPS |
| `$server_name` | Name of the server handling the request |
| `$status` | Response status code |
| `$http_user_agent` | User-Agent request header |
| `$http_upgrade` | Upgrade request header |
| `$args` | Query-string arguments |
| `$request_time` | Total time spent processing the request |
| `$body_bytes_sent` | Response body bytes sent to the client |

## 9.2 Example

```nginx
proxy_set_header Host $host;
```

Interpretation:

Set the upstream Host header to the value of NGINX's `$host` variable.

Another example:

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

This passes the client address as seen by NGINX to the upstream application.

## 9.3 Variables inside strings

```nginx
return 302 https://$host$request_uri;
```

NGINX expands the variables when evaluating the directive.

A request for:

```text
http://example.com/users?page=2
```

Can be redirected to:

```text
https://example.com/users?page=2
```

Not every directive supports variables in every argument position.

---

# 10. Understanding Regular Expressions

Regular expressions are used extensively in NGINX location matching and rewrite rules.

Example:

```nginx
location ~ \.php$ {
    ...
}
```

Breakdown:

| Character | Meaning |
|---|---|
| `\.` | Literal dot |
| `php` | Characters php |
| `$` | End of string |

This matches URIs ending in `.php`.

## 10.1 Common regex characters

| Symbol | Meaning |
|---|---|
| `.` | Any character |
| `\.` | Literal dot |
| `^` | Beginning of string |
| `$` | End of string |
| `*` | Zero or more repetitions |
| `+` | One or more repetitions |
| `?` | Zero or one repetition |
| `( )` | Capture/group |
| `|` | Alternative |
| `[ ]` | Character class |
| `.*` | Any sequence of characters |

## 10.2 Example: PHP files

```nginx
location ~ \.php$ {
    ...
}
```

Matches:

```text
index.php
admin.php
users.php
```

Does not match:

```text
index.html
index.php/test
```

## 10.3 Example: Image extensions

```nginx
location ~* \.(jpg|jpeg|png|gif)$ {
    ...
}
```

Matches:

```text
image.jpg
image.PNG
photo.jpeg
logo.gif
```

The `~*` modifier makes the regular expression case-insensitive.

## 10.4 Capture groups

```nginx
rewrite ^/old/(.*)$ /new/$1 permanent;
```

Here:

- `(.*)` captures the remaining URI characters.
- `$1` references the first capture group.

For example:

```text
/old/users
```

Becomes:

```text
/new/users
```

---

# 11. Understanding Location Matching

Location matching determines which location block handles a request.

There are five important forms.

| Syntax | Meaning |
|---|---|
| `location /api/` | Prefix matching |
| `location = /health` | Exact matching |
| `location ^~ /static/` | Prefix match that prevents regex evaluation when selected |
| `location ~ \.php$` | Case-sensitive regex |
| `location ~* \.(jpg\|png)$` | Case-insensitive regex |

## 11.1 Prefix matching

```nginx
location /api/ {
    ...
}
```

Matches URIs beginning with `/api/`.

Examples:

```text
/api/users
/api/payments
/api/transactions
/api/v1/users
```

## 11.2 Exact matching

```nginx
location = /health {
    return 200 "OK";
}
```

Only matches the exact URI `/health`.

## 11.3 The ^~ modifier

```nginx
location ^~ /static/ {
    ...
}
```

If this is the longest matching prefix, NGINX does not subsequently evaluate regex locations.

This is useful when static-file routing should take precedence over general regex rules.

## 11.4 Location selection algorithm

The simplified selection process:

1. Check for an exact match using `=`.
2. Find the longest matching prefix.
3. If the selected prefix uses `^~`, use it without checking regex locations.
4. Otherwise, evaluate regex locations in configuration order.
5. The first matching regex wins.
6. If no regex matches, use the longest matching prefix.

Nested locations and certain internal redirects introduce additional nuances.

---

# 12. Understanding Upstream Blocks

An upstream block defines a named group of backend servers.

Example:

```nginx
upstream application {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}
```

The group can be referenced elsewhere:

```nginx
location /api/ {
    proxy_pass http://application;
}
```

## 12.1 Common load-balancing parameters

### Default round robin

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### Least connections

```nginx
upstream backend {
    least_conn;

    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### Backup server

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080 backup;
}
```

The backup server is used when the primary servers are unavailable according to NGINX's upstream availability handling.

### Weight

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=1;
}
```

The first server receives a larger proportion of traffic under weighted round-robin distribution.

### Maximum failures and timeout

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
}
```

These parameters influence how NGINX temporarily considers a server unavailable.

Passive failure detection has limitations. It is not equivalent to a dedicated application health-check system.

---

# 13. Understanding Map Blocks

The `map` directive creates a variable based on another variable.

Example:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

This can be understood as a lookup table.

| Input | Output |
|---|---|
| Non-empty upgrade header | upgrade |
| Empty upgrade header | close |

The resulting variable can be used elsewhere:

```nginx
proxy_set_header Upgrade $http_upgrade;

proxy_set_header Connection $connection_upgrade;
```

Maps are useful for:

- WebSocket configuration.
- Conditional header values.
- Request classification.
- Creating variables from request attributes.

Map blocks are normally defined in the HTTP context.

---

# 14. Understanding Proxy Directives

These directives are especially important when investigating NGINX reverse-proxy configurations.

## 14.1 proxy_pass

Example:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

Forwards requests to the application listening on port 8080.

### Important: trailing slash behaviour

Consider:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```

A request for:

```text
/api/users
```

is forwarded as:

```text
/users
```

The URI component `/` in `proxy_pass` replaces the matching location prefix.

Compare:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8080;
}
```

Here, the original URI is generally forwarded as:

```text
/api/users
```

This distinction matters when troubleshooting unexpected backend 404 responses.

## 14.2 proxy_set_header

Syntax:

```nginx
proxy_set_header HEADER_NAME HEADER_VALUE;
```

Example:

```nginx
proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;
```

These headers communicate information about the original client request to the upstream application.

Applications often use them to understand:

- Original hostname.
- Client IP information.
- Original request protocol.
- Proxy chain.

The upstream application should only trust forwarded headers from trusted proxies.

## 14.3 proxy_connect_timeout

```nginx
proxy_connect_timeout 10s;
```

Maximum time allowed for establishing a connection to an upstream server.

## 14.4 proxy_send_timeout

```nginx
proxy_send_timeout 60s;
```

Timeout between successive write operations while sending a request to the upstream.

## 14.5 proxy_read_timeout

```nginx
proxy_read_timeout 60s;
```

Timeout between successive read operations from the upstream.

Important: this is not necessarily a total request execution deadline.

## 14.6 Example configuration

```nginx
location /api/ {
    proxy_pass http://application;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 10s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```

---

# 15. Understanding Rewrite and Try Files

## 15.1 Rewrite

Example:

```nginx
rewrite ^/old/(.*)$ /new/$1 permanent;
```

This rewrites the URI and returns a permanent redirect.

Common flags:

| Flag | Meaning |
|---|---|
| last | Restart location searching using the rewritten URI |
| break | Stop rewrite processing in the current location |
| redirect | Return a temporary redirect |
| permanent | Return a permanent redirect |

## 15.2 Try files

Example:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

NGINX checks:

1. Whether the requested file exists.
2. Whether the corresponding directory exists.
3. If neither exists, internally redirects to `/index.html`.

This is commonly used with single-page applications.

## 15.3 Root versus alias

These directives are easy to confuse.

Using `root`:

```nginx
location /images/ {
    root /var/www;
}
```

A request for:

```text
/images/logo.png
```

maps to:

```text
/var/www/images/logo.png
```

Using `alias`:

```nginx
location /images/ {
    alias /var/www/assets/;
}
```

The same request maps to:

```text
/var/www/assets/logo.png
```

The key difference:

- `root` appends the URI to the configured root directory.
- `alias` replaces the matching location prefix with the alias path.

Trailing slashes and regex locations require particular care when using `alias`.

---

# 16. Understanding SSL Configuration

An HTTPS server commonly looks like this:

```nginx
server {
    listen 443 ssl;

    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.crt;

    ssl_certificate_key /etc/ssl/private/example.key;

    ssl_protocols TLSv1.2 TLSv1.3;
}
```

Important directives:

| Directive | Purpose |
|---|---|
| listen | Listening address and port |
| ssl_certificate | Certificate file |
| ssl_certificate_key | Private key file |
| ssl_protocols | Enabled TLS protocol versions |
| ssl_ciphers | Allowed cipher suites, where applicable |
| ssl_session_cache | TLS session cache configuration |
| ssl_session_timeout | TLS session lifetime |

A common HTTP-to-HTTPS redirect:

```nginx
server {
    listen 80;

    server_name example.com;

    return 301 https://$host$request_uri;
}
```

This redirects HTTP requests to HTTPS.

Never expose SSL private keys, passwords or certificate-management credentials when sharing configuration files.

---

# 17. Understanding Logging Configuration

NGINX has two major logging categories.

## 17.1 Access logs

Record information about HTTP requests.

Example:

```nginx
access_log /var/log/nginx/access.log;
```

## 17.2 Error logs

Record operational and configuration errors.

Example:

```nginx
error_log /var/log/nginx/error.log warn;
```

## 17.3 Log formats

A custom log format can be defined:

```nginx
http {
    log_format main
        '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $body_bytes_sent '
        '"$http_referer" "$http_user_agent"';

    access_log /var/log/nginx/access.log main;
}
```

The name `main` references the defined log format.

Common error-log levels include:

- debug
- info
- notice
- warn
- error
- crit
- alert
- emerg

A lower severity threshold generally includes more messages.

---

# 18. Understanding Common NGINX Directives

| Directive | Purpose |
|---|---|
| listen | Defines listening address and port |
| server_name | Defines hostnames handled by a server |
| root | Defines the document root |
| alias | Maps a location to a filesystem path |
| index | Defines default index files |
| include | Loads another configuration file |
| return | Returns an HTTP response or redirect |
| rewrite | Rewrites request URIs |
| try_files | Tests filesystem paths |
| proxy_pass | Forwards requests to an upstream |
| proxy_set_header | Sets headers sent to an upstream |
| proxy_connect_timeout | Upstream connection timeout |
| proxy_send_timeout | Upstream write timeout |
| proxy_read_timeout | Upstream read timeout |
| client_max_body_size | Maximum accepted request-body size |
| client_body_timeout | Timeout between successive reads of a client request body |
| sendfile | Enables sendfile-based file transmission |
| gzip | Enables response compression |
| access_log | Configures access logging |
| error_log | Configures error logging |
| limit_req | Applies request-rate limiting |
| limit_conn | Limits simultaneous connections |
| expires | Controls expiration headers |
| add_header | Adds response headers |
| deny | Denies access |
| allow | Allows access |
| return | Immediately returns a response |
| error_page | Configures error-page handling |

---

# 19. How to Read an Unfamiliar NGINX Configuration

When you open a configuration file containing hundreds of lines, do not immediately try to understand every directive.

Use a structured approach.

## Step 1: Identify the configuration hierarchy

Find:

```nginx
http {
    ...
}
```

Then identify:

- Server blocks.
- Upstream blocks.
- Map blocks.
- Location blocks.
- Include directives.

## Step 2: Identify included files

Search for:

```nginx
include
```

Follow each include path to understand where the rest of the configuration comes from.

## Step 3: Identify virtual servers

Search for:

```nginx
server {
```

For each server, identify:

- listen
- server_name
- SSL settings
- access_log
- error_log

## Step 4: Identify request routing

Inspect each location:

```nginx
location /api/ {
    proxy_pass http://backend;
}
```

Determine which application receives the request.

## Step 5: Identify referenced variables

Whenever you see `$variable`, identify what it represents and where it is defined.

Pay particular attention to custom variables created using `map` or `set`.

## Step 6: Identify dependencies

Look for:

- Named upstream groups.
- Custom log formats.
- Included snippets.
- Certificates.
- Filesystem paths.
- Rate-limiting zones.
- Cache zones.

## Step 7: Understand inheritance

Determine whether directives are defined at HTTP, server or location level and whether their values are inherited.

## Step 8: Trace an example request

For example:

```text
https://api.example.com/api/users
```

Ask:

1. Which listening socket receives it?
2. Which server block matches?
3. Which location matches?
4. Which directives apply?
5. Is the request proxied?
6. What upstream receives it?
7. What URI reaches the upstream?
8. What headers are forwarded?
9. Which logs record the request?

This process turns an unfamiliar configuration into a traceable request-processing flow.

---

# 20. NGINX Inspection and Troubleshooting Commands

These commands are particularly useful when working on production Linux servers.

## Version and build information

```bash
nginx -v
```

Displays the installed NGINX version.

```bash
nginx -V
```

Displays version and build configuration, including compiled modules and build options.

## Configuration inspection

```bash
sudo nginx -t
```

Tests configuration syntax and attempts to open referenced files.

```bash
sudo nginx -T
```

Tests and dumps the complete active configuration, including included files.

This is one of the most useful commands when investigating an unfamiliar NGINX installation.

## Service inspection

```bash
sudo systemctl status nginx
```

Displays service status.

```bash
sudo systemctl reload nginx
```

Requests a configuration reload through systemd.

```bash
sudo systemctl restart nginx
```

Restarts the service.

## Logs

```bash
sudo tail -f /var/log/nginx/access.log
```

Follows access logs.

```bash
sudo tail -f /var/log/nginx/error.log
```

Follows error logs.

```bash
sudo journalctl -u nginx -n 100
```

Displays the latest 100 service journal entries.

```bash
sudo journalctl -u nginx -f
```

Follows the service journal.

## Listening ports

```bash
sudo ss -tulpn | grep nginx
```

Displays listening sockets associated with NGINX.

## Process inspection

```bash
ps aux | grep nginx
```

Displays NGINX processes.

## Filesystem inspection

```bash
ls -lah /etc/nginx/
```

Lists NGINX configuration files.

```bash
find /etc/nginx -type f
```

Finds regular files under the NGINX directory.

```bash
find /etc/nginx -type l
```

Finds symbolic links.

---

# 21. Common NGINX Errors

## 502 Bad Gateway

Usually indicates that NGINX could not obtain a valid response from an upstream server.

Possible causes:

- Application is not running.
- Wrong upstream IP or port.
- Connection refused.
- Upstream application crashed.
- Invalid response from upstream.
- DNS resolution problem.

Investigate:

```bash
sudo tail -f /var/log/nginx/error.log
```

Check application connectivity:

```bash
curl -v http://127.0.0.1:8080/
```

## 504 Gateway Timeout

Usually indicates that NGINX timed out while waiting for an upstream response.

Possible causes:

- Slow application processing.
- Database delays.
- Upstream overload.
- Network latency.
- Timeout configuration too restrictive.

Investigate:

```bash
sudo tail -f /var/log/nginx/error.log
```

Check:

```nginx
proxy_connect_timeout;
proxy_send_timeout;
proxy_read_timeout;
```

These directives control different phases of upstream communication.

## 403 Forbidden

Possible causes:

- Filesystem permission problems.
- NGINX worker cannot access a file.
- Explicit access restrictions.
- Missing directory index.
- Incorrect `deny` or access-control rules.

Investigate:

```bash
sudo tail -f /var/log/nginx/error.log
```

Check:

```bash
ls -lah /var/www/
```

## 404 Not Found

Possible causes:

- Incorrect URI routing.
- Incorrect root or alias.
- Missing static file.
- Incorrect `try_files` configuration.
- Backend application returning 404.

Determine whether the 404 originates from NGINX or the upstream application.

## Configuration test failures

Example:

```text
nginx: [emerg] unexpected "}" in /etc/nginx/nginx.conf
```

Possible causes:

- Incorrect brace placement.
- Missing semicolon.
- Invalid directive.
- Directive placed in an unsupported context.

Use:

```bash
sudo nginx -t
```

The output usually identifies the relevant file and line number.

---

# 22. Safe Configuration Change Workflow

Production NGINX changes should follow a deliberate workflow.

## Step 1: Inspect the active configuration

```bash
sudo nginx -T
```

Understand which files are currently loaded.

## Step 2: Identify the target file

Do not assume that a configuration file is active merely because it exists.

Check the include chain and enabled symlinks.

## Step 3: Back up the file

```bash
sudo cp /etc/nginx/sites-available/api.conf \
        /etc/nginx/sites-available/api.conf.bak
```

## Step 4: Make the change

```bash
sudo vim /etc/nginx/sites-available/api.conf
```

Only change what you understand and have permission to change.

## Step 5: Validate

```bash
sudo nginx -t
```

Do not proceed if validation fails.

## Step 6: Reload

```bash
sudo systemctl reload nginx
```

A reload normally allows NGINX to start new workers with the new configuration while old workers gracefully finish their existing connections.

## Step 7: Verify

```bash
sudo systemctl status nginx
```

Test the application:

```bash
curl -v https://api.example.com/health
```

Inspect the logs:

```bash
sudo tail -f /var/log/nginx/error.log
```

## Step 8: Roll back if necessary

Restore the backup and validate again.

```bash
sudo cp /etc/nginx/sites-available/api.conf.bak \
        /etc/nginx/sites-available/api.conf

sudo nginx -t

sudo systemctl reload nginx
```

Important: a successful `nginx -t` validates configuration loading, not the application's actual behaviour. Functional verification is still required.

---

# 23. Practical Exercises

The best way to become comfortable reading NGINX configurations is to practise interpreting them.

## Exercise 1: Identify the contexts

Given:

```nginx
http {
    upstream backend {
        server 127.0.0.1:8080;
    }

    server {
        listen 80;

        location /api/ {
            proxy_pass http://backend;
        }
    }
}
```

Questions:

1. How many contexts are present?
2. Which context contains the upstream?
3. Which context contains the proxy_pass directive?
4. What is the name of the upstream group?

## Exercise 2: Understand include

Given:

```nginx
http {
    include /etc/nginx/conf.d/*.conf;
}
```

Questions:

1. What does `include` do?
2. What does `*.conf` mean?
3. Does NGINX automatically load every file under `/etc/nginx`?
4. Why must the included configuration be valid inside the HTTP context?

## Exercise 3: Understand symbolic links

Given:

```text
sites-available/api.conf
sites-enabled/api.conf -> ../sites-available/api.conf
```

Questions:

1. Which file contains the actual configuration?
2. Is the enabled file a copy?
3. What happens when the symlink is deleted?
4. Why does NGINX load the configuration?

## Exercise 4: Understand variables

Given:

```nginx
proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-Proto $scheme;
```

Explain each variable and why the application might need the corresponding header.

## Exercise 5: Understand location matching

Given:

```nginx
location / {
    ...
}

location /api/ {
    ...
}

location = /api/health {
    ...
}

location ~ \.php$ {
    ...
}
```

Determine which location handles:

```text
/
/api/users
/api/health
/index.php
```

## Exercise 6: Understand proxy_pass

Given:

```nginx
location /api/ {
    proxy_pass http://localhost:8080/;
}
```

What URI does the upstream receive when the original request is:

```text
/api/users
```

What changes if the trailing URI slash is removed from `proxy_pass`?

## Exercise 7: Read a production-style configuration

Interpret:

```nginx
upstream backend {
    least_conn;

    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 443 ssl;

    server_name api.example.com;

    client_max_body_size 20M;

    location /api/ {
        proxy_pass http://backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }

    location = /health {
        access_log off;
        return 200 "OK";
    }
}
```

Identify:

- The upstream group.
- The load-balancing algorithm.
- The virtual server.
- The request-body limit.
- The API routing destination.
- The forwarded headers.
- The health endpoint behaviour.

---

# 24. Official Documentation

The official NGINX documentation should be your primary reference when encountering unfamiliar directives.

| Resource | Link |
|---|---|
| Beginner's Guide | https://nginx.org/en/docs/beginners_guide.html |
| Main Module | https://nginx.org/en/docs/ngx_core_module.html |
| HTTP Core Module | https://nginx.org/en/docs/http/ngx_http_core_module.html |
| Proxy Module | https://nginx.org/en/docs/http/ngx_http_proxy_module.html |
| Upstream Module | https://nginx.org/en/docs/http/ngx_http_upstream_module.html |
| Map Module | https://nginx.org/en/docs/http/ngx_http_map_module.html |
| Rewrite Module | https://nginx.org/en/docs/http/ngx_http_rewrite_module.html |
| SSL Module | https://nginx.org/en/docs/http/ngx_http_ssl_module.html |
| Log Module | https://nginx.org/en/docs/http/ngx_http_log_module.html |
| Access Module | https://nginx.org/en/docs/http/ngx_http_access_module.html |
| Limit Request Module | https://nginx.org/en/docs/http/ngx_http_limit_req_module.html |

---

# Final Notes

When reading an unfamiliar NGINX configuration, remember these principles:

1. **Identify the context before interpreting a directive.** The same word can have different roles depending on where it appears.
2. **Follow include directives.** A configuration file may depend on several other files.
3. **Understand symbolic links.** A file in `sites-enabled` may point to a different physical file.
4. **Pay attention to braces and semicolons.** They establish the configuration's grammar.
5. **Understand variables.** Anything beginning with `$` deserves attention.
6. **Learn basic regular expressions.** They are essential for understanding location matching and rewrites.
7. **Understand inheritance.** A directive's effective value may come from a parent context.
8. **Trace requests.** Follow a request from server selection through location matching to its final destination.
9. **Never assume a file is active.** Check the actual include chain.
10. **Validate before reloading.** Use `nginx -t` and verify actual application behaviour after changes.

The ultimate goal is to become comfortable reading NGINX configurations as naturally as you read source code.

You do not need to memorize every NGINX directive. You need to understand the configuration grammar, know how to investigate unfamiliar directives, and be able to trace how configuration decisions affect incoming requests.
