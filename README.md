Go EST Client
==========
[![GoDoc](https://godoc.org/github.com/thales-e-security/estclient?status.svg)](https://godoc.org/github.com/thales-e-security/estclient)
[![Build Status](https://travis-ci.org/thales-e-security/estclient.svg?branch=master)](https://travis-ci.org/thales-e-security/estclient)

A minimal EST client SDK, written in Go (see [RFC 7030](https://tools.ietf.org/html/rfc7030)). 

Support is provided for calling the EST endpoints:
- `/.well-known/est/cacerts`
- `/.well-known/est/simpleenroll`
- `/.well-known/est/simplereenroll`

Initial enrollment must use HTTP authentication (id and secret). Re-enrollment uses mutual TLS with a previously issued
certificate and optionally HTTP authentication.  

## Examples

See `examples/examples.go` for an example that uses the test server at http://www.testrfc7030.com.

To execute the example script, run `go run examples/examples.go`. Read on for a description of the code.

### Get the EST certificates

```go
client := estclient.NewEstClient("testrfc7030.com:8443")

cacerts, err := client.CaCerts()
if err != nil {
    panic(err)
}

fmt.Printf("EST Root Cert: %+v\n", cacerts.EstTA.Subject)
fmt.Printf("Old EST Root Cert: %+v\n", cacerts.OldWithOld)
fmt.Printf("Old Cert Signed By New Key: %+v\n", cacerts.OldWithNew)
fmt.Printf("New Cert Signed By Old Key: %+v\n", cacerts.NewWithOld)
fmt.Printf("Other chain certs: %+v\n", cacerts.EstChainCerts)
```

The `EstTA` cert is the root of trust for all certificates issued by the EST CA. The EST CA may be a subordinate of
this trust anchor, in which case `EstChainCerts` should contain the necessary intermediate certificates to validate
certificates issued by that EST CA.

If the root of trust has been renewed, there may be cross-signed certificates returned in `OldWithOld`, `OldWithNew`
and `NewWithOld`.

### Request a certificate

```go
// Create key and certificate request
key, err := rsa.GenerateKey(rand.Reader, 2048)
panicOnError(err)

template := x509.CertificateRequest{Subject: pkix.Name{CommonName: "Test"}}

reqBytes, err := x509.CreateCertificateRequest(rand.Reader, &template, key)
panicOnError(err)

req, err := x509.ParseCertificateRequest(reqBytes)
panicOnError(err)

// Enroll with EST CA
cert, err := client.SimpleEnroll("estuser", "estpwd", req)
panicOnError(err)
fmt.Printf("Initial cert (DER): %x\n", cert.Raw)
```

### Renew a certificate

This client assumes mutual TLS will be required, using the previously issued key and certificate. Additional HTTP
authentication (with an id and secret) is optional. Pass `nil` if these values are not required by the server.


```go
// Re-enroll with EST CA
id := "estuser"
secret := "estpwd"
cert2, err := client.SimpleReenroll(&id, &secret, key, cert, req)
panicOnError(err)
fmt.Printf("Renewed cert (DER): %x\n", cert2.Raw)
```

## Contributing

Contributions are very welcome. Please ensure you add tests for new functionality and consider opening an issue
to discuss larger changes before beginning.

When developing, you can disable integration tests (which hit `testrfc7030.com:8443`) by running 
`go test -short ./...`.

## TODOs

- [ ] Improve coverage of non-happy paths
- [ ] Support more initial enrollment (and re-enrollment) authentication options. 
- [ ] Tackle any "N" items from the analysis of the RFC below
- [ ] Convert any "TBC" items into "Y" or "N"

## Analysis of RFC requirements

| Category | Item                                                                                                                                                                                                                                                                                                                                                                                                                          | DONE | Notes                                                                                                    | 
|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------|----------------------------------------------------------------------------------------------------------|
| MUST     | NULL and anon cipher suites MUST NOT be used because they do not provide confidentiality or support mutual certificate-based or certificate-less authentication, respectively.                                                                                                                                                                                                                                                | N    | Not currently enforced                                                                                   | 
| MUST     | If the CSR KeyUsage extension indicates that the private key can be used to generate digital signatures, then the client MUST generate the CSR signature using the private key.                                                                                                                                                                                                                                               | N    | This is not enforced currently.                                                                          | 
| MUST     | The client MUST wait at least the specified "retry-after" time before repeating the same request.                                                                                                                                                                                                                                                                                                                             | N    | This is not enforced.                                                                                    | 
| MUST     | HTTP Basic and Digest authentication MUST only be performed over TLS 1.1 [RFC4346] or later versions.                                                                                                                                                                                                                                                                                                                         | TBC  | Not sure if we do/can enforce this                                                                       | 
| MUST     | TLS 1.1 [RFC4346] (or a later version) MUST be used for all EST communications.                                                                                                                                                                                                                                                                                                                                               | TBC  | Need to check if this is enforced.                                                                       | 
| MUST     | Certificate validation MUST be performed as per [RFC5280].                                                                                                                                                                                                                                                                                                                                                                    | TBC  | Not sure if compliant - rely on the built-in validation within Go.                                       | 
| MUST     | The EST client MUST store the extracted EST CA certificate as an Explicit TA database entry for subsequent EST server authentication.                                                                                                                                                                                                                                                                                         | TBC  | I have raised a question with IETF about this. Don't believe it should be a MUST.                        | 
| MUST     | If the client disables the Implicit TA database, and if the EST server certificate was verified using an Implicit TA database entry, then the client MUST include the "Trusted CA Indication" extension in future TLS sessions [RFC6066].                                                                                                                                                                                     | TBC  | In part, depending upon response from IETF.                                                              | 
| MUST     | TLS cipher suites that include "_EXPORT_" and "_DES_" in their names MUST NOT be used.                                                                                                                                                                                                                                                                                                                                        | TBC  | Not sure if this is automatically the case with Go TLS library.                                          | 
| MUST     | The EST client MUST be able to handle these certificates in the response.                                                                                                                                                                                                                                                                                                                                                     | Y    | Note: this refers to the OldWithOld, OldWithNew, NewWithOld certs                              | 
| MUST     | Implementations conforming to this standard MUST provide the ability to designate Explicit TAs.                                                                                                                                                                                                                                                                                                                               | Y    |                                                                        | 
| MUST     | Implementations conforming to this standard MUST provide the ability to disable use of any Implicit TA database.                                                                                                                                                                                                                                                                                                              | Y    | Implicit TA database is disabled when explicit TA database provided.                                                               | 
| MUST     | The EST client MUST be capable of generating and parsing Simple PKI messages (see Section 4.2).                                                                                                                                                                                                                                                                                                                               | Y    |                                                                                                          | 
| MUST     | The client MUST also be able to request CA certificates from the EST server and parse the returned "bag" of certificates (see Section 4.1).                                                                                                                                                                                                                                                                                   | Y    |                                                                    | 
| MUST     | If the client is not capable of retrying the request or it is not capable of Basic or Digest authentication, then the client MUST terminate the connection.                                                                                                                                                                                                                                                                   | Y    |                                                                                                          | 
| MUST     | The client MUST NOT respond to the server's HTTP authentication request unless the client has authorized the EST server (as per Section 3.6).                                                                                                                                                                                                                                                                                 | Y    |                                                                                                          | 
| MUST     | The EST server MUST be authenticated during the TLS handshake unless the client is requesting Bootstrap Distribution of CA certificates (Section 4.1.1) or Full CMC (Section 4.3).                                                                                                                                                                                                                                            | Y    | 4.1.1 is not supported.                                                                                  | 
| MUST     | HTTPS MUST be used.                                                                                                                                                                                                                                                                                                                                                                                                           | Y    |                                                                                                          | 
| MUST     | Certificate validation MUST be performed as per [RFC5280].                                                                                                                                                                                                                                                                                                                                                                    | Y    |                                                                                                          | 
| MUST     | The EST server certificate MUST conform to the [RFC5280] certificate profile.                                                                                                                                                                                                                                                                                                                                                 | Y    |                                                                                                          | 
| MUST     | The client MUST maintain a distinction between the use of Explicit and Implicit TA databases during authentication in order to support proper authorization.                                                                                                                                                                                                                                                                  | Y    | Although there doesn't appear to be much meaningful difference between the two authorisation approaches. | 
| MUST     | The EST client MUST perform authorization checks as specified in Section 3.6.                                                                                                                                                                                                                                                                                                                                                 | Y    | Partial support, no checking of id-kp-cmcRA. That part of RFC is ambiguous.                              | 
| MUST     | If the certificate to be renewed or rekeyed is appropriate for the negotiated cipher suite, then the client MUST use it for the TLS handshake, otherwise the client SHOULD use an alternate certificate that is suitable for the cipher suite and contains the same subject identity information.                                                                                                                             | Y    | Yes, assuming that the certificates issued by the CA are suitable for use in mutual TLS.                 | 
| MUST     | The EST client certificate MUST conform to the [RFC5280] certificate profile.                                                                                                                                                                                                                                                                                                                                                 | Y    |                                                                                                          | 
| MUST     | When performing renegotiation, TLS "secure_renegotiation" [RFC5746] MUST be used.                                                                                                                                                                                                                                                                                                                                             | Y    | Supported in the sense that Go defaults to disabling renegotiation.                                      | 
| MUST     | The client MUST check EST server authorization before accepting any server responses or responding to HTTP authentication requests.                                                                                                                                                                                                                                                                                           | Y    |                                                                                                          | 
| MUST     | When the EST client Explicit TA database is used to validate the EST server certificate, the client MUST check either the configured URI or the most recent HTTP redirection URI against the server's identity according to the rules specified in [RFC6125], Section 6.4, or the EST server certificate MUST contain the id-kp-cmcRA [RFC6402] extended key usage extension.                                                 | Y    | We check against the configured URI.                                                                     | 
| MUST     | When the EST client Implicit TA database is used to validate the EST server certificate, the client MUST check the configured URI and each HTTP redirection URI according to the rules specified in [RFC6125], Section 6.4.                                                                                                                                                                                                   | Y    |                                                                                                          | 
| MUST     | HTTP authentication requests MUST NOT be responded to if the server has not been authenticated as specified in Section 3.3.1 or if the optional certificate-less authentication is used as specified in Section 3.3.3.                                                                                                                                                                                                        | Y    |                                                                                                          | 
| MUST     | EST clients and servers MUST support the /cacerts function.                                                                                                                                                                                                                                                                                                                                                                   | Y    |                                                                                                          | 
| MUST     | The client MUST authenticate the EST server, as specified in Section 3.3.1 if certificate-based authentication is used or Section 3.3.3 if the optional certificate-less authentication is used, and check the server's authorization as given in Section 3.6, or follow the procedure outlined in Section 4.1.1.                                                                                                             | Y    |                                                                                                          | 
| MUST     | Any other response code indicates an error and the client MUST abort the protocol.                                                                                                                                                                                                                                                                                                                                            | Y    |                                                                                                          | 
| MUST     | The client MUST authenticate the EST server as specified in Section 3.3.1 if certificate-based authentication is used or Section 3.3.3 if the optional certificate-less authentication is used.                                                                                                                                                                                                                               | Y    |                                                                                                          | 
| MUST     | The client MUST verify the authorization of the EST server as specified in Section 3.6.                                                                                                                                                                                                                                                                                                                                       | Y    |                                                                                                          | 
| MUST     | When HTTPS POSTing to /simpleenroll, the client MUST include a Simple PKI Request as specified in CMC [RFC5272], Section 3.1 (i.e., a PKCS #10 Certification Request [RFC2986]).                                                                                                                                                                                                                                              | Y    |                                                                                                          | 
| MUST     | A client MUST authenticate an EST server, as specified in Section 3.3.1 if certificate-based authentication is used or Section 3.3.3 if the optional certificate-less authentication is used, and check the server's authorization as given in Section 3.6.                                                                                                                                                                   | Y    |                                                                                                          | 
| MUST     | The client MUST ignore any OID or attribute it does not recognize.                                                                                                                                                                                                                                                                                                                                                            | Y    |                                                                                                          | 
| MUST     | If a client does not support TLS client authentication, then it MUST support HTTP-based client authentication (Section 3.2.3) or certificate-less TLS authentication (Section 3.3.3)                                                                                                                                                                                                                                          | n/a  | We support client auth                                                                                   | 
| MUST     | When using certificate-less mutual authentication in TLS for enrollment, the cipher suite MUST be based on a protocol that is resistant to dictionary attack and MUST be based on a zero knowledge protocol.                                                                                                                                                                                                                  | n/a  | This is not a supported authentication mode.                                                             | 
| MUST     | If the EST client continues with an unauthenticated connection, the client MUST extract the HTTP content data from the response (Sections 4.1.3 or 4.3.2) and engage a human user to authorize the CA certificate using out-of-band data such as a CA certificate "fingerprint" (e.g., a SHA-256 or SHA-512 [SHS] hash on the whole CA certificate).                                                                          | n/a  | Not a supported approach                                                                                 | 
| MUST     | EST clients MUST NOT engage in any other protocol exchange until after the /cacerts response has been accepted and a new TLS session has been established (using TLS certificate-based Authentication).                                                                                                                                                                                                                       | n/a  | 4.1.1 not supported                                                                                      | 
| MUST     | After out-of-band validation occurs, all the other certificates MUST be validated using normal [RFC5280] certificate path validation (using the most recent CA certificate as the TA) before they can be used to build certificate paths during certificate validation.                                                                                                                                                       | n/a  |                                                                                                          | 
| MUST     | If the key can be used to generate digital signatures but the requested CSR KeyUsage extension prohibits generation of digital signatures, then the CSR signature MAY still be generated using the private key, but the key MUST NOT be used for any other signature operations (this is consistent with the recommendations concerning submission of proof-of-possession to an RA or CA as described in [SP-800-57-Part-1]). | n/a  | This is surely outside the scope of what the EST client can enforce.                                     | 
| MUST     | Cipher suites that have a NULL confidentiality algorithm MUST NOT be used as they will disclose the contents of an unprotected private key.                                                                                                                                                                                                                                                                                   | n/a  | No support for server key gen                                                                            | 
| MUST     | If the client desires to receive the private key with encryption that exists outside of and in addition to that of the TLS transport used by EST or if server policy requires that the key be delivered in such a form, the client MUST include an attribute in the CSR indicating the encryption key to be used.                                                                                                             | n/a  |                                                                                                          | 
| MUST     | The client MUST also include an SMIMECapabilities attribute ([RFC2633], Section 2.5) in the CSR to indicate the key encipherment algorithms the client is willing to use.                                                                                                                                                                                                                                                     | n/a  |                                                                                                          | 
| MUST     | To specify a symmetric encryption key to be used to encrypt the server-generated private key, the client MUST include a DecryptKeyIdentifier attribute (as defined in Section 2.2.5 of [RFC4108]) specifying the identifier of the secret key to be used by the server to encrypt the private key.                                                                                                                            | n/a  |                                                                                                          | 
| MUST     | To specify an asymmetric encryption key to be used to encrypt the server-generated private key, the client MUST include an AsymmetricDecryptKeyIdentifier attribute.                                                                                                                                                                                                                                                          | n/a  |                                                                                                          | 
