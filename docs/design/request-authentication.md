# Request Authentication

## Overview

[Issue 336][] specifies the need for exposing user-facing APIs to configure request authentication. Envoy Gateway
leverages [Gateway API][] for configuring managed Envoy proxies. Gateway API defines core, extended, and
implementation-specific API [support levels][] for implementors such as Envoy Gateway to expose features. Since
implementing request authentication is not covered by `Core` or `Extended` APIs, an `Implementation-specific` API will
be created for this purpose.

Request authentication specifies authentication mechanisms to be enforced by Envoy on a per-request basis. A connection
will be rejected if it contains invalid authentication information, based on the configured authentication rules. This
includes the credential (X.509, JWT, etc), parameters (cipher suites, key algorithms). A request that does not contain
any authentication credentials will be accepted but will not have any authenticated identity. The policy is similar to
[OpenAPI 3.1 security objects][] without the API key part, and should be easily translatable from it with some
additions.

## Goals

* Define an API for configuring request authentication, using [JWT] as the first implemented authentication type.
* Provide the foundation for exposing Gateway API `Implementation-specific` features.
* Allow users that manage routes, e.g. [HTTPRoute][], to include authentication for matched requests before forwarding
  to a backend.

## Non-Goals

* Allow infrastructure administrators to override or establish default authentication policies.

## Use-Cases

These use-cases are presented as an aid for discussion, and a reference for how users may attempt to utilize the outputs
of the design. They are not an exhaustive list of features for authentication support in Envoy Gateway. As a service
producer, I need the ability to:

1. Authenticate a request before forwarding it to a backend service.
2. Have different authentication mechanisms per route rule.
3. Choose from different authentication mechanisms supported by Envoy, e.g. OIDC.
4. Others?

### Security API Group

A new API group, `security.gateway.envoyproxy.io` is introduced to group security-related APIs. This will allow security
APIs to be versioned, changed, etc. over time.

### AuthenticationProvider API Type

The `AuthenticationProvider` API type defines an authentication provider for authenticating requests through managed
Envoy proxies for supported route types, e.g. `HTTPRoute`.

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

)

type AuthenticationProvider struct {
	metav1.TypeMeta
	metav1.ObjectMeta

	// Spec defines the desired state of the Authentication type.
	Spec AuthenticationProviderSpec
	// Status defines the actual state of the Authentica type.
	Status AuthenticationProviderStatus
}

// AuthenticationProviderSpec defines the desired state of the AuthenticationProvider type.
// +union
type AuthenticationProviderSpec struct {
	// Type defines the type of authentication provider to use. Supported provider types are:
	//
	// * JWT
	//
	//   JWT defines the JSON Web Token (JWT) authentication provider type.
	//
	// +unionDiscriminator
	Type AuthenticationProviderType `json:"type"`
	
	// JWT defines the JSON Web Token (JWT) authentication provider type. For additional
	// details, see:
	//
	//   https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/jwt_authn_filter.html
	//
	// +optional
	JwtProviders []JwtAuthenticationProvider `json:"jwtProviders"`
}

// AuthenticationProviderType is a type of authentication provider.
// +kubebuilder:validation:Enum=JWT
type AuthenticationProviderType string

const (
	// JwtAuthenticationProviderType is the JWT authentication provider type.
	JwtAuthenticationProviderType AuthenticationProviderType = "JWT"
)

// JwtAuthenticationProvider defines the JSON Web Token (JWT) authentication provider type
// and how JWTs should be verified:
type JwtAuthenticationProvider struct {
	// Name defines the of the JWT provider. A name can have a variety of forms,
	// including RFC1123 subdomains, RFC 1123 labels, or RFC 1035 labels.
	//
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=253
	Name string `json:"name"`

	// Issuer is the principal that issued the JWT.	For additional details, see:
	//
	//   https://tools.ietf.org/html/rfc7519#section-4.1.1
	//
	// Example:
	//  issuer: https://auth.example.com
	//
	// If not provided, JWT issuers are not checked.
	//
	//
	// +optional
	Issuer string `json:"issuer,omitempty"`

	// Audiences is a list of JWT audiences allowed to access. For additional details, see:
	//
	//   https://tools.ietf.org/html/rfc7519#section-4.1.3
	//
	// Example:
	//   audiences:
	//   - foo.apps.example.com
	//     bar.apps.example.com
	//
	// If not provided, JWT audiences are not checked.
	//
	// +optional
	Audiences []string `json:"audiences,omitempty"`

	// RemoteJWKS defines how to fetch and cache JSON Web Key Sets (JWKS) from a
	// remote HTTP/HTTPS endpoint.
	//
	// +kubebuilder:validation:Required
	RemoteJWKS RemoteJWKS `json:"remoteJWKS"`

	// TODO: Add TBD fields.
}

// RemoteJWKS defines how to fetch and cache JSON Web Key Sets (JWKS) from a remote
// HTTP/HTTPS endpoint.
type RemoteJWKS struct {
  // TODO
}

type AuthenticationStatus struct {
	// TODO
}

```


The following is a JWT authentication provider example:

```yaml
apiVersion: security.gateway.envoyproxy.io/v1alpha1
kind: AuthenticationProvider
metadata:
  name: example
spec:
  type: JWT
  jwtProviders:
  - name: example
    issuer: https://www.example.com
    audiences:
    - foo.com
    remoteJwks:
      url: https://foo.com/jwt/public-key/jwks.json
      <INSERT>
status:
  <INSERT>
```

The JWT authentication types is translated to Envoy's JWT authentication filter. The JWKS URI need to be translated to a
separate cluster for JWKS fetch and refresh.

This is the Envoy HTTP filter config for JWT authentication specified in the above `AuthenticationProvider`.

```yaml
providers:
   example:
     issuer: https://www.example.com
     audiences:
     - foo.com
     remote_jwks:
       http_uri:
         uri: https://foo.com/jwt/public-key/jwks.json
         cluster: example_jwks_cluster
         timeout: 1s
```

Envoy Gateway adds a cluster via [CDS][] for the remote JWKS, as required by the Envoy configuration.

## Additional Authentication Provider Types

Additional authentication provider types can be added in the future through the `ProviderType` API. For example, to add
the `Foo` authentication provider type:

Define the `FooProvider` type:

```go
// FooProvider defines the "Foo" authentication provider type.
type FooProvider struct {
	// TODO: Define fields of the Foo authentication provider type.
}
```

Add the `FooProvider` type to `AuthenticationProviderSpec`:

```go
type AuthenticationProviderSpec struct {
	...
	
	// Foo defines the Foo authentication type. For additional
	// details, see:
	//
	//   <INSERT_LINK>
	//
	// +optional
	Foo *FooAuthentication `json:"foo"`
}
```

Additional authentication types that may be supported are:
- mutualTLS (client certificate)
- OAuth2
- OIDC
- External authentication

In general, authentication types translate into Envoy HTTP filters at HTTP connection manager level. The authentication
APIs defined here are referenced as extended filters of supported route types, e.g. HTTPRoute.

[Issue 336]: https://github.com/envoyproxy/gateway/issues/336
[Gateway API]: https://gateway-api.sigs.k8s.io/
[support levels]: https://gateway-api.sigs.k8s.io/concepts/conformance/?h=extended#2-support-levels
[HTTPRoute]: https://gateway-api.sigs.k8s.io/api-types/httproute/
[OpenAPI 3.1 security objects]: https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.1.0.md#securitySchemeObject
[CDS]: https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cds
[JWKS]: https://www.rfc-editor.org/rfc/rfc7517
