# Informe de incidente: compromiso de cuenta de Microsoft 365 por phishing

> **Caso de práctica simulado.** Empresa, usuario, direcciones IP y dominios son ficticios. Las IPs pertenecen a rangos reservados para documentación (RFC 5737) y los indicadores están desactivados (`[.]`, `hxxp`) para evitar clics accidentales.

| Campo | Valor |
|---|---|
| **Tipo de incidente** | Robo de credenciales por phishing / acceso no autorizado |
| **Severidad** | Alta |
| **Activo afectado** | Cuenta de Microsoft 365 del usuario X |
| **Estado** | Escalado a nivel 2 |
| **Analista** | SOC nivel 1 |

---

## 1. Resumen

El 22/10 el usuario X recibió un correo de phishing del remitente `soporte@micros0ft-office365[.]example` con el asunto *"URGENTE: su buzón será desactivado en 24 horas"*. El usuario accedió al enlace e introdujo sus credenciales en una web falsa, con lo que el atacante obtuvo su usuario y contraseña y accedió a la cuenta desde dos IPs extranjeras.

**El atacante podría seguir dentro de nuestro sistema**, ya que no consta que se hayan revocado las sesiones, y podría acceder a información delicada.

## 2. Cronología

| Fecha y hora | Evento |
|---|---|
| 22/10 08:58 | El usuario inicia sesión con normalidad desde `198.51.100.24` (España). |
| 22/10 14:12 | El usuario inicia sesión con normalidad desde `198.51.100.24` (España). |
| 22/10 (hora desconocida) | Llega el correo del atacante con un enlace a una web falsa. |
| 22/10 (hora desconocida) | El usuario introduce sus credenciales en la web falsa. |
| 22/10 23:47 | El atacante intenta acceder desde `203.0.113.17` (Rusia). Inicio de sesión fallido. |
| 22/10 23:48 | El atacante accede a la cuenta desde `203.0.113.17` (Rusia). Inicio de sesión correcto. |
| 23/10 03:15 | El atacante accede a la cuenta desde `192.0.2.9` (Países Bajos), posiblemente mediante VPN. |
| 23/10 09:02 | El usuario inicia sesión desde `198.51.100.24` (España). |

Acceso no autorizado confirmado durante **al menos 3 h 27 min** (23:48 – 03:15).

## 3. Indicadores de compromiso (IoC)

| Tipo | Valor |
|---|---|
| Remitente | `soporte@micros0ft-office365[.]example` |
| URL de phishing | `hxxp://office365-verificacion[.]login-secure[.]example/auth` |
| Dominio de phishing | `login-secure[.]example` |
| IP | `203.0.113.17` (Rusia) |
| IP | `192.0.2.9` (Países Bajos) |

## 4. Impacto

Con las credenciales del usuario, el atacante puede:

- Leer y descargar toda la información de la cuenta de correo electrónico del usuario.
- Enviar correos haciéndose pasar por el usuario a compañeros de la empresa o a clientes.
- Crear reglas de reenvío para recibir automáticamente una copia de los correos del usuario, incluso después de que se cambie la contraseña.
- Acceder a otras aplicaciones vinculadas a la cuenta: OneDrive, SharePoint o Teams.

## 5. Acciones recomendadas

1. Cambiar la contraseña, revocar todas las sesiones y activar el doble factor en la cuenta.
2. Bloquear el dominio, las IPs y el remitente.
3. Verificar si el mismo correo ha llegado a otros usuarios.
4. Investigar las reglas del buzón, la carpeta de enviados y los registros de auditoría para ver qué hizo el atacante o si guardó cambios.
5. Comunicar el aviso a la plantilla y valorar con el responsable si hay que notificar a la AEPD.
