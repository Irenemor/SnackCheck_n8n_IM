# SnackCheck_n8n_IM
Proyecto final n8n: automatización para la verificación de Snacks

## 1. Propósito
Comprar en el supermercado e interpretar las etiquetas nutricionales suele ser confuso y tedioso: tablas con porcentajes técnicos, reclamos publicitarios engañosos y clasificaciones parciales que dificultan saber si un producto es adecuado para el día a día o debe reservarse para ocasiones puntuales.

**SnackCheck** resuelve este problema actuando como ese amigo de confianza con conocimientos de nutrición que te acompaña en la compra:
- Recibe el código de barras escaneado desde la app móvil.
- Consulta datos nutricionales reales, abiertos y contrastados en la base de datos global de **Open Food Facts**.
- Aplica una evaluación dual estricta combinando el **Nutri-Score oficial** con el **semáforo nutricional** (azúcares, grasas saturadas y sal), asegurando que el veredicto final corresponda siempre al peor de ambos para evitar falsos positivos (por ejemplo, bebidas azucaradas diluidas).
- Genera un veredicto estructurado y un párrafo amigable mediante Inteligencia Artificial (Groq), redactado con calidez humana: alentador cuando el alimento es saludable y empático pero firme cuando no lo es.
- Implementa una arquitectura tolerante a fallos: ante códigos ausentes, errores de formato o productos sin datos suficientes, responde de forma didáctica sin generar caídas ni pantallas técnicas intimidantes.

## 2. Cómo Funciona: Descripción general del flujo
El workflow implementado en n8n consta de las siguientes fases secuenciales:

1. **Recepción (Webhook Node):** Escucha las peticiones entrantes tipo `POST` o `GET` enviadas desde la app cliente con el código de barras.
2. **Validación de entrada (IF Nodes):**
   - Comprueba si el parámetro del código de barras existe. Si viene vacío, responde con una guía de bienvenida e instrucciones de uso.
   - Valida que el código sea puramente numérico. Si contiene caracteres inválidos, devuelve un mensaje explicativo.
3. **Consulta de datos externos (HTTP Request Node):** Llama a la API REST pública de **Open Food Facts** mediante el endpoint `https://world.openfoodfacts.org/api/v2/product/{barcode}.json`.
4. **Validación de existencia y datos suficientes (IF Nodes):**
   - Verifica el código de estado del producto (`status === 1`). Si no existe en la base de datos, devuelve un mensaje informativo de "producto no encontrado".
   - Comprueba la disponibilidad de la información nutricional mínima requerida (presencia de `nutriments`). Si los campos clave están en blanco o ausentes, detiene la clasificación para evitar considerar erróneamente el producto como "saludable".
5. **Cálculo de reglas de negocio (Code Node - JavaScript):**
   - Evalúa los niveles individuales del semáforo para azúcares, grasas y sal por cada 100g/ml (bajo, moderado, alto).
   - Calcula la **Puntuación de Preocupación** (conteo de 0 a 3 de nutrientes en nivel alto).
   - Extrae el grado de Nutri-Score (`a` a `e`). Si es `unknown` o no existe, prioriza el cálculo del semáforo.
   - Aplica la **regla del peor veredicto**: el nivel general del producto es el más desfavorable entre el semáforo y el Nutri-Score (`Saludable`, `Moderado`, `No saludable`).
6. **Generación de lenguaje natural (Groq AI Chat Model):**
   - Transfiere las métricas calculadas y el tono asignado (alentador o cauteloso con delicadeza) a un modelo LLM optimizado a través de Groq.
   - El modelo produce una explicación humana y cercana en un párrafo conciso.
7. **Formato y Entrega (Respond to Webhook Node):** Ensambla el payload final estructurado (titular, desglose de semáforo con marcadores visuales, puntuación de preocupación y párrafo del veredicto) y lo devuelve a la app móvil.

## Diagrama de flujo en https://excalidraw.com/
<img width="3510" height="948" alt="image" src="https://github.com/user-attachments/assets/37c893f6-09a0-4c3d-a0ed-715a2af9f78d" />


## 3.  Uso: JSON de entrada/salida de ejemplo
1. Petición estándar (Entrada válida)
Request:
POST /webhook/snackcheck

JSON
{
  "barcode": "8480000142878"
}

2. Respuesta estándar (Salida válida)
📑 Informe del producto Nutella, Ferrero, Yum yum\n\n➡️ El veredicto sobre el producto escaneado es: unhealthy : 🔴\n\n🍎🍔🍩Recomendación: Este producto aporta 56.3 g de carbohidratos, 0.107 g de grasa y 30.9 g de proteína por 100 g. Con un bajo contenido graso y una buena cantidad de proteína, su alto nivel de carbohidratos y la clasificación NutriScore “unhealthy” sugieren un perfil calórico y azucarado elevado. Es adecuado para quienes buscan energía rápida, pero no es recomendable para dietas bajas en carbohidratos o con control de azúcar. 🍎🍔🍩  \n\n📆 Generado el 8 de septiembre de 2026


## 4. Motor de Snackccccheck
El nodo LOGICA - Code in JavaScript es el motor de cálculo del flujo: procesa los datos de Open Food Facts y define el veredicto objetivo antes de redactar con IA mediante 4 pasos clave:

- Semáforo por nutriente: Clasifica azúcar, grasas saturadas y sal, y umbrales de salud pública.
- Puntuación de preocupación: Cuenta cuántos nutrientes salieron en nivel rojo (un valor simple de 0 a 3).
- Equivalencia de Nutri-Score: Mapea la letra oficial (A/B = Saludable, C = Moderado, D/E = No saludable). Si no existe, se ignora.
- Regla del peor veredicto: Cruza el semáforo y el Nutri-Score; para ser Saludable ambos deben coincidir, quedándose siempre con el resultado más estricto si discrepan.
- Veredicto: Mensaje unhealthy:🔴; Mensaje moderate:🟡; Mensaje healthy:🟢.
Deja como salida el veredicto final, los colores de los nutrientes y la directiva de tono lista para el prompt de Groq.


## 5. Manejo de Errores
| Error observado | Causa raíz | Solución implementada en el flujo |
| --- | --- | --- |
| HTTP 200: Vacío o sin barcode | El usuario abrió la app o probó el webhook sin parámetros. | El nodo IF - Validar existencia Barcode redirige al branch falso devolviendo un mensaje amigable con instrucciones claras.|
| HTTP 400: Código con letras o símbolos | símbolos	Lectura parcial o ruido en el escáner del cliente. | El nodo IF - Validar Barcode filtra mediante regex ^\d+$ y notifica que el código debe contener solo dígitos. |
| HTTP 404: Producto no catalogado (status: 0) | El código no está registrado en Open Food Facts. | El nodo IF - Validar existencia de Producto devuelve un mensaje cordial |
| HTTP 400:Nutrientes vacíos o nulos | El producto existe pero la ficha de nutrientes está sin rellenar. | El nodo IF - Validar presencia Nutrientes corta la evaluación. Nunca califica como saludable un producto sin datos. Informa que faltan datos para emitir un veredicto honesto. |


## 5. Dependencias
- n8n Cloud: https://irenemoor.app.n8n.cloud/
- Open Food Facts API v2
- Disparador: https://hoppscotch.io/
- Groq Chat Model con el modelo openai/gpt-oss-20b
- Open Food Facts no requiere credencial en este workflow. La única credencial que necesita este proyecto es tu clave de Groq para el paso de IA.
