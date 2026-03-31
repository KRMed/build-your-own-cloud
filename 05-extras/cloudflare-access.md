## Protect Internal Apps with Cloudflare Access

**Recommended if using Option A (Cloudflare Tunnel). Skip if using Option B or C.**

Tools like ArgoCD, Grafana, and Prometheus should not be publicly accessible, even though they are reachable via the tunnel. Cloudflare Access adds an authentication layer at Cloudflare's edge - before any request reaches your cluster. A user who is not authenticated sees a login page served by Cloudflare, not your app. Your cluster never receives the unauthenticated request.

> **If you are using Option B (port forwarding or direct public IP):** You do not have Cloudflare Access, but your internal tools are still publicly reachable. Make sure you have changed the ArgoCD default password (see [bootstrap-argocd.md](../03-gitops/bootstrap-argocd.md#access-argocd-for-the-first-time)) and set a strong Grafana admin password (see [cluster-secrets.md](../03-gitops/cluster-secrets.md#monitoring-credentials)). Consider restricting access to internal tool subdomains by IP allowlist in your ingress-nginx configuration, or by using ingress-nginx's built-in [basic auth](https://kubernetes.github.io/ingress-nginx/examples/auth/basic/) as an additional layer in front of tools that do not have strong authentication of their own (Prometheus, Alertmanager).

> **Reference:** [Publish a self-hosted application](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/)

### Create an Access application for each protected service

1. In the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com), navigate to **Access → Applications → Add an application** (if you cannot find this path, use the reference link above; Cloudflare occasionally reorganizes its UI)
2. Select **Self-hosted**
3. Set the **Application domain** to your subdomain (e.g. `argocd.your-domain.com`)
4. Under **Policies**, add a rule:
   - To allow only yourself: **Selector: Emails**, value: your email address
   - To allow a team: **Selector: Emails ending in**, value: `@your-org.com`
   - To use GitHub, Google, or another identity provider: connect one under **Zero Trust → Settings → Authentication** first
5. Save. Repeat for each subdomain that should be protected

Your public-facing apps should **not** have an Access policy - they should be reachable by anyone.

### Available identity providers

Cloudflare Access supports authentication via:

- **Email OTP** - requires no OAuth app setup. You still need to add an allow policy specifying which email addresses or domains are permitted (step 4 above), but no external identity provider configuration is needed. Good for getting started
- **GitHub, Google, Microsoft, Okta, and others** - requires creating an OAuth app in the provider and entering the credentials in Cloudflare
- **One-time PINs**

> **Reference:** [Cloudflare Access - Identity providers](https://developers.cloudflare.com/cloudflare-one/identity/idp-integration/)
