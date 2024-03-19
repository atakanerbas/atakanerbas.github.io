---
layout: post
title: "Using AWS Cognito with Golang middlewares"
description: "A guide to implement a middleware for Go net/http handlers to authenticate with AWS Cognito"
date: 2024-08-21 15:41:52 +0300
categories: programming
image: "/assets/image/gopher.jpg"
---

In this article:

[Introduction](#introduction) \\
[The Flow](#the-flow) \\
[Basic HTTP Server and Middleware](#basic-http-server-and-middleware) \\
[How to Verify a JWT](#how-to-verify-a-jwt) \\
[Implementing the Middleware](#implementing-the-middleware)
### Introduction

Using AWS Cognito with Go is a bit tricky as Cognito lacks examples of token verification, and golang-jwt is still under development.
I will show how to implement a Go middleware for Cognito token authentication and closure for reusability, as I could not find a good source for it!

### The Flow

I will not go into details of JWT flow, but simply, if you are going to use Cognito user pool authentication flow for authentication, the flow will be:

1. User authenticates via Cognito
2. User sends a request to the backend with Authorization headers, with a token from Cognito.
3. Check the headers and get the Authorization field.
4. Decode the token
5. Validate/Verify the token
6. Ensure the token is signed by the auth server, and verify the signature using the auth server's public keys.
7. Check the claims, depending on auth service.
8. Check the scopes of the user.
9. Serve the request.

### Basic HTTP Server and Middleware

I assume you have set up the user pool and created a client in AWS Console and users can authenticate with the hosted UI/custom login page.
The minimal HTTP server for Go is:

```go
package main

import (
    "fmt"
    "net/http"
)

func SuperSafeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", SuperSafeHandler)
    http.ListenAndServe(":8080", nil)
}

```

We will implement a middleware to check the Authorization header and verify the token. Basically, middlewares are functions executed as a part of the request-response cycle.
In Go, middlewares are implemented as a function that takes a handler function and returns a handler function, allowing it to be chained and executed in order. For example:

```go
package main

import "fmt"
import "log"
import "net/http"

func DummyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println("Before")
        next.ServeHTTP(w, r)
        log.Println("After")
    })
}

func SuperSafeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

func main() {
    mux := http.NewServeMux()
    safeHandler := http.HandlerFunc(SuperSafeHandler)
    mux.Handle("/", DummyMiddleware(safeHandler))
    http.ListenAndServe(":8080", mux)
}
```

http.HandlerFunc is a type which allows us to convert a function of f(w http.ResponseWriter, r \*http.Request) to a type that implements the http.Handler interface. After that, we can apply the middleware.
For further information, you can check the [official net/http documentation](https://golang.org/pkg/net/http/#HandlerFunc).
Also worth reading:

- [Eli Bendersky - Life of an HTTP request in a Go server](https://eli.thegreenplace.net/2021/life-of-an-http-request-in-a-go-server/)
- [Alex Edwards - Making and Using HTTP Middleware](https://www.alexedwards.net/blog/making-and-using-middleware)

### How to verify a JWT?

Cognito exposes a JWKS endpoint, which contains the public keys that can be used to verify the token. We will fetch the keys and verify the token with the keys. Format of the Cognito JWKS URL is:

`https://cognito-idp.{region}.amazonaws.com/{userPoolId}/.well-known/jwks.json`

We will fetch the keys from the JWKS endpoint and verify the token with the keys. Before implementing the middleware, let's implement a function which end to end verifies the token. We will use jwt-go package, which can be installed with:

```bash
go get github.com/golang-jwt/jwt/v5
```

Please note that: **You need to cache the keys in a concurrency-safe manner and refresh after TTL; fetching the keys for each request is not a good idea. You also need a config struct for the values.** I will provide an example of a configurable middleware at the end of the post.

A simple function to validate the token:

```go
package main

import (
    "crypto/rsa"
    "encoding/json"
    "encoding/base64"
    "fmt"
    "log"
    "math/big"
    "net/http"

    "github.com/golang-jwt/jwt/v5"
)

type jwk struct {
    KeyType string `json:"kty"`
    KID    string `json:"kid"`
    Algorithm string `json:"alg"`
    N string `json:"n"`
    E string `json:"e"`
    Use string `json:"use"`
}

type jwkContainer struct {
    Keys []jwk `json:"keys"`
}

func ValidateToken(region, userPoolId, clientId, tokenString string) {
    // Fetch the JWKS
    jwksURL := fmt.Sprintf("https://cognito-idp.%s.amazonaws.com/%s/.well-known/jwks.json", region, userPoolId)
    publicKeys, err := fetchJWKS(jwksURL)
    if err != nil {
        log.Println("Error fetching JWKS: ", err)
        return
    }
    // Parse the token. Parse takes the token string and a function as parameters, the function must return a key.
    token, err := jwt.Parse(tokenString, func(token *jwt.Token)(interface{}, error) {
        kid := token.Header["kid"].(string)
        for _, key := range publicKeys.Keys {
            if key.KID == kid {
                n, err := base64.RawURLEncoding.DecodeString(key.N)
                if err != nil {
                    return nil, err
                }
                e, err := base64.RawURLEncoding.DecodeString(key.E)
                if err != nil {
                    return nil, err
                }
                return &rsa.PublicKey{
                    N: big.NewInt(0).SetBytes(n),
                    E: int(big.NewInt(0).SetBytes(e).Int64())}, nil
            }
        }
        return nil, fmt.Errorf("unable to find key")
    })
    if err != nil || token == nil {
        log.Println("Error parsing token: ", err)
        return
    }
    // Parse the claims
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok || !token.Valid {
        log.Println("Invalid token")
        return
    }
    // Check the claims, some of these are only for Cognito,
    // https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html
    if claims["iss"] != fmt.Sprintf("https://cognito-idp.%s.amazonaws.com/%s", region, userPoolId) {
        log.Println("Invalid issuer")
        return
    }
    if claims["aud"] != clientId {
        log.Println("Invalid audience")
        return
    }
    // check access or id token
    // if claims["token_use"] != "access" {
    //     log.Println("Invalid token use")
    //     return
    // }

    // Check the scopes
    // ...
}

func fetchJWKS(wellknownUrl string) (jwkContainer, error) {
    resp, err := http.Get(wellknownUrl)
    if err != nil {
        return jwkContainer{}, err
    }
    defer resp.Body.Close()
    if resp.StatusCode != http.StatusOK {
        return jwkContainer{}, fmt.Errorf("unexpected status code: %d", resp.StatusCode)
    }

    var jwks jwkContainer
    if err := json.NewDecoder(resp.Body).Decode(&jwks); err != nil {
        return jwkContainer{}, err
    }
    return jwks, nil

}

func main() {
    ValidateToken(<YOUR_TOKEN>)


```

### Implementing the middleware

Now, we can implement the middleware. Middleware will also add a user ID to the context. My implementation is different from the usual ones; I will use a closure to pass the configuration values to the middleware. This way, we can reuse the middleware with different configurations.
Don't forget to:

- **Add a concurrency-safe caching mechanism for public keys. Use sync.RWMutex to lock and unlock the map to avoid race conditions. Periodically refresh the keys.**
  1. Call GetPublicKeys
  2. GetPublicKeys should check if the keys exist in the cache or not.
  3. If TTL is passed or keys do not exist in the cache, fetch JWK from .well-known
  4. For new keys, construct PublicKey objects and create the map.
  5. Return the keys.
- Logs, logs, logs.
- Call panic if you need to.
- **Do not assume the algorithm or key type, and don't forget to check the audience, issuer and algorithm.**
- You can use ParseWithClaims instead of Parse if you need to check the claims. Struct embedding can be used for custom claims. jwt.Parse is also calling ParseWithClaims under the hood with the type jwt.MapClaims, basically a map[string]interface{}.

The GetCognitoAuthMiddleware function returns a "middleware" function that accepts an http.Handler and returns an http.handler, using a closure. This allows us to configure the middleware.

```go

type CognitoMiddlewareConfig struct {
    Region string
    UserPoolId string
    ClientId string
}

func NewCognitoMiddleware(config CognitoMiddlewareConfig) func(http.Handler) http.Handler {
        middleware := func(w http.ResponseWriter, r *http.Request){
        // Get the token from the headers
            tokenString := r.Header.Get("Authorization")
            if tokenString == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            // Delete the Bearer prefix if exists
            tokenString = strings.TrimPrefix(tokenString, "Bearer ")
            // GetPublicKeys function must be implemented, get from cache or fetch from JWKS. Return a map of kid(string) -> *rsa.PublicKey
            // For testing, just define a global variable and fetch with the above fetchJWKS function.
            pubKeys, err := GetPublicKeys(config.Region, config.UserPoolId)
            if err != nil {
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
                return
            }
            token, err := jwt.Parse(
                tokenString,
                func(token *jwt.Token) (interface{}, error) {
                    kid := token.Header["kid"].(string)
                    key, ok := pubKeys[kid]
                    if !ok {
                        return nil, fmt.Errorf("unable to find key")
                    }
                    return key, nil
                },
                jwt.WithAudience(config.ClientId), // Checks "aud" claim.
                jwt.WithIssuer(fmt.Sprintf("https://cognito-idp.%s.amazonaws.com/%s",config.Region,config.UserPoolId)) // Checks "iss" claim
                jwt.WithValidMethods([]string{"RS256"}), // Cognito uses RS256 for signing. Important check [WithValidMethods] : https://pkg.go.dev/github.com/golang-jwt/jwt/v5#WithValidMethods
            )
            if err != nil || token == nil {
                fmt.Println(err)
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            if !token.Valid {
                fmt.Println("Token is not valid")
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }
            claims, _ := token.Claims.(jwt.MapClaims)
            // Don't forget to check the claims for groups, email_verified field and possibly add to the context.
            // API Gateway Authorizers let non verified emails to send requests.
            ctx := context.WithValue(r.Context(), "userId", claims["sub"])
            next.ServeHTTP(w, r.WithContext(ctx))

        }
    return func(next http.Handler) http.Handler{
        return http.HandlerFunc(middleware)
    }
}

```
This middleware can be created with a configuration struct with the help of a closure. You can use the middleware like this:

```go

cognitoMiddleware := NewCognitoMiddleware(CognitoMiddlewareConfig{
    Region: "us-east-1",
    UserPoolId: "us-east-1_XXXXX",
    ClientId: "XXXXXXXX"
})

mux := http.NewServeMux()
safeHandler := http.HandlerFunc(SuperSafeHandler)
mux.Handle("/", cognitoMiddleware(safeHandler))
http.ListenAndServe(":8080", mux)
```

However, environment variables should be used for the configuration values. For easier configuration management and debugging, you can use [godotenv](https://github.com/joho/godotenv) .

The following post will cover the caching mechanism and configuration of environment variables. Stay tuned!

