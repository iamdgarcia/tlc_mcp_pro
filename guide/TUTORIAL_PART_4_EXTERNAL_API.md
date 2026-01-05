# Parte 4: Integración con APIs Externas (Clima)

En esta sección, ampliaremos las capacidades de nuestro servidor MCP para interactuar con el mundo real. Modificaremos el `servidor_clima.py` para que, en lugar de devolver datos simulados, consulte una API pública real.

## 1. La API de Open-Meteo

Usaremos [Open-Meteo](https://open-meteo.com/), una API de clima gratuita que no requiere clave de API (API Key), lo cual es perfecto para aprender.

Necesitaremos una librería para hacer peticiones HTTP. `httpx` es una excelente opción moderna y asíncrona.

Añade `httpx` a tu `requirements.txt`:

```txt
mcp==1.13.1
python-dotenv>=1.0.1
uvicorn>=0.27.0
starlette>=0.36.0
httpx>=0.27.0
```

Instálalo:
```bash
pip install httpx
```

## 2. Implementando el Servidor de Clima Real

Vamos a modificar `src/servidor_clima.py`. La lógica será:
1.  Buscar las coordenadas (latitud/longitud) de la ciudad solicitada.
2.  Consultar el clima actual usando esas coordenadas.

**`src/servidor_clima.py`**
```python
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("Servidor Clima Real")

# NUEVO: Función auxiliar para buscar coordenadas
async def get_coordinates(city: str):
    """Busca lat/lon para una ciudad usando la API de geocoding de Open-Meteo."""
    url = "https://geocoding-api.open-meteo.com/v1/search"
    async with httpx.AsyncClient() as client:
        response = await client.get(url, params={"name": city, "count": 1, "language": "es", "format": "json"})
        data = response.json()
        
    if not data.get("results"):
        return None
        
    return data["results"][0]

# NUEVO: Herramienta actualizada para usar datos reales
@mcp.tool(name="get_weather")
async def get_weather(city: str) -> str:
    """
    Obtiene el clima actual real para una ciudad especificada.
    """
    # 1. Obtener coordenadas
    coords = await get_coordinates(city)
    if not coords:
        return f"No se encontró la ciudad: {city}"
    
    lat, lon = coords["latitude"], coords["longitude"]
    name = coords["name"]
    country = coords.get("country", "")

    # 2. Obtener clima
    weather_url = "https://api.open-meteo.com/v1/forecast"
    params = {
        "latitude": lat,
        "longitude": lon,
        "current": ["temperature_2m", "relative_humidity_2m", "weather_code"],
        "timezone": "auto"
    }
    
    async with httpx.AsyncClient() as client:
        response = await client.get(weather_url, params=params)
        data = response.json()
        
    current = data.get("current", {})
    temp = current.get("temperature_2m")
    humidity = current.get("relative_humidity_2m")
    
    return f"El clima en {name}, {country} es: {temp}°C con {humidity}% de humedad."

if __name__ == "__main__":
    mcp.run()
```

## 3. Probando el Clima Real

Actualiza tu `src/cliente.py` si es necesario para asegurarte de que puede llamar a `servidor_clima.py` (ya lo configuramos en la Parte 1).

Ejecuta:
```bash
python src/cliente.py --server clima
```

Deberías ver una respuesta real basada en la ciudad que solicites (puedes modificar el cliente para pedir otra ciudad o cambiar el código hardcodeado).

¡Ahora tu agente de IA tiene acceso a datos del mundo real!

## Apéndice: Código Final de `cliente.py` (Stdio)

Si necesitas el código completo del cliente local (`src/cliente.py`) que hemos ido construyendo en las partes anteriores, aquí lo tienes:

**`src/cliente.py`**
```python
import asyncio
import argparse
import sys
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run_client(server_script: str):
    # Configurar parámetros del servidor (ejecución local stdio)
    server_params = StdioServerParameters(
        command="python", 
        args=[server_script],
        env=None
    )

    print(f"Conectando a {server_script}...")

    async with stdio_client(server_params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            
            # Listar herramientas
            tools = await session.list_tools()
            print("\n=== Herramientas Disponibles ===")
            for t in tools.tools:
                print(f"- {t.name}: {t.description}")
            
            # Listar recursos
            resources = await session.list_resources()
            if resources.resources:
                print("\n=== Recursos Disponibles ===")
                for r in resources.resources:
                    print(f"- {r.uri}: {r.name}")

            # Listar prompts
            prompts = await session.list_prompts()
            if prompts.prompts:
                print("\n=== Prompts Disponibles ===")
                for p in prompts.prompts:
                    print(f"- {p.name}: {p.description}")
            
            print("===============================\n")

            # Lógica específica para demostración según el servidor
            if "servidor_simple.py" in server_script:
                print(">> Ejecutando herramienta 'sumar'...")
                result = await session.call_tool("sumar", {"a": 10, "b": 20})
                print(f"Resultado: {result.content}")

            elif "servidor_bd.py" in server_script:
                print(">> Ejecutando herramienta 'add_message'...")
                await session.call_tool("add_message", {"username": "DemoUser", "count": 5})
                
                print(">> Ejecutando herramienta 'get_top_chatters'...")
                result = await session.call_tool("get_top_chatters", {"limit": 3})
                print(f"Top Chatters: {result.content}")
                
                print(">> Leyendo recurso 'community://chatters'...")
                try:
                    res_content = await session.read_resource("community://chatters")
                    print(f"Contenido del recurso (primeras lineas):\n{str(res_content.contents[0].text)[:100]}...")
                except Exception as e:
                    print(f"No se pudo leer el recurso: {e}")

            elif "servidor_clima.py" in server_script:
                print(">> Ejecutando herramienta 'get_weather' para Madrid...")
                result = await session.call_tool("get_weather", {"city": "Madrid"})
                print(f"Clima en Madrid: {result.content}")

def main():
    parser = argparse.ArgumentParser(description="Cliente MCP de prueba")
    parser.add_argument("--server", choices=["simple", "bd", "clima"], default="simple", help="El servidor a ejecutar")
    args = parser.parse_args()

    script_map = {
        "simple": "src/servidor_simple.py",
        "bd": "src/servidor_bd.py",
        "clima": "src/servidor_clima.py"
    }

    server_script = script_map[args.server]
    
    try:
        asyncio.run(run_client(server_script))
    except KeyboardInterrupt:
        print("\nCliente detenido.")
    except Exception as e:
        print(f"\nError: {e}")

if __name__ == "__main__":
    main()
```

## Apéndice: Código Final de `cliente_sse.py` (SSE)

Aquí tienes el código completo para el cliente que soporta Server-Sent Events, necesario para conectar con el servidor Docker o Cloud:

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