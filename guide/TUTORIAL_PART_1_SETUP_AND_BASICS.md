# Parte 1: Configuración y Fundamentos de MCP

En esta primera parte, configuraremos nuestro entorno de desarrollo y crearemos un servidor MCP (Model Context Protocol) básico, junto con un cliente para probarlo.

## 1. Estructura del Proyecto

Primero, crea una carpeta para tu proyecto y dentro de ella la siguiente estructura:

```
mcp_course/
├── data/
├── src/
│   ├── __init__.py
│   ├── cliente.py
│   └── servidor_simple.py
└── requirements.txt
```

Puedes crear esta estructura rápidamente ejecutando el siguiente comando en tu terminal:

```bash
mkdir -p mcp_course/{data,src}
touch mcp_course/requirements.txt mcp_course/src/__init__.py mcp_course/src/cliente.py mcp_course/src/servidor_simple.py
```

## 2. Instalación de Dependencias

Crea el archivo `requirements.txt` con las librerías necesarias. Usaremos `mcp` para el protocolo y `uvicorn` para el servidor web más adelante.

**`requirements.txt`**
```txt
mcp==1.13.1
python-dotenv>=1.0.1
uvicorn>=0.27.0
starlette>=0.36.0
```

Instala las dependencias en tu entorno virtual:

```bash
pip install -r requirements.txt
```

## 3. Creando el Primer Servidor MCP

Vamos a crear un servidor muy simple que exponga una herramienta básica: una calculadora de sumas. MCP permite exponer funciones de Python como "Herramientas" que los modelos de IA pueden invocar.

Crea el archivo `src/servidor_simple.py`:

**`src/servidor_simple.py`**
```python
from mcp.server.fastmcp import FastMCP

# Inicializamos el servidor con un nombre descriptivo
server = FastMCP("Mi Servidor de Ejemplo")

# El decorador @server.tool registra la función como una herramienta disponible
@server.tool(name="sumar", description="Suma dos números enteros y devuelve el resultado")
def sumar(a: int, b: int) -> int:
    return a + b

if __name__ == "__main__":
    # server.run() inicia el servidor usando stdio (entrada/salida estándar) por defecto
    server.run()
```

## 4. Creando un Cliente de Prueba

Para probar nuestro servidor sin necesidad de una interfaz compleja (como Claude Desktop), crearemos un script en Python que actúe como cliente. Este script lanzará el servidor como un subproceso y se comunicará con él mediante `stdio`.

Crea el archivo `src/cliente.py`:

**`src/cliente.py`**
```python
import asyncio
import argparse
import sys
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run_client(server_script: str):
    # Configurar parámetros del servidor (ejecución local stdio)
    # Esto le dice al cliente cómo ejecutar nuestro servidor
    server_params = StdioServerParameters(
        command="python", 
        args=[server_script],
        env=None
    )

    print(f"Conectando a {server_script}...")

    # Establecemos la conexión stdio
    async with stdio_client(server_params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            
            # 1. Listar herramientas disponibles
            tools = await session.list_tools()
            print("\n=== Herramientas Disponibles ===")
            for t in tools.tools:
                print(f"- {t.name}: {t.description}")
            
            # 2. Listar recursos (veremos esto en la Parte 2)
            resources = await session.list_resources()
            if resources.resources:
                print("\n=== Recursos Disponibles ===")
                for r in resources.resources:
                    print(f"- {r.uri}: {r.name}")

            # 3. Listar prompts (veremos esto en la Parte 2)
            prompts = await session.list_prompts()
            if prompts.prompts:
                print("\n=== Prompts Disponibles ===")
                for p in prompts.prompts:
                    print(f"- {p.name}: {p.description}")
            
            print("===============================\n")

            # Lógica de prueba específica para nuestro servidor simple
            if "servidor_simple.py" in server_script:
                print(">> Ejecutando herramienta 'sumar'...")
                # Llamamos a la herramienta 'sumar' con argumentos
                result = await session.call_tool("sumar", {"a": 10, "b": 20})
                print(f"Resultado: {result.content}")

            # (Aquí añadiremos más lógica para otros servidores en el futuro)

def main():
    parser = argparse.ArgumentParser(description="Cliente MCP de prueba")
    parser.add_argument("--server", choices=["simple", "bd", "clima"], default="simple", help="El servidor a ejecutar")
    args = parser.parse_args()

    script_map = {
        "simple": "src/servidor_simple.py",
        # "bd": "src/servidor_bd.py",   # Descomentaremos esto en la Parte 2
        # "clima": "src/servidor_clima.py"
    }

    # Manejo básico de errores si el servidor no está en el mapa aún
    if args.server not in script_map:
        print(f"Servidor '{args.server}' no configurado aún.")
        return

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

## 5. Ejecutando la Prueba

Ahora, ejecuta tu cliente desde la terminal:

```bash
python src/cliente.py --server simple
```

Deberías ver una salida similar a esta:

```text
Conectando a src/servidor_simple.py...

=== Herramientas Disponibles ===
- sumar: Suma dos números enteros y devuelve el resultado
===============================

>> Ejecutando herramienta 'sumar'...
Resultado: [TextContent(type='text', text='30')]
```

¡Felicidades! Has creado tu primer servidor MCP y un cliente capaz de descubrir sus herramientas y ejecutarlas.

En la [Parte 2](TUTORIAL_PART_2_ADVANCED_MCP.md), añadiremos complejidad real con bases de datos y recursos.
