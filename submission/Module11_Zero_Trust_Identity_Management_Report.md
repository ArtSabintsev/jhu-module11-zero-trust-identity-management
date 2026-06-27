# JHU WebSec Module 11: Zero Trust and Identity Management

## 1. Environment Summary

This lab runs Ghost CMS behind Nginx Proxy Manager, with Authelia in front of the admin route. The tested runtime used Nginx Proxy Manager `2.15.1`, Authelia `4.39.20`, Ghost `6.47.0`, and MySQL `8.0.46`. NPM terminates self-signed TLS for `blog.home.local`, `admin.home.local`, and `auth.home.local`, then sends traffic to Ghost or Authelia on the internal Docker network. Before the route restriction, `https://blog.home.local/ghost/` exposed the Ghost admin surface on the public blog hostname. The fix keeps the blog public, but pushes admin access to `admin.home.local`, where Authelia enforces a `two_factor` policy.

## 2. Initial Exposure and Trust Assumptions

The asset being protected is Ghost's administrative URL hierarchy at `/ghost/`. In the initial state, `https://blog.home.local/ghost/` returned `HTTP/2 200` and served Ghost admin application content. That is the bad state: a public hostname exposed a management surface. The weak assumption was that the blog hostname only carried public content, but Ghost puts admin under the same application unless the proxy blocks or reroutes it. Zero Trust breaks that assumption. Being able to reach the blog should not mean being able to reach the CMS administration path.

## 3. Identity and Access Design

The trust boundary sits at Nginx Proxy Manager. Client requests hit NPM first. NPM serves public blog traffic, redirects blocked `/ghost` traffic to the protected admin hostname, or sends an `auth_request` subrequest to Authelia before Ghost ever sees the request. Authelia uses a file-backed user database with Argon2 password hashing, a session cookie scoped to `home.local`, and `default_policy: deny`.

The Authelia rules are domain-scoped: `auth.home.local` is bypassed so the portal can load, `blog.home.local` is bypassed for public content, and `admin.home.local` is set to `two_factor`. The admin reverse-proxy config calls Authelia's verify endpoint before allowing traffic to Ghost. If Authelia returns unauthorized, NPM redirects the browser to `https://auth.home.local/?rd=...`. This gives the admin route an identity gate, but it is still coarse. Authelia protects the admin hostname; NPM path rules decide what remains public on the blog hostname.

## 4. Evidence and Observed Behavior

| Evidence | Observed behavior | What it proves |
| --- | --- | --- |
| `submission/evidence/01-baseline-and-admin-redirect.log` | `GET https://blog.home.local/` returned `HTTP/2 200`. | The intended public route remained accessible. |
| `submission/evidence/01-baseline-and-admin-redirect.log` | Before `ghostBlock.config`, `GET https://blog.home.local/ghost/` returned `HTTP/2 200` and the HTML contained Ghost admin signals. | The administrative surface was exposed on the public blog hostname before the added restriction. |
| `submission/evidence/01-baseline-and-admin-redirect.log` | Unauthenticated `GET https://admin.home.local/ghost/` returned `HTTP/2 302` with `Location: https://auth.home.local/?rd=https://admin.home.local/ghost/`. | The protected admin hostname denied anonymous access and redirected to the identity portal. |
| `submission/evidence/02-after-ghost-block.log` | After applying `ghostBlock.config`, `GET https://blog.home.local/ghost/` returned `HTTP/2 301` to `https://admin.home.local/ghost/`. | The `/ghost` path behavior changed from public exposure to protected-host redirection. |
| `submission/evidence/02-after-ghost-block.log` | After the restriction, `GET https://blog.home.local/` still returned `HTTP/2 200`. | The control was targeted: it blocked the admin path without breaking the public blog. |
| `submission/evidence/03-authelia-identity-evidence.log` | First-factor login returned `HTTP/2 200`, Authelia generated a local one-time identity-verification code, and submitting the redacted code returned `HTTP/2 200`. | Identity verification occurred and was accepted; the one-time code and session cookie were redacted. |
| `submission/evidence/03-authelia-identity-evidence.log` | After identity verification, the TOTP registration endpoint returned `HTTP/2 200` with registration parameters. | The identity-verification step allowed access to security-setting registration flow. |
| `submission/evidence/03-authelia-identity-evidence.log` | Authelia logged anonymous access to `admin.home.local/ghost/` as unauthorized and logged that the admin route requires 2FA. | The identity layer was making authorization decisions before Ghost received protected admin traffic. |

This evidence demonstrates first-factor authentication, a successful one-time identity-verification step, and the enforced 2FA gate. It does not claim completed TOTP enrollment or an authenticated Ghost admin session.

## 5. AAA and CI4A Analysis

Authentication is stronger because admin access now goes through Authelia instead of relying on Ghost's path being obscure or ignored. Authorization is stronger because `admin.home.local` requires `two_factor` while `blog.home.local` stays public, although the policy is still domain-based rather than role- or path-specific inside Ghost. Accounting is better because Authelia logs denied anonymous admin attempts and the first-factor/identity-verification flow, but those logs are local container logs, not centralized audit records. Confidentiality is better because the admin UI is no longer exposed on the public blog path and TLS terminates at NPM, but backend container traffic is still trusted inside Docker. Integrity is better because admin modification paths are funneled through a controlled hostname before reaching Ghost, but the whole design still depends on correct NPM `location` rules. Accountability or assurance improves through per-user identity verification, but it remains limited by a local file-backed identity store and a single lab admin account.

## 6. Residual Risk and Improvement

First, `ghostBlock.config` allows the entire `/ghost/api/` prefix on `blog.home.local`; the evidence shows `/ghost/api/admin/site/` remained network-reachable with `HTTP/2 200`. Ghost still applies its own application-level authentication, but the network-layer identity policy is bypassed for the admin API namespace. The fix is to narrow the public exception to the content API, for example `/ghost/api/content/`, and redirect or protect `/ghost/api/admin/` through `admin.home.local`.

Second, this is still a local proxy-centered design, not a complete enterprise Zero Trust architecture. The identity data is local, there is no device posture signal, there is no continuous session reevaluation, and accounting is limited to local logs. A production version should use per-user accounts from a central IdP, group-based authorization, shorter and risk-aware sessions, device posture checks, and centralized audit logging.

## 7. Focused Analysis Question

Question: Where is the trust boundary in your design?

The trust boundary is the client-to-Nginx Proxy Manager boundary. NPM is the policy enforcement point: it terminates TLS, selects the host and path rule, asks Authelia whether protected admin traffic is authorized, and only then proxies allowed requests to Ghost. Everything behind NPM on the Docker network is treated as a more trusted internal zone. That is why proxy configuration correctness matters. If the NPM `location` rules are wrong, the identity layer can be bypassed even when Authelia itself is configured correctly.
