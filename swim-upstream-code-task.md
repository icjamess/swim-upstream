Commit the swim-upstream MCP server to this repository (icjamess/swim-upstream).

1. Write the four files below at the repo ROOT, contents VERBATIM (no reformatting, renaming, or upgrades). Replace the existing README.md entirely. Add no other files.
2. Verify: `python -m py_compile server.py`; `pip install -r requirements.txt`; start `PORT=10000 python server.py` in the background; POST an MCP initialize request to http://127.0.0.1:10000/mcp with headers `Content-Type: application/json` and `Accept: application/json, text/event-stream`; expect HTTP 200 and serverInfo.name "swim-upstream"; stop the server. requirements.txt must stay exactly `mcp>=1.10,<2`.
3. Commit message: `Initial swim-upstream MCP server for Claude connector`. Push to `main`; if main is refused, push your branch and open a PR into main.
4. Report the commit SHA, branch, the four paths, and verification results. Never claim a step that did not complete. Do not deploy anything or change any other repo or setting.

FILE 1: `server.py`
````python
"""swim-upstream — a fresh remote MCP connector for Claude.

Streamable HTTP, stateless, authless. Serves at /mcp.
Built new; shares nothing with existing connectors.
"""
import os
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(
    "swim-upstream",
    host="0.0.0.0",
    port=int(os.environ.get("PORT", "8000")),
    stateless_http=True,
    json_response=True,
)

MODES = {
    "stem": "sustained headway straight against the gradient",
    "breast": "one continuous push (blocking call)",
    "buffet": "repeated discrete strikes (retry with backoff)",
    "froward": "orientation, not effort: traverse against the edges",
}


@mcp.tool()
def upstream_frame(concept: str, cheap_direction: str, mode: str = "froward") -> dict:
    """Render a concept as motion against its gradient.

    concept: the thing being examined.
    cheap_direction: which way the system naturally flows.
    mode: stem | breast | buffet | froward.
    """
    mode = mode.lower()
    if mode not in MODES:
        return {"error": f"mode must be one of {sorted(MODES)}"}
    return {
        "gradient": f"{concept}: cheap direction is {cheap_direction}",
        "cost": f"moving against '{cheap_direction}' costs attention, latency, and error budget; cost scales with slope",
        "mode": f"{mode} — {MODES[mode]}",
        "termination": "stop when net headway per step falls below epsilon (futility bound)",
        "headway": "measure displacement net of drift; report depth reached, not depth requested",
    }


@mcp.tool()
def headway(slope: float = 0.3, drift: float = 0.1, budget: int = 500,
            effort: float = 1.0, fatigue: float = 0.99, epsilon: float = 1e-3) -> dict:
    """Climb against a gradient until motion ends on its own.

    Each step: net = effort*(1-slope) - drift; effort decays by `fatigue`.
    Halts when net < epsilon or budget (steps) is spent.
    """
    if not 0 <= slope < 1:
        return {"error": "slope must be in [0, 1)"}
    depth, steps, e = 0.0, 0, effort
    reason = "budget exhausted"
    while steps < budget:
        net = e * (1 - slope) - drift
        if net < epsilon:
            reason = "headway below epsilon — edge reached"
            break
        depth += net
        steps += 1
        e *= fatigue
    return {"depth_reached": round(depth, 4), "steps": steps, "stop_reason": reason}


if __name__ == "__main__":
    mcp.run(transport="streamable-http")
````

FILE 2: `Dockerfile`
````dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
ENV PORT=8000
EXPOSE 8000
CMD ["python", "server.py"]
````

FILE 3: `requirements.txt`
````text
mcp>=1.10,<2
````

FILE 4: `README.md`
````markdown
# swim-upstream — new remote MCP connector

Fresh build. Adds a new connector alongside existing ones; edits nothing.

## Tools
- `upstream_frame(concept, cheap_direction, mode)` — gradient / cost / mode / termination / headway
- `headway(slope, drift, budget, ...)` — climbs until motion ends; reports depth reached

## Run locally
    pip install -r requirements.txt
    python server.py          # serves http://localhost:8000/mcp

## Deploy (must be public HTTPS — Claude connects from Anthropic's cloud)
Any container host works (Render, Railway, Fly.io, Cloud Run):
    docker build -t swim-upstream . && docker run -p 8000:8000 swim-upstream
The host must set PORT or expose 8000.

## Add to Claude
Customize → Connectors → **+** → Add custom connector
- Name: `swim-upstream` (distinct from existing "upstream")
- URL: `https://<your-host>/mcp`   ← single slash after host, ends in /mcp
- Advanced settings: leave blank (authless)

Then enable it in the chat's tools menu.
````
