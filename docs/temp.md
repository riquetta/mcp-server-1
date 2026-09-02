Akamai fits cleanly here — Zuplo has documented Managed Dedicated on Akamai Cloud, and it's a real supported topology rather than something improvised.

``` mermaid
flowchart TB
    Claude["Claude Desktop<br/>(any user, no VPN)"]
    subgraph AK["Akamai Edge"]
        WAF["WAF / Bot Manager<br/>DDoS / rate control<br/>TLS termination<br/>Geo (Edgescape)"]
    end
    subgraph DED["Zuplo Managed Dedicated"]
        GW["Zuplo Gateway<br/>Entra OAuth · rate limit<br/>capability filter · MCP proxy"]
        MCP["MCP Server<br/>(mcpServerHandler)"]
    end
    subgraph CORP["Your Private Network"]
        API["Internal API"]
    end
    Entra["Microsoft Entra"]

    Claude -->|"mcp.yourcompany.com"| WAF
    WAF -->|"origin: Zuplo URL"| GW
    GW --> MCP
    MCP -->|"PrivateLink / peering"| API
    Entra -.->|"token validation"| GW
    Claude -.->|"OAuth login (browser)"| Entra

```

Division of labor: Akamai handles network-layer threats (DDoS, bots, volumetric abuse, geo-blocking, TLS) — things that should never reach your gateway. Zuplo handles identity-layer control (who you are, which tools you may call, per-user rate limits) — things Akamai can't decide because they require understanding Entra tokens and MCP semantics.

Config that matters for your setup specifically:

Two CDN properties — one for the gateway, one for the Dev Portal. Documented separately.
Allowed Methods: enable POST and OPTIONS. Non-negotiable — MCP is JSON-RPC over POST, and OAuth preflight needs OPTIONS. This is the single most likely thing to silently break your setup.
Host header rewrite — a Set Variable behavior with a regex substitution, so the correct host reaches the Zuplo origin. Documented with exact regex.
Don't cache /mcp/* or /.well-known/*. Tool calls are dynamic POSTs and OAuth discovery must stay fresh. Cache the Dev Portal, not the gateway.
Lock the gateway to Akamai with a custom-code-inbound IP-restriction policy using your account's Akamai ranges — but the docs are explicit that Akamai is multi-tenant, so IP allowlisting alone isn't sufficient. Add a secret header injected at the edge and validated in a Zuplo policy.
The risk I keep flagging, now more concretely: Bot Manager and JS challenges are designed to interrogate browsers. Claude Desktop's OAuth flow involves a browser redirect plus a non-browser PKCE token exchange from the app itself. Excluding /.well-known/*, /__zuplo/oauth/*, and /mcp/* from bot challenges is where I'd start.
Source: Setting up Akamai CDNs
Cheapest way to de-risk: put an Akamai property in front of your current .zuplo.app gateway and reconnect Claude Desktop. You'd validate the WAF/OAuth interaction now, before committing to the Dedicated migration.

https://zuplo.com/docs/dedicated/akamai/cdn?_gl=1*it71oe*_gcl_au*MTg2NDExNDM5LjE3ODgyODkyMDAuMTcyOTk0NjQzNi4xNzg4Mjg5MjU3LjE3ODgyOTc5MTEuOTQ0NzU5MDQwLjE3ODgyODkyNTcuMTc4ODI5NzkxMQ..*_ga*NzM0MzY0OTg1LjE3ODgyODkyMDA.*_ga_FJ4E4W746T*czE3ODgzMTg0MDckbzIkZzAkdDE3ODgzMTg0MDckajYwJGwwJGgxMzU5MzY2NDQ.*_fplc*MHBCJTJCb3hDcFd1YjJTSTdQR2ZrQ3ElMkJIazFGYXVSZ3llWWluMDFDa2JBUDlEOGF1UVBWQzRFQzJ3NGx6OWFXV0g2dTBmMHRIblU4MTkzbG83MndLY1duSTdpOHBSSGJvZ1diQ2JvV01aZzc3QWhtYzdpRjIzTUIzaXpwZlZiZyUzRCUzRA..
