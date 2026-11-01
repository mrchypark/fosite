# GoAuthy temporary Fosite JOSE v4 migration

This is a complete source copy of canonical `github.com/ory/fosite` v0.49.0,
commit `653c812bc40cbda049857b432ad3346f1a53d42d`. The Apache-2.0 license,
source notices, upstream tests and fixtures remain. Root go.mod uses a local
replace for the canonical Fosite module; this is not an official Fosite release.

Purpose: remove go-jose v3 from GoAuthy's built authorization-server dependency
path and use official `github.com/go-jose/go-jose/v4` v4.1.6 for
[GHSA-mgcc-cww5-fp2q](https://github.com/go-jose/go-jose/security/advisories/GHSA-mgcc-cww5-fp2q).
The fixed JOSE artifact is upstream commit
`a0111f82742fa92bceec9ede40845e73319e6230`. No JOSE fork, synthetic fixed
version, parser override or scanner suppression is used.

[Upstream PR824](https://github.com/ory/fosite/pull/824) is closed, unmerged,
and reference material only. This source starts from the released v0.49.0 tree,
not that PR's unmerged source. The local migration is limited to:

- All Fosite Go imports of JOSE v3 become v4, including upstream tests.
- JWT builders use v4's Serialize in place of CompactSerialize.
- ParseSigned calls supply explicit HS256/384/512, RS256/384/512,
  ES256/384/512, PS256/384/512 and EdDSA parse algorithms. Both the token adapter
  and RFC7523 handler also retain parsing of `none`. This is parsing permission,
  not authorization: the adapter still requires the existing explicit
  UnsafeAllowNoneSignatureType for unsigned verification, and RFC7523 retains
  its configured trusted-key/signature/algorithm checks. Client policy remains.
- Verification keeps HMAC/Ed25519 keys as slices (including dereferencing their
  pointer forms); RSA/EC and other existing key conversion remains unchanged.
- The nested module directly requires official JOSE v4.1.6; its Go baseline
  becomes 1.24 as required by JOSE, below GoAuthy's 1.27.3.
- Two existing refresh WithDebugf calls use a literal format string so current
  Go vet can check them. An upstream RFC7523 test uses a 2048-bit RSA key rather
  than Go 1.27's rejected 512-bit key. Assertions remain unchanged.
- New token/jwt security tests cover all supported signed algorithms, wrong keys,
  callback ordering, malicious compact/JSON RSA headers and 2048/4096-bit JWK
  public/private round trips. GoAuthy endpoint tests are maintained separately.

The official JOSE v4.1.6 default RSA parse bound is 4096 bits; larger JWKs are
rejected. GoAuthy does not raise the bound. This is an explicit security-policy
compatibility change, not a silent claim that all historical key sizes work.

Run the complete isolated upstream suite, including its local HTTP integration
fixtures (no external services), explicitly because root ./... excludes modules:

```
(cd third_party/fosite && go test -p 3 -count=1 -timeout=3m ./...)
(cd third_party/fosite && go test -race -p 3 -count=1 -timeout=3m ./...)
```

CI runs both. Docker stages copy this module before dependency download.

All four shipped GoAuthy binary package graphs are checked for absence of JOSE v3.
The unrelated `github.com/ory/x@v0.0.665` go.mod still declares an UNUSED
JOSE v3.0.3 requirement, so `go mod graph`/`go list -m all` can show it. No v3
package is built into the checked binaries; do not call the entire module graph
v3-free. Image/binary scanner results must independently verify the actual
artifact and official v4.1.6 identification. A blank local Fosite version is not
proof of a JOSE fix; retained provenance and the actual JOSE dependency matter.

Remove this local replacement when an official compatible Fosite release uses a
fixed JOSE and passes the same parser, client-authentication, OAuth, JWT/JWKS,
DPoP and full nested normal/race controls. Delete this copy only after verifying
upstream behavior and image scanner evidence; do not blind-replace JOSE majors.
