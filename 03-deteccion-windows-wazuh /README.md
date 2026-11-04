# Detección de actividad sospechosa en Windows con Wazuh

> **Prácticas en laboratorio propio.** Las acciones "maliciosas" las provoqué yo mismo en una máquina virtual aislada para comprobar cómo se detectan desde un SIEM. Las capturas son reales de mi laboratorio.

| Campo | Valor |
|---|---|
| **Herramientas** | Wazuh 4.14 (SIEM), agente de Wazuh, registro de seguridad de Windows |
| **Equipo vigilado** | Windows 11 Enterprise (agente `PC-ANTONIO`, equipo `DESKTOP-P8ERETT`) |
| **Eventos analizados** | 4625, 4624, 4720, 4732 |
| **Analista** | SOC nivel 1 |

---

## 1. Objetivo y entorno

Comprobar cómo se ven en un SIEM dos situaciones habituales en un SOC: varios intentos de contraseña fallidos y la creación de un administrador no autorizado, e investigarlas como lo haría un analista de nivel 1.

Laboratorio montado en VirtualBox, en una red aislada:

| Máquina | Sistema | Función |
|---|---|---|
| `soc-wazuh` | Ubuntu Server 24.04 | Wazuh (servidor, indexador y panel) |
| `PC-ANTONIO` | Windows 11 Enterprise | Equipo de un empleado, con el agente de Wazuh |

Durante el montaje resolví problemas de red (servidor DHCP de la red NAT), de espacio en disco, de paquetes a medio instalar y de rendimiento de la virtualización por Hyper-V.

---

## 2. Práctica 1: inicios de sesión fallidos

### Qué hice

Bloqueé la sesión de Windows, introduje mal la contraseña 6 veces seguidas y después entré con la correcta.

### Qué se vio en Wazuh

![Eventos de inicio de sesión fallido y correcto en Wazuh](practica1-eventos.png)

- **15:41:25 – 15:41:35:** seis alertas *Logon Failure - Unknown user or bad password* (regla 60122, nivel 5), una cada 2 segundos aproximadamente.
- **15:43:52:** *Windows Workstation Logon Success* (regla 60118): el inicio de sesión correcto.

### Campos clave

![Campos eventID, targetUserName y logonType del evento](practica1-campos.png)

| Campo | Valor | Significado |
|---|---|---|
| `eventID` | 4625 | Inicio de sesión fallido |
| `targetUserName` | PC-ANTONIO | Cuenta con la que se intentó entrar |
| `logonType` | 2 | Inicio interactivo: alguien delante del propio equipo |

El tipo de inicio de sesión cambia la investigación: los mismos fallos con `logonType` 10 (escritorio remoto) desde una IP externa serían mucho más preocupantes.

No saltó la alerta de *Multiple Windows Logon Failures* porque 6 fallos no alcanzan el umbral de la regla. Los atacantes conocen estos umbrales y prueban contraseñas despacio para no llegar a ellos.

### Nota de cierre

> **Alerta:** Logon Failure - Unknown user or bad password (regla 60122, nivel 5).
> **Equipo/usuario:** equipo DESKTOP-P8ERETT (agente PC-ANTONIO), usuario PC-ANTONIO.
> **Qué se observó:** 6 inicios de sesión fallidos (evento 4625) el 02/11/2026 entre las 15:41:25 y las 15:41:35, de tipo 2 (presencial). A las 15:43 el usuario inició sesión correctamente desde el mismo equipo (evento 4624).
> **Verificación:** se contactó con el usuario, que confirmó que se había equivocado al introducir la contraseña.
> **Clasificación:** falso positivo (actividad legítima).
> **Acción:** cerrado sin más acciones.

---

## 3. Práctica 2: administrador no autorizado

### Qué hice

Simulé la persistencia de un atacante: creé una cuenta con un nombre que parece legítimo y la añadí al grupo de Administradores.

```
net user soporte2 <contraseña> /add
net localgroup Administradores soporte2 /add
```

### Qué se vio en Wazuh

![Eventos de creación de cuenta y cambio en el grupo de Administradores](practica2-eventos.png)

- **16:50:46:** *User account enabled or created* (regla 60109, nivel 8): creación de la cuenta (evento 4720).
- **16:51:12:** *Administrators Group Changed* (regla 60154, **nivel 12**): la cuenta se añade a Administradores (evento 4732).
- *Service startup type was changed* (nivel 3) no tiene relación con la práctica: es ruido de Windows.

### Campos clave

![Campos subjectUserName y targetUserName del evento de creación de cuenta](practica2-campos.png)

| Campo | Valor | Significado |
|---|---|---|
| `subjectUserName` | PC-ANTONIO | Quién hizo la acción |
| `targetUserName` | soporte2 | Cuenta creada |
| `targetSid` | ...-1002 | Identificador interno de la cuenta nueva |

En el evento 4732, `targetUserName` es el grupo (**Administradores**) y la cuenta añadida se identifica por su SID (`memberSid`), que coincide con el de soporte2.

La alerta tiene nivel 12 porque ser administrador da control total del equipo (desactivar el antivirus, instalar programas, borrar registros) y porque añadir administradores es poco frecuente y casi nunca inofensivo si nadie lo ha pedido.

### Nota de escalado

> **Alerta:** Administrators Group Changed (regla 60154, nivel 12).
> **Equipo/usuario:** equipo DESKTOP-P8ERETT (agente PC-ANTONIO). Cuenta creada: soporte2. Cuenta que la creó: PC-ANTONIO.
> **Qué se observó:** el 03/11/2026 a las 16:50:46 se creó la cuenta soporte2 (evento 4720) y a las 16:51:12 se añadió al grupo local Administradores (evento 4732).
> **Verificación:** se consultó con el departamento de IT y nadie había solicitado ni autorizado la creación de la cuenta.
> **Clasificación:** verdadero positivo: se creó una cuenta con privilegios de administrador sin ninguna solicitud ni autorización.
> **Acción:** se deshabilitó la cuenta soporte2 sin borrarla, para conservar las pruebas. Se escala a N2 para investigar si la cuenta PC-ANTONIO está comprometida: inicios de sesión previos, tipo de acceso y origen.

Al terminar la práctica, deshabilité y borré la cuenta (eventos 4725 y 4726).

---

## 4. Lecciones aprendidas

- El mismo evento puede ser inofensivo o grave según el contexto: el tipo de inicio de sesión y lo que pasa después importan más que el fallo en sí.
- No basta con mirar las alertas que saltan: las reglas tienen umbrales y un atacante puede quedarse por debajo de ellos.
- Una alerta de falso positivo solo se cierra después de confirmarlo con el usuario, y siempre se documenta.
- Ante una cuenta sospechosa, primero se contiene (deshabilitar) y nunca se destruyen pruebas: borrar es una decisión posterior del N2.
