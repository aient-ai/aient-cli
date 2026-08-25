# Security

Report suspected vulnerabilities privately to
[security@aient.ai](mailto:security@aient.ai). Do not include credentials,
customer data, or exploit details in a public GitHub issue.

## Release identity

`aient-ai/aient-cli` is a binary-only public distribution repository. Aient
builds releases from the private `haf/glimt` repository with GitHub Actions,
then mirrors the resulting bytes without rebuilding or re-signing them.

The expected Sigstore identity for a release tag is therefore:

```text
https://github.com/haf/glimt/.github/workflows/aient-cli-release.yml@refs/tags/aient-cli-v<VERSION>
```

with OIDC issuer:

```text
https://token.actions.githubusercontent.com
```

Verification must use that exact identity. Do not replace it with a permissive
regular expression. The distinct SLSA provenance builder identity is:

```text
https://github.com/haf/glimt/.github/workflows/aient-cli-provenance.yml@refs/tags/v1.0.1
```

The builder tag is protected against creation, update, and deletion with no
bypass actors. Its exact GitHub workflow signer digest is:

```text
cdab76e75fef610b59a7f528a6dba359e624d6af
```

Use a current GitHub CLI's `gh attestation verify` with the release bundle. Pin
the exact certificate identity, OIDC issuer, signer digest, source tag, and
peeled source commit; for 0.11.3 that source commit is
`b70bd1e2f3caa673f7203b4eb4ad30ba1bf95a37`. The verifier owns the Sigstore
signature, Rekor inclusion, certificate, artifact digest, and source checks.
The mirror additionally requires the verified statement to be in-toto v0.1
with SLSA provenance v0.2, exactly 18 release subjects, the exact builder and
generic build type, and the exact tagged config source and release-workflow
entrypoint. Predicate parsing never substitutes for cryptographic verification.

The private repository identity is public in the transparency and provenance
records; private source content is not.

GitHub's generated source archives contain only this repository's public
documentation and are not part of the verified Aient CLI release.
