# Parte 5: Seguridad y Autenticación

Exponer un servidor a internet sin seguridad es arriesgado. En esta sección, añadiremos una capa de autenticación simple basada en **API Keys** a nuestro servidor MCP desplegado.

## 1. Estrategia de Seguridad

Implementaremos un middleware en nuestra aplicación `Starlette` (en `src/sse.py`) que intercepte todas las peticiones y verifique la presencia de un encabezado `X-API-Key` válido.

## 2. Modificando el Adaptador SSE

Edita `src/sse.py` para incluir la verificación.

**`src/sse.py`**
```python
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route, Mount
from starlette.requests import Request
from starlette.responses import Response
from starlette.middleware import Middleware
from starlette.middleware.base import BaseHTTPMiddleware
import os
from mcp.server.fastmcp import FastMCP

# --- Configuración ---
SERVER_TYPE = os.getenv("MCP_SERVER_TYPE", "bd")
API_KEY = os.getenv("MCP_API_KEY") # NUEVO: Leemos la clave de las variables de entorno

# --- Selección del Servidor ---
if SERVER_TYPE == "bd":
    from src.servidor_bd import mcp as mcp_server
elif SERVER_TYPE == "clima":
    from src.servidor_clima import mcp as mcp_server
else:
    from src.servidor_simple import server as mcp_server

# --- Middleware de Autenticación ---
# NUEVO: Clase para interceptar peticiones y verificar la clave
class AuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Si no hay API Key configurada, permitimos todo (modo inseguro/desarrollo)
        if not API_KEY:
            return await call_next(request)

        # Verificar el header
        request_key = request.headers.get("X-API-Key")
        if request_key != API_KEY:
            return Response("Unauthorized: Invalid API Key", status_code=401)
            
        return await call_next(request)

# --- Construcción de la App ---

# Obtenemos la app original del servidor MCP
if hasattr(mcp_server, 'sse_app'):
    mcp_app = mcp_server.sse_app()
elif hasattr(mcp_server, 'create_asgi_app'):
    mcp_app = mcp_server.create_asgi_app()
else:
    # Fallback manual si FastMCP no expone la app directamente
    print("FastMCP.sse_app not found. Using fallback SSE implementation.")
    from mcp.server.sse import SseServerTransport
    
    sse = SseServerTransport("/messages")

    async def handle_sse(request: Request):
        async with sse.connect_sse(
            request.scope, request.receive, request._send
        ) as streams:
            # Intentamos acceder al servidor subyacente si es FastMCP
            # FastMCP usa _mcp_server internamente
            server = getattr(mcp_server, "_mcp_server", mcp_server)
            
            await server.run(
                streams[0], streams[1], server.create_initialization_options()
            )

    async def handle_messages(request: Request):
        await sse.handle_post_message(request.scope, request.receive, request._send)

    mcp_app = Starlette(debug=True, routes=[
        Route("/sse", endpoint=handle_sse),
        Route("/messages", endpoint=handle_messages, methods=["POST"]),
    ])

# Envolvemos la app con nuestro middleware de seguridad
# NUEVO: Creamos una nueva app que monta la anterior y aplica el middleware
app = Starlette(
    middleware=[Middleware(AuthMiddleware)],
    routes=[
        Mount("/", app=mcp_app) # Montamos la app de MCP en la raíz
    ]
)
```

## 3. Configuración en Docker y Cloud

Ahora necesitas definir la variable de entorno `MCP_API_KEY`.

### En Docker Local:


```bash
docker build -t mcp-server .
```

```bash
docker run -p 8080:8080 \
  -e MCP_SERVER_TYPE=clima \
  -e MCP_API_KEY=mi-secreto-super-seguro \
  mcp-server
```

### En DigitalOcean:
Ve a la configuración de tu App -> Settings -> Environment Variables y añade:
*   Key: `MCP_API_KEY`
*   Value: `(tu clave secreta)`

## 4. Conectando con Autenticación

Tu cliente ahora debe enviar el encabezado. Aquí tienes el código completo actualizado de `src/cliente_sse.py` para soportar autenticación.

**`src/cliente_sse.py`**
```python
import asyncio
import argparse
from mcp import ClientSession
from mcp.client.sse import sse_client

async def run_sse_client(url: str, api_key: str = None):
    print(f"Conectando a {url}...")
    
    # Preparamos los headers si hay API Key
    headers = {}
    if api_key:
        headers["X-API-Key"] = api_key
        print("Using API Key authentication")

    # Conexión SSE con headers opcionales
    async with sse_client(url, headers=headers) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            
            # Listar herramientas
            tools = await session.list_tools()
            print("\n=== Herramientas Remotas ===")
            for t in tools.tools:
                print(f"- {t.name}: {t.description}")
            
            # Ejemplo de llamada (si es el servidor de BD)
            try:
                print("\n>> Intentando obtener top chatters...")
                result = await session.call_tool("get_top_chatters", {"limit": 1})
                print(f"Resultado: {result.content}")
            except Exception as e:
                print(f"Nota: No se pudo ejecutar la herramienta de prueba: {e}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--url", default="http://localhost:8080/sse", help="URL del servidor SSE")
    parser.add_argument("--api-key", help="API Key para autenticación")
    args = parser.parse_args()
    
    asyncio.run(run_sse_client(args.url, args.api_key))
```

Ejecútalo con la clave:
```bash
python src/cliente_sse.py --url http://localhost:8080/sse --api-key mi-secreto-super-seguro
```

¡Ahora tu servidor MCP es seguro y solo accesible para quienes tengan la llave!
