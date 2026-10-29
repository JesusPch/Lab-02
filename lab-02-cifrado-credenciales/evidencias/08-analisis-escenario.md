### Análisis de Riesgos y Estrategia de Mitigación en Seguridad de Base de Datos

#### 1. Principales riesgos identificados
* **Exposición masiva de credenciales:** Las contraseñas almacenadas en texto claro permiten la suplantación inmediata de identidad y ataques de reutilización de credenciales (*credential stuffing*) en otros sistemas.
* **Fuga de datos personales y financieros:** La falta de cifrado en el RFC e información financiera expone información confidencial, aumentando el riesgo de fraude e incumplimiento directo de normativas legales de privacidad (como la LFPDPPP o GDPR).
* **Compromiso total del esquema criptográfico:** La presencia de claves criptográficas en archivos SQL provoca su fuga a través de registros del servidor (*logs*), historial de comandos (`.psql_history`), monitoreo en tiempo real (`pg_stat_activity`) y repositorios de código (Git).
* **Vulnerabilidad a ataques Man-in-the-Middle (MitM):** Conectar clientes sin validar el certificado SSL/TLS del servidor permite que un atacante posicionado en la red intercepte la sesión, altere consultas o capture datos en tránsito.
* **Extracción directa de datos mediante respaldos:** Un archivo de respaldo (*backup*) sin protección que sea extraído del almacenamiento puede ser leído o restaurado libremente en otro servidor sin barreras de acceso.

---

#### 2. Mecanismo de protección recomendado por elemento

| Elemento / Situación | Mecanismo de Seguridad Recomendado | Tipo de Protección |
| :--- | :--- | :--- |
| **Contraseña** | Hashing unidireccional con sal (`SCRAM-SHA-256`, `Argon2` o `crypt()` con `gen_salt()`). | Credencial en reposo |
| **RFC e Información Financiera** | Cifrado simétrico a nivel de columna (`pgp_sym_encrypt`) o cifrado previo desde la aplicación. | Datos sensibles en reposo |
| **Claves Criptográficas** | Gestores de secretos (AWS KMS, HashiCorp Vault) e inyección segura por variables de entorno. | Gestión de secretos |
| **Conexiones Cliente-Servidor** | Cifrado de transporte TLS/SSL forzado con validación de certificado (`sslmode=verify-full`). | Datos en tránsito |
| **Respaldos (Backups)** | Cifrado de archivos (`GPG`, `AES-256`) o cifrado de volumen a nivel de disco (`LUKS` / EBS Encryption). | Archivos en reposo |
| **Nombre y Correo Electrónico** | Control de acceso basado en roles (RBAC) combinado con cifrado en tránsito (TLS). | Datos operativos |

---

#### 3. Problema más crítico y justificación
El problema más crítico es la **inclusión de claves criptográficas en archivos SQL y su falta de gestión aislada**.

**Justificación:**
En criptografía, la seguridad de los datos depende exclusivamente de la confidencialidad de la clave (*Principio de Kerckhoffs*). Si la clave criptográfica se encuentra escrita dentro de las consultas SQL o archivos del proyecto, **cualquier otro mecanismo de cifrado en reposo queda completamente anulado**. No importa qué tan robustos sean los algoritmos aplicados al RFC o a la información financiera; si la clave está expuesta en el mismo código que consulta la base de datos, cualquier atacante o usuario con acceso al código/logs podrá desencriptar toda la información de forma inmediata.

---

#### 4. ¿Tener correctamente configurados los usuarios y permisos elimina estos riesgos?
**No, no los elimina.**

**Justificación:**
El control de acceso basado en usuarios y roles (RBAC mediante `GRANT` y `REVOKE`) representa únicamente una capa lógica de autorización **dentro** del motor de la base de datos activa. No mitiga los riesgos descritos por los siguientes factores:

1. **Evasión mediante archivos de respaldo:** Si un atacante o empleado obtiene acceso al archivo de respaldo (`.sql` o `.dump`), puede abrirlo directamente con un editor de texto o restaurarlo en su propio equipo, saltándose por completo todos los usuarios y permisos configurados en la base de datos original.
2. **Exposición en la red:** Los permisos internos no protegen los datos mientras viajan por el cable/Wi-Fi. Si la red no valida TLS (`verify-full`), la información puede ser interceptada en tránsito sin importar qué rol ejecute la consulta.
3. **Acceso irrestricto de administradores:** Un usuario con rol `superuser` o el administrador del sistema operativo puede leer cualquier tabla. Sin cifrado o hashing en reposo, el personal técnico con privilegios elevados tiene visibilidad total de contraseñas y datos financieros sin necesidad justificada.
4. **Inobservancia de la Defensa en Profundidad (*Defense in Depth*):** La seguridad informática requiere múltiples capas independientes. Un control de permisos correcto es indispensable, pero debe ser complementado obligatoriamente con cifrado en reposo, cifrado en tránsito y gestión externa de secretos.