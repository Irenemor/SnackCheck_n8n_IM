# Casos de prueba — SnackCheck_n8n_IM

Pruebas realizadas sobre la versión final a entregar del proyecto SnackCheck_n8n_IM

## TEST CASES: FUNCIONAL/INTEGRACIÓN/RENDIMIENTO
| TC | PRUEBA | ENTRADA | SALIDA | RESULTADO |
| --- | --- | --- | --- | --- |
| TC-001 | Producto Válido | 3068320120256 | Éxito Webhook > healthy : 🟢 | OK |
| TC-002 | Producto Válido | 3017620422003 | Éxito Webhook > unhealthy : 🔴 | OK |
| TC-003 | Tiempo de respuesta | 3068320120256 | Éxito Webhook > segundos | OK |



## TEST CASES: ERROR
| TC | PRUEBA | ENTRADA | SALIDA | RESULTADO |
| --- | --- | --- | --- | --- |
| TC-004 | Producto inválido | 30683201XXXXXX | Error 400: El código no está presente o no es numérico | OK |
| TC-005 | Lectura sin código | "    " | Error 400: El código no está presente o no es numérico | OK |
| TC-006 | Vacío |  {    }  | Error 200: Vacío/ sin código de barras | OK |

Pruebas realizadas el 09/09/2026
