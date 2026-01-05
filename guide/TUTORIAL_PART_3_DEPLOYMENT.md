# Parte 3: Dockerización y Despliegue en DigitalOcean

En esta parte final, transformaremos nuestro servidor MCP local en un servicio web robusto accesible a través de internet. Usaremos **Docker** para empaquetar la aplicación y **DigitalOcean App Platform** para el despliegue.

## 1. Adaptador SSE (Server-Sent Events)

Hasta ahora, hemos usado `stdio` (entrada/salida estándar) para comunicar cliente y servidor. Esto es genial para uso local, pero para la web necesitamos HTTP. MCP usa **SSE (Server-Sent Events)** para enviar mensajes del servidor al cliente.

Necesitamos un punto de entrada que envuelva nuestro servidor `FastMCP` en una aplicación web ASGI (compatible con servidores como Uvicorn).

Crea el archivo `src/sse.py`:

**`src/sse.py`**
```python
from mcp.server.sse import SseServerTransport
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.requests import Request
from starlette.responses import Response
import os
from mcp.server.fastmcp import FastMCP

# Importar los servidores dinámicamente según variable de entorno
SERVER_TYPE = os.getenv("MCP_SERVER_TYPE", "bd")

if SERVER_TYPE == "bd":
    from src.servidor_bd import mcp as mcp_server
elif SERVER_TYPE == "clima":
    from src.servidor_clima import mcp as mcp_server
else:
    from src.servidor_simple import server as mcp_server

# Crear la aplicación Starlette para SSE
if hasattr(mcp_server, 'sse_app'):
    app = mcp_server.sse_app()
elif hasattr(mcp_server, 'create_asgi_app'):
    app = mcp_server.create_asgi_app()
else:
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

    app = Starlette(debug=True, routes=[
        Route("/sse", endpoint=handle_sse),
        Route("/messages", endpoint=handle_messages, methods=["POST"]),
    ])
```

*Nota: Este script selecciona qué servidor ejecutar basándose en la variable de entorno `MCP_SERVER_TYPE`.*

## 2. Dockerización

Docker nos permite empaquetar nuestro código, dependencias y configuración en una imagen portátil.

Crea el archivo `Dockerfile` en la raíz del proyecto (fuera de `src`):

**`Dockerfile`**
```dockerfile
# Usar una imagen base ligera de Python
FROM python:3.11-slim

# Establecer el directorio de trabajo
WORKDIR /app

# Copiar los archivos de requerimientos e instalar dependencias
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copiar el código fuente y datos
COPY src/ src/
COPY data/ data/

# Crear directorio de datos si no existe y asignar permisos
# Importante para que SQLite pueda escribir
RUN mkdir -p data && chmod 777 data

# Inicializar la base de datos durante la construcción
RUN python src/init_sqlite.py

# Exponer el puerto 8080
EXPOSE 8080

# Variable de entorno por defecto
ENV MCP_SERVER_TYPE=bd

# Comando de inicio: Ejecutar el servidor SSE con Uvicorn
CMD ["uvicorn", "src.sse:app", "--host", "0.0.0.0", "--port", "8080"]
```

## 3. Prueba Local con Docker

Antes de subir a la nube, probemos que el contenedor funciona.

1.  **Construir la imagen:**
    ```bash
    docker build -t mcp-server .
    ```

2.  **Ejecutar el contenedor:**
    ```bash
    docker run -p 8080:8080 -e MCP_SERVER_TYPE=bd mcp-server
    ```

Ahora tu servidor está corriendo en `http://localhost:8080/sse`.

### Creando un Cliente SSE

Para conectarnos a nuestro servidor desplegado (o al Docker local), necesitamos un cliente que hable SSE en lugar de stdio.

Crea el archivo `src/cliente_sse.py`:

**`src/cliente_sse.py`**
```python
import asyncio
import argparse
from mcp import ClientSession
from mcp.client.sse import sse_client

async def run_sse_client(url: str):
    print(f"Conectando a {url}...")
    
    # Conexión SSE
    # Nota: sse_client maneja la conexión y el handshake inicial
    async with sse_client(url) as (read_stream, write_stream):
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
                # Nota: Asegúrate de que el servidor corriendo tenga esta herramienta
                result = await session.call_tool("get_top_chatters", {"limit": 1})
                print(f"Resultado: {result.content}")
            except Exception as e:
                print(f"Nota: No se pudo ejecutar la herramienta de prueba (¿quizás estás corriendo otro servidor?): {e}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--url", default="http://localhost:8080/sse", help="URL del servidor SSE")
    args = parser.parse_args()
    
    asyncio.run(run_sse_client(args.url))
```

Ejecútalo localmente:
```bash
python src/cliente_sse.py --url http://localhost:8080/sse
```

## 4. Despliegue en DigitalOcean App Platform

DigitalOcean App Platform es ideal para este tipo de servicios porque maneja HTTPS y escalado automáticamente.

### Pasos para desplegar:

1.  **Sube tu código a GitHub**: Asegúrate de que tu proyecto esté en un repositorio público o privado de GitHub.
2.  **Crea una App en DigitalOcean**:
    *   Ve al panel de control de DigitalOcean -> Apps -> Create App.
    *   Selecciona **GitHub** como fuente y elige tu repositorio.
    *   Deja la configuración de **Source Directory** como `/`.
    *   **Autodetect**: DigitalOcean detectará el `Dockerfile`.
3.  **Configuración de Recursos**:
    *   El plan "Basic" (o incluso el plan gratuito de funciones si fuera serverless, pero aquí usamos Docker) es suficiente.
    *   Asegúrate de que el **HTTP Port** esté configurado en `8080`.
4.  **Variables de Entorno**:
    *   En la sección de configuración, puedes añadir `MCP_SERVER_TYPE` = `bd` (o `clima` o `simple`) para cambiar qué servidor se ejecuta sin modificar el código.
5.  **Deploy**: Haz clic en "Create Resource".

Una vez desplegado, obtendrás una URL pública (ej. `https://mi-mcp-app-xyz.ondigitalocean.app`).

### Conectando tu Cliente al Servidor Remoto

Una vez desplegado, puedes usar el mismo script `src/cliente_sse.py` apuntando a tu nueva URL:

```bash
python src/cliente_sse.py --url https://tu-app.ondigitalocean.app/sse
```



¡Felicidades! Has completado la parte de despliegue.

## Siguientes Pasos

Continúa aprendiendo con los módulos avanzados:

*   **[Parte 4: Integración con APIs Externas (Clima)](TUTORIAL_PART_4_EXTERNAL_API.md)**: Aprende a conectar tu servidor a datos del mundo real.
*   **[Parte 5: Seguridad y Autenticación](TUTORIAL_PART_5_SECURITY.md)**: Protege tu servidor con API Keys.
