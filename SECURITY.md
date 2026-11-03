# Security policy

## Reporting a vulnerability

Please do not open a public issue for a security problem in this Action or in the engine image it runs (`ghcr.io/iwebai/railo`).

Email [zarif.latif@railo.dev](mailto:zarif.latif@railo.dev) with a subject line starting `SECURITY:`. It is the same contact railo.dev publishes in [`/.well-known/security.txt`](https://www.railo.dev/.well-known/security.txt). Include:

- the version you run: the tag or commit in your `uses:` line, or the image digest from the job log;
- what you found, and how to reproduce it;
- whether you intend to publish, and when.

We aim to acknowledge within 5 business days and agree a timeline for a fix and for disclosure with you.

## Supported versions

The latest release behind the moving major tag (`v0` for Fix Gate, `v2` for Railo) gets fixes. Older tags get them by moving to the latest.

## Safe harbor

We support good-faith research that follows this policy and does not degrade user services.
