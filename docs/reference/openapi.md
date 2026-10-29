---
hide:
  - toc
---

# OpenAPI

The full REST API, as each edition describes itself. These are the `openapi.json`
files served at `/openapi.json` by **enterprise 0.21.0** and **community 0.13.0**,
unchanged. A running server shows the same thing, interactively, at `/docs`.

[Download enterprise 0.21.0 :material-download:](../assets/openapi/enterprise.json){ .md-button }
[Download community 0.13.0 :material-download:](../assets/openapi/community.json){ .md-button }

=== "Enterprise 0.21.0"

    <div id="oa-enterprise" class="tdb-openapi"></div>

=== "Community 0.13.0"

    <div id="oa-community" class="tdb-openapi"></div>

For MCP, which is JSON-RPC over `POST /v1/mcp`, see the [MCP reference](../api/mcp.md).

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.17.14/swagger-ui.css">
<script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.17.14/swagger-ui-bundle.js"></script>
<script>
  window.addEventListener("load", function () {
    [["oa-enterprise", "enterprise"], ["oa-community", "community"]].forEach(function (pair) {
      SwaggerUIBundle({
        url: "../../assets/openapi/" + pair[1] + ".json",
        dom_id: "#" + pair[0],
        supportedSubmitMethods: [],
        defaultModelsExpandDepth: 0,
        docExpansion: "list",
        deepLinking: false
      });
    });
  });
</script>
