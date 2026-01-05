# Parte 2: MCP Avanzado - Bases de Datos y Recursos

En esta parte, llevaremos nuestro servidor MCP al siguiente nivel. En lugar de funciones simples, interactuaremos con una base de datos SQLite persistente y exploraremos dos conceptos clave de MCP: **Resources** (Recursos) y **Prompts**.

## 1. Preparando la Base de Datos

Primero, necesitamos un script para inicializar nuestra base de datos SQLite. Esto simulará una base de datos de una comunidad de chat.

Crea el archivo `src/init_sqlite.py`:

**`src/init_sqlite.py`**
```python
import sqlite3
from pathlib import Path

# Definimos la ruta de la base de datos en la carpeta 'data'
DB_PATH = Path(__file__).resolve().parent.parent / "data" / "community.db"
DB_PATH.parent.mkdir(parents=True, exist_ok=True)

def init_db():
    conn = sqlite3.connect(DB_PATH.as_posix())
    cur = conn.cursor()
    # Limpiamos y creamos la tabla
    cur.execute("DROP TABLE IF EXISTS chatters;")
    cur.execute("CREATE TABLE chatters (name TEXT, messages INTEGER);")
    # Insertamos datos de prueba
    cur.executemany(
        "INSERT INTO chatters (name, messages) VALUES (?, ?);",
        [("Alice", 120), ("Bob", 75), ("Carlos", 50)],
    )
    conn.commit()
    conn.close()
    print(f"SQLite inicializada en {DB_PATH}")

if __name__ == "__main__":
    init_db()
```

Ejecuta este script una vez para crear la base de datos:
```bash
python src/init_sqlite.py
```

## 2. Creando el Servidor de Base de Datos

Ahora crearemos `src/servidor_bd.py`. Este servidor implementará:
1.  **Tools**: Para leer (`get_top_chatters`) y escribir (`add_message`) en la base de datos.
2.  **Resources**: Para exponer la tabla completa como un archivo de texto/CSV que el LLM puede leer directamente.
3.  **Prompts**: Plantillas predefinidas para ayudar al LLM a realizar tareas comunes.

Crea el archivo `src/servidor_bd.py`:

**`src/servidor_bd.py`**
```python
from mcp.server.fastmcp import FastMCP
import sqlite3
from pathlib import Path
from typing import List

# Configuración de la base de datos
DB_PATH = Path(__file__).resolve().parent.parent / "data" / "community.db"

# Inicializar FastMCP
mcp = FastMCP("Servidor BaseDatos")

# --- Herramientas (Tools) ---
# Las herramientas son funciones que el modelo puede EJECUTAR.

@mcp.tool(name="get_top_chatters")
def get_top_chatters(limit: int = 5) -> List[dict]:
    """
    Obtiene la lista de usuarios ordenados por mensajes enviados (descendente).
    Args:
        limit: Número máximo de usuarios a devolver (por defecto 5).
    """
    conn = sqlite3.connect(DB_PATH.as_posix())
    cursor = conn.cursor()
    cursor.execute("SELECT name, messages FROM chatters ORDER BY messages DESC LIMIT ?", (limit,))
    results = cursor.fetchall()
    conn.close()
    return [{"name": n, "messages": m} for (n, m) in results]

@mcp.tool(name="add_message")
def add_message(username: str, count: int = 1) -> str:
    """
    Incrementa el contador de mensajes para un usuario.
    Si el usuario no existe, lo crea.
    """
    conn = sqlite3.connect(DB_PATH.as_posix())
    cursor = conn.cursor()
    
    # Verificar si existe
    cursor.execute("SELECT messages FROM chatters WHERE name = ?", (username,))
    row = cursor.fetchone()
    
    if row:
        new_count = row[0] + count
        cursor.execute("UPDATE chatters SET messages = ? WHERE name = ?", (new_count, username))
    else:
        new_count = count
        cursor.execute("INSERT INTO chatters (name, messages) VALUES (?, ?)", (username, count))
        
    conn.commit()
    conn.close()
    return f"Usuario '{username}' actualizado. Total mensajes: {new_count}"

# --- Recursos (Resources) ---
# Los recursos son datos que el modelo puede LEER, como archivos.
# Aquí exponemos la base de datos como si fuera un archivo CSV virtual.

@mcp.resource("community://chatters")
def get_chatters_resource() -> str:
    """Devuelve todo el contenido de la tabla chatters como texto CSV."""
    conn = sqlite3.connect(DB_PATH.as_posix())
    cursor = conn.cursor()
    cursor.execute("SELECT name, messages FROM chatters")
    results = cursor.fetchall()
    conn.close()
    
    csv_lines = ["name,messages"]
    for name, msgs in results:
        csv_lines.append(f"{name},{msgs}")
    
    return "\n".join(csv_lines)

# --- Prompts ---
# Los prompts son plantillas que ayudan al usuario o al modelo a iniciar una interacción.

@mcp.prompt("summarize_activity")
def summarize_activity_prompt() -> str:
    """Prompt para generar un resumen de la actividad de la comunidad."""
    return "Por favor, usa la herramienta 'get_top_chatters' para obtener los usuarios más activos y genera un resumen motivacional para la comunidad."

if __name__ == "__main__":
    mcp.run()
```

## 3. Actualizando el Cliente

Para probar estas nuevas capacidades, asegúrate de que tu archivo `src/cliente.py` tenga la lógica para manejar el servidor `bd`. Si seguiste la Parte 1, ya deberías tener la estructura, pero asegúrate de que el bloque `elif "servidor_bd.py" in server_script:` esté presente y descomentado en el mapa de scripts.

El bloque relevante en `src/cliente.py` es:

```python
            elif "servidor_bd.py" in server_script:
                print(">> Ejecutando herramienta 'add_message'...")
                await session.call_tool("add_message", {"username": "DemoUser", "count": 5})
                
                print(">> Ejecutando herramienta 'get_top_chatters'...")
                result = await session.call_tool("get_top_chatters", {"limit": 3})
                print(f"Top Chatters: {result.content}")
                
                print(">> Leyendo recurso 'community://chatters'...")
                try:
                    res_content = await session.read_resource("community://chatters")
                    # Mostramos solo los primeros 100 caracteres
                    print(f"Contenido del recurso (primeras lineas):\n{str(res_content.contents[0].text)[:100]}...")
                except Exception as e:
                    print(f"No se pudo leer el recurso: {e}")
```

Y actualiza el `script_map` en la función `main`:

```python
    script_map = {
        "simple": "src/servidor_simple.py",
        "bd": "src/servidor_bd.py",
        "clima": "src/servidor_clima.py"
    }
```

## 4. Probando el Servidor Avanzado

Ejecuta el cliente apuntando al servidor de base de datos:

```bash
python src/cliente.py --server bd
```

Deberías ver cómo el cliente:
1.  Lista las nuevas herramientas, recursos y prompts.
2.  Añade un mensaje para "DemoUser".
3.  Obtiene los usuarios más activos.
4.  Lee el contenido "crudo" de la base de datos a través del recurso `community://chatters`.

En la [Parte 3](TUTORIAL_PART_3_DEPLOYMENT.md), aprenderemos a empaquetar todo esto en Docker y desplegarlo en la nube.
