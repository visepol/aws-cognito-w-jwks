# AWS Cognito Integration with TypeScript

This repository is the companion code for the article
["OAuth2, JWT, and JWKS: Using Amazon Cognito as IdP"](https://dev.to/visepol/oauth2-jwt-and-jwks-using-amazon-cognito-as-idp-1jod) on Dev.to.

A copy of the article is kept in this repository at [`docs/oauth2-jwt-jwks-cognito.md`](./docs/oauth2-jwt-jwks-cognito.md), images included, so it stays readable if the original link ever goes away.

## Overview

A small [Fastify](https://fastify.dev/) API with two endpoints:

- `POST /authenticate` — sends the username and password to Cognito through the AWS SDK
  (`InitiateAuth` with the `USER_PASSWORD_AUTH` flow) and returns the **ID token** issued
  by the user pool.
- `GET /verify-jwt` — reads the `kid` from the token header, fetches the matching public
  key from the user pool's JWKS endpoint, and verifies the token's RS256 signature
  (and its expiration) with `jsonwebtoken`.

## Getting Started

### Prerequisites

- Node.js 16 through 24
  - 16 is the minimum required by `@aws-sdk/client-cognito-identity-provider`.
  - Node.js 25 fails at startup: `buffer-equal-constant-time`, pulled in by
    `jsonwebtoken → jws → jwa`, reads `SlowBuffer`, which Node.js 25 removed.
- npm
- An AWS account with a Cognito user pool, set up as described in the article:
  - an app client of type **Public client** (no client secret — the code does not send a
    `SECRET_HASH`) with the **ALLOW_USER_PASSWORD_AUTH** flow enabled;
  - a user whose status is **Confirmed**. The article sets a permanent password with
    `aws cognito-idp admin-set-user-password`, which requires AWS CLI credentials.

> **Tested environment:** the article was produced on Linux x86_64 (Ubuntu 20.04 under WSL).
> Other platforms are untested against a real user pool.

### Installation

```sh
git clone git@github.com:visepol/aws-cognito-w-jwks.git
cd aws-cognito-w-jwks
npm install
```

Use `npm install` rather than `npm ci`: the lockfile was generated on Linux x86_64 and
only records that platform's `esbuild` binary, so `npm ci` refuses to run elsewhere.

### Configuring Environment Variables

Copy `.env.example` to `.env` and fill in the values from your user pool:

```sh
cp .env.example .env
```

| Variable            | Required | Description                                                                                                                                          |
| ------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `COGNITO_CLIENT_ID` | yes      | ID of the app client (App integration → App client list).                                                                                            |
| `AWS_REGION`        | no       | Region of the user pool. Defaults to `us-east-1`.                                                                                                    |
| `JWKS_URI`          | yes      | The pool's "Token signing key URL": `https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/jwks.json`.                               |
| `DEBUG`             | no       | Set to `jwks` to turn on `jwks-rsa` debug logging.                                                                                                   |

### Running the Server

```sh
npm start
```

This runs `tsx watch src/server.ts`, which reloads on file changes. The server listens on
`0.0.0.0:3000` and prints:

```
🍃 HTTTP Server Running.
```

## Making Requests

[`request.http`](./request.http) holds both requests, ready for the VS Code REST Client
extension or the JetBrains HTTP Client. Replace the credentials with your user's, and
`<Token>` with the token returned by `/authenticate`. With curl:

```sh
curl -X POST http://localhost:3000/authenticate \
  -H 'content-type: application/json' \
  -d '{"username": "<username or email>", "password": "<password>"}'

curl http://localhost:3000/verify-jwt \
  -H 'Authorization: Bearer <token>'
```

### Responses

| Endpoint             | Situation                                                               | Status |
| -------------------- | ----------------------------------------------------------------------- | ------ |
| `POST /authenticate` | Valid credentials                                                       | `201` with `{ "token": "..." }` |
| `POST /authenticate` | Cognito answers with a challenge instead of tokens (e.g. the user still has to change the password) | `401` |
| `GET /verify-jwt`    | Valid signature, not expired                                            | `200` |
| `GET /verify-jwt`    | Token that cannot be decoded                                            | `401` |

Every other failure reaches the generic error handler in `src/app.ts`, which logs the
error and responds with `500`. That includes wrong credentials (Cognito throws
`NotAuthorizedException`), an invalid or expired signature, a `kid` not present in the
JWKS, a missing `Authorization` header, and a request body that fails validation.

> **Scope of the validation:** `/verify-jwt` only checks the signature and the `exp` claim.
> It does not check `iss`, `aud`, or `token_use`, which a production resource server
> should also validate.

## Project Structure

```sh
aws-cognito-w-jwks/
│── src/
│   ├── server.ts                        # Loads .env and starts the server on port 3000
│   ├── app.ts                           # Fastify instance, routes, and error handler
│   ├── http/
│   │   ├── _routes.ts                   # Route registration
│   │   ├── authenticate.controller.ts   # POST /authenticate
│   │   └── verify-jwt.controller.ts     # GET /verify-jwt
│   └── lib/
│       ├── cognito.ts                   # InitiateAuth call (USER_PASSWORD_AUTH)
│       └── jwks.ts                      # JWKS lookup and signature verification
│── docs/                                # Local copy of the Dev.to article + its images
│── request.http                         # Example requests
│── .env.example                         # Environment variables template
│── package.json                         # Dependencies and scripts
└── README.md                            # Project documentation
```
