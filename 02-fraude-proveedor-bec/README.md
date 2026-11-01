# Informe de incidente: fraude del proveedor (BEC) por cambio de cuenta bancaria

> **Caso de práctica simulado.** Empresa, personas, direcciones IP, dominios e IBAN son ficticios. Las IPs pertenecen a rangos reservados para documentación (RFC 5737) y los dominios usan el sufijo reservado `.example`; los indicadores están desactivados (`[.]`) para evitar clics accidentales.

| Campo | Valor |
|---|---|
| **Tipo de incidente** | Fraude del proveedor / BEC (Business Email Compromise) |
| **Severidad** | Alta |
| **Activo afectado** | Departamento de Contabilidad (pago a proveedores) |
| **Clasificación** | Verdadero positivo |
| **Estado** | Escalado a nivel 2 |
| **Analista** | SOC nivel 1 |

---

## 1. Resumen

El 26/10/2026 a las 09:41, Antonio (Contabilidad) recibió un correo que suplantaba al proveedor Suministros Norte y solicitaba pagar la factura FN-2026-0847 (14.380 €) a una cuenta bancaria nueva. Antonio consultó antes de pagar, se identificó un intento de fraude BEC y no se realizó la transferencia.

## 2. Análisis del correo

### Origen real

El correo se originó en el equipo `192.0.2.77` y se envió a través de `vps-4471[.]hosting-barato[.]example` (`203.0.113.45`), un servidor ajeno al proveedor. No pasó por su servidor legítimo, `mail[.]suministros-norte[.]example` (`198.51.100.10`).

### Autenticación del correo

| Comprobación | Resultado | Interpretación |
|---|---|---|
| **SPF** | fail | El servidor que entregó el correo (`203.0.113.45`) no está autorizado por `suministros-norte[.]example` para enviar correos en su nombre. |
| **DKIM** | none | El correo no lleva ninguna firma digital. Un proveedor que firma sus correos legítimos no mandaría uno sin firmar. Por sí solo no es prueba de nada, pero suma. |
| **DMARC** | fail (política `p=none`) | Ni SPF ni DKIM validan el dominio del From, así que el remitente está suplantado. Como el proveedor tiene la política en `p=none`, nuestro servidor entregó el correo en la bandeja de entrada pese al fallo. |

### Remitentes

| Campo | Dirección |
|---|---|
| From | `facturacion@suministros-norte[.]example` (falsificado) |
| Return-Path | `facturacion@suministros-norte[.]example` (falsificado) |
| Reply-To | `facturacion.suministrosnorte@correo-rapido[.]example` |

El Reply-To es la única diferencia visible, y además está disfrazado: la parte anterior a la `@` imita el nombre del proveedor para que no se note de un vistazo.

Si Antonio hubiera respondido para preguntar si el cambio de cuenta era real, la respuesta le habría llegado al atacante, que le habría confirmado el cambio. Con esto el atacante consigue dos cosas:

- **Neutraliza la comprobación de Antonio:** pregunta, y le confirma el propio estafador.
- **Se queda con la conversación:** el proveedor real nunca se entera de nada.

## 3. Indicadores de compromiso (IoC)

| Tipo | Valor |
|---|---|
| IP del equipo del atacante | `192.0.2.77` |
| Servidor de envío | `vps-4471[.]hosting-barato[.]example` |
| IP del servidor de envío | `203.0.113.45` |
| Dirección de respuesta (Reply-To) | `facturacion.suministrosnorte@correo-rapido[.]example` |
| IBAN fraudulento | `ES00 1234 5678 9012 3456 7890` |

## 4. Impacto

Pérdida potencial de 14.380 €, difícil de recuperar una vez realizada la transferencia.

El atacante conocía el número de una factura real pendiente de pago. Esto indica una posible filtración de información, por ejemplo por el compromiso del buzón del proveedor o de una cuenta de la empresa. Debe investigarse.

## 5. Clasificación y decisión

- **Clasificación:** verdadero positivo.
- **Tipo de ataque:** BEC (Business Email Compromise), también conocido como fraude del proveedor o del cambio de IBAN.
- **Decisión:** escalar a N2 con prioridad alta, porque hay dinero en juego y una posible filtración de información.

## 6. Acciones recomendadas

1. Indicar a Antonio que no pague, no responda y conserve el correo.
2. Verificar con el proveedor por teléfono, a un número ya conocido.
3. Si se hubiera pagado: contactar con el banco de inmediato.
4. Bloquear las IPs y la dirección del Reply-To.
5. No bloquear la dirección del From: `facturacion@suministros-norte[.]example` es la dirección real del proveedor, y bloquearla cortaría sus correos legítimos.
6. Buscar si el correo llegó a otros buzones.
7. Escalar a N2 para investigar cómo conocía el atacante la factura.
8. Avisar a Suministros Norte de la suplantación y recomendarles DMARC `quarantine` o `reject`.
9. Añadir en nuestro filtro un aviso para correos que fallen SPF o DMARC.
10. Norma interna: todo cambio de cuenta bancaria se verifica por teléfono.
