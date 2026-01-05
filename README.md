# Curso Práctico de MCP (Model Context Protocol)

Bienvenido a este curso práctico donde aprenderás a construir, probar y desplegar servidores MCP desde cero. Este proyecto ha sido diseñado para guiarte paso a paso desde los conceptos básicos hasta el despliegue en la nube.

## Antes de empezar


## ¡Hola! 👋

¡Estoy aquí para ayudarte a tener el máximo éxito! No dudes en contactarme si puedo ayudarte, ya sea en la plataforma o por correo electrónico. Si deseas que amplíe tu progreso, comparte tus proyectos o publicaciones y estaré encantado de participar.


- **Contact email:** `info@iamdgarcia.com`  
- **LinkedIn:** `https://www.linkedin.com/in/iamdgarcia/`  
- **Substack :** `https://iamdgarcia.substack.com`  

## Contenido del Curso

Este tutorial está dividido en tres partes fundamentales:

### [Parte 1: Configuración y Fundamentos](guide/TUTORIAL_PART_1_SETUP_AND_BASICS.md)
En esta primera parte, estableceremos nuestro entorno de desarrollo y construiremos nuestro primer servidor MCP simple.
*   **Lo que aprenderás:**
    *   Qué es MCP y por qué es útil.
    *   Configuración del entorno Python.
    *   Creación de un servidor básico (`servidor_simple.py`).
    *   Creación de un cliente de prueba (`cliente.py`) para interactuar con tu servidor.

### [Parte 2: MCP Avanzado - Bases de Datos y Recursos](guide/TUTORIAL_PART_2_ADVANCED_MCP.md)
Profundizaremos en las capacidades de MCP integrando una base de datos SQLite real.
*   **Lo que aprenderás:**
    *   Integración de SQLite con MCP.
    *   Implementación de **Tools** (Herramientas) para modificar datos.
    *   Uso de **Resources** (Recursos) para exponer datos a LLMs.
    *   Creación de **Prompts** para guiar la interacción del modelo.

### [Parte 3: Dockerización y Despliegue en DigitalOcean](guide/TUTORIAL_PART_3_DEPLOYMENT.md)
Llevaremos nuestro servidor local a la nube para que pueda ser consumido por cualquier cliente MCP a través de internet.
*   **Lo que aprenderás:**
    *   Adaptación del servidor para SSE (Server-Sent Events).
    *   Creación de un `Dockerfile` optimizado.
    *   Despliegue en DigitalOcean App Platform.

### [Parte 4: Integración con APIs Externas](guide/TUTORIAL_PART_4_EXTERNAL_API.md)
Conectaremos nuestro servidor al mundo real usando APIs públicas.
*   **Lo que aprenderás:**
    *   Uso de `httpx` para peticiones asíncronas.
    *   Consumo de la API de Open-Meteo para datos climáticos reales.

### [Parte 5: Seguridad y Autenticación](guide/TUTORIAL_PART_5_SECURITY.md)
Protegeremos nuestro servidor expuesto a internet.
*   **Lo que aprenderás:**
    *   Implementación de Middleware en Starlette.
    *   Autenticación mediante API Keys.

## Requisitos Previos

*   Python 3.10 o superior.
*   Conocimientos básicos de Python y terminal.
*   Una cuenta de GitHub (para el despliegue).
*   Una cuenta de DigitalOcean (para la Parte 3).

¡Empecemos con la [Parte 1](guide/TUTORIAL_PART_1_SETUP_AND_BASICS.md)!

