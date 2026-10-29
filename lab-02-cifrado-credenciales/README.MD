# Reporte Técnico: Implementación y Auditoría de Seguridad en PostgreSQL

## 1. Introducción
El manejo de información confidencial en sistemas de bases de datos relacionales exige la implementación de estrategias robustas de seguridad tanto para los datos almacenados (datos en reposo) como para las comunicaciones en red (datos en tránsito). En entornos por defecto de PostgreSQL, factores como el almacenamiento de credenciales o identificadores personales en texto claro, la exposición de claves criptográficas dentro de scripts SQL y la falta de cifrado de transporte incrementan exponencialmente la superficie de ataque.

Este informe documenta el análisis, la mitigación de riesgos y la implementación práctica de controles de seguridad en PostgreSQL, abarcando desde el hashing de contraseñas y el cifrado simétrico a nivel de columna hasta el aseguramiento de la capa de transporte mediante TLS 1.3 y la definición de políticas estrictas para la gestión del ciclo de vida de claves criptográficas.

---

## 2. Protección de Credenciales
Para mitigar el acceso no autorizado y la suplantación de identidad, es imprescindible comprender la diferencia entre los distintos estados en los que se puede representar la información sensible:

| Tipo | Definición | Reversibilidad | Caso de Uso |
| :--- | :--- | :--- | :--- |
| **Texto Claro** | Datos almacenados sin ninguna transformación o protección legible directamente por cualquier usuario o proceso. | N/A (Totalmente visible) | Datos públicos o de libre acceso. **Nunca debe usarse en contraseñas.** |
| **Hash (Unidireccional)** | Transformación matemática no reversible que convierte una cadena de entrada en un valor de longitud fija de forma determinista. Al incorporar una "sal" (*salt*), se protegen contra ataques de tablas de arcoíris. | **Irreversible** | Almacenamiento seguro de contraseñas (`SCRAM-SHA-256`, `crypt` con `gen_salt`). |
| **Cifrado (Simétrico/Asimétrico)** | Transformación matemática reversible de datos mediante el uso de una o más claves criptográficas. | **Reversible** (requiere la clave correspondiente) | Protección de datos sensibles que deben ser recuperados originalmente por la aplicación (`pgp_sym_encrypt`). |

---

## 3. Protección de Datos Sensibles
En la base de datos se categorizaron y protegieron los siguientes elementos debido al impacto de confidencialidad e integridad que representan:

* **Contraseñas de Usuario:** Se procesaron mediante funciones de hash unidireccional con sal (`crypt` y `gen_salt`). Esto garantiza que, aun si la base de datos o un respaldo sufren una exfiltración, no sea posible recuperar la contraseña original en texto plano.
* **Registro Federal de Contribuyentes (RFC) e Información Financiera:** Se aplicó cifrado simétrico a nivel de columna utilizando la extensión `pgcrypto` (`pgp_sym_encrypt`). Estos datos personales identificables (PII) deben ser protegidos en reposo para cumplir con leyes de privacidad (como la LFPDPPP y GDPR), pero requieren ser desencriptados mediante una clave autorizada cuando la aplicación de negocio lo requiera.

---

## 4. Seguridad en Tránsito (TLS)
El uso de Cifrado de Transporte mediante **TLS (Transport Layer Security)** resuelve el problema de la intercepción de tráfico (*network sniffing*) y los ataques de intermediario (*Man-in-the-Middle* / MitM) en la red local o distribuida. Sin TLS, las consultas SQL, las credenciales y los datos retornados viajan en paquetes de red en texto plano.

### Verificaciones realizadas en PostgreSQL:
1. **Configuración del Servidor:**
   
   ``SHOW ssl;                         -- Confirmó soporte activo (on)
   SHOW ssl_min_protocol_version;    -- Verificó versión mínima requerida (TLSv1.2)
   SHOW port;                        -- Identificó puerto de escucha (5432)
   SHOW listen_addresses;            -- Confirmó las interfaces de red configuradas``

   ssl = true: Confirmó la canalización cifrada de la sesión actual.

version = TLSv1.3: Confirmó el uso del protocolo de transporte más reciente y seguro.

cipher = TLS_AES_256_GCM_SHA384: Validó el uso de una suite de cifrado simétrico de 256 bits con autenticación GCM e integridad SHA-384.

5. Gestión de Claves Criptográficas
El almacenamiento de claves criptográficas dentro de archivos de código o scripts SQL (por ejemplo, en la consulta SELECT pgp_sym_decrypt(rfc_cifrado, 'CLAVE-DE-PRUEBA')) representa uno de los fallos de seguridad más severos.

Riesgos de almacenar claves junto con los scripts:
Exposición en Logs de Servidor: PostgreSQL registra las sentencias SQL ejecutadas en texto claro dentro de sus archivos de registro (PostgreSQL logs).

Historial de Comandos: Las consultas ejecutadas quedan almacenadas permanentemente en archivos locales como .psql_history.

Vistas de Monitoreo en Tiempo Real: Usuarios con privilegios de lectura o administración pueden visualizar las llaves expuestas consultando la vista del sistema pg_stat_activity.

Exfiltración en Repositorios Git: Si el archivo .sql se sube a un repositorio de código, la llave queda expuesta públicamente o a nivel organizacional, permaneciendo accesible en el historial de versiones incluso si el archivo es eliminado posteriormente.

Las claves criptográficas deben gestionarse externamente mediante variables de entorno o gestores de secretos (como HashiCorp Vault o AWS KMS) e inyectarse dinámicamente en tiempo de ejecución.

6. Resultados Obtenidos
Cifrado de Datos en Reposo: Se logró la separación efectiva entre credenciales (protegidas con hash e irreversibles) y datos sensibles (RFC/financieros protegidos mediante pgp_sym_encrypt).

Canal Seguro Confirmado: Se comprobó que el 100% de la comunicación entre el cliente (127.0.0.1) y el servidor PostgreSQL se realiza de manera cifrada bajo la suite TLS_AES_256_GCM_SHA384 y protocolo TLS 1.3.

Modelo Comparativo Integrado: Se definió la matriz de controles de seguridad en reposo, en tránsito y de claves para cada tipo de dato en la organización (07-comparacion.md).

Política de Seguridad: Se estableció un marco normativo de 8 reglas para normar la generación, almacenamiento, rotación, revocación y destrucción de llaves de cifrado.

7. Problemas Encontrados y Dificultades Técnicas
Inyección Hardcodeada de Claves: Durante las pruebas iniciales con pgp_sym_encrypt, la llave criptográfica formaba parte de la instrucción SQL. Se identificó el riesgo de persistencia en .psql_history y en los registros del servidor.

Persistencia en Git: Se analizó el impacto de publicar scripts .sql con claves en repositorios de versiones, concluyendo la necesidad de utilizar herramientas de limpieza de historial (BFG Repo-Cleaner / git filter-repo) y escáneres automáticos de secretos (Gitleaks).

Configuración de Conexión de Clientes: En clientes gráficos o scripts psql, la falta del parámetro explícito sslmode=verify-full o la aceptación implícita de certificados no autorizados podría exponer la sesión a ataques Man-in-the-Middle si no se valida el certificado CA del servidor.

8. Conclusiones y Medidas para Entornos Productivos
Aprendizajes Clave:
La Autorización no es Suficiente: La correcta configuración de usuarios y roles (RBAC) restringe el acceso dentro del motor de PostgreSQL activo, pero no protege los datos en archivos de respaldo (dumps), tráfico de red no cifrado o contra administradores con acceso a disco.

Estrategia de Defensa en Profundidad (Defense in Depth): Es indispensable combinar controles lógicos de acceso, cifrado en reposo (pgcrypto), cifrado en tránsito (TLS 1.3) y gestión aislada de secretos.

Principio de Kerckhoffs: La seguridad de los datos cifrados radica exclusivamente en el resguardo de la clave. Exponer la clave anula por completo la efectividad de cualquier algoritmo de cifrado por complejo que sea.

Medidas Recomendadas para Entornos Productivos:
Forzar TLS con Validación Estricta: Configurar pg_hba.conf con reglas exclusivamente hostssl e implementar en todos los clientes la directiva sslmode=verify-full junto con certificados firmados por una CA válida.

Integración de Gestores de Secretos: Prohibir cualquier clave en texto plano en archivos .sql o .env locales; utilizar integraciones con HashiCorp Vault, AWS KMS o GCP Secret Manager.

Cifrado de Respaldo y Volúmenes: Implementar cifrado a nivel de sistema de archivos/disco (LUKS o AWS EBS Encryption) para los volúmenes de datos de PostgreSQL y cifrar los archivos .dump mediante GPG (AES-256) antes de su transferencia a almacenamiento secundario.

Pipeline de CI/CD Seguro: Integrar escáneres de secretos (Trufflehog o Gitleaks) en el flujo de desarrollo para bloquear automáticamente cualquier commit que contenga credenciales o claves criptográficas.

# Preguntas de reflexion

### Preguntas de Reflexión: Seguridad en Bases de Datos y Criptografía

#### 1. ¿Por qué una contraseña no debe almacenarse como texto claro?
Porque en caso de una brecha de seguridad, exfiltración de archivos de respaldo o acceso por parte de un usuario interno con privilegios (como administradores de sistema), las credenciales quedan expuestas directamente. Esto permite la suplantación inmediata de identidad y facilita ataques de reutilización de contraseñas (*credential stuffing*) en otras aplicaciones y servicios de la organización o del usuario.

#### 2. ¿Por qué un hash no es equivalente a un cifrado?
Un **hash** es un proceso unidireccional e irreversible (no existe clave para recuperar la entrada original), diseñado para verificar integridad o autenticar contraseñas sin revelar el texto plano. En cambio, el **cifrado** es un proceso bidireccional y reversible mediante el uso de una clave criptográfica, diseñado para proteger datos que posteriormente requieren ser recuperados en su formato original.

#### 3. ¿Por qué el cifrado de una columna no sustituye al control de acceso?
El cifrado de columna protege la **confidencialidad de los datos en reposo** (impidiendo su lectura desde el disco o desde un archivo de respaldo), mientras que el control de acceso (RBAC) gestiona **quién está autorizado para interactuar con la base de datos**. Sin control de acceso, un usuario no autorizado con conexión a la base de datos podría eliminar o modificar tablas enteras, o bien ejecutar funciones de desencriptación si obtiene acceso a la clave. Ambos mecanismos se complementan bajo el principio de *Defensa en Profundidad*.

#### 4. ¿Por qué TLS no sustituye a los mecanismos de autorización de PostgreSQL?
**TLS** únicamente asegura el canal de comunicación en la red (*datos en tránsito*), evitando el espionaje de paquetes (*sniffing*) y ataques de intermediario (*Man-in-the-Middle*). No obstante, TLS no determina ni restringe las acciones que un usuario autenticado puede realizar; son los mecanismos de autorización de PostgreSQL (roles, `GRANT` / `REVOKE`, RLS) los que definen a qué tablas, vistas o consultas tiene acceso cada conexión.

#### 5. ¿Qué ocurriría si una organización pierde una clave utilizada para cifrar información?
Se produce una **pérdida permanente e irreversible de los datos** cifrados con dicha clave (destrucción criptográfica de la información). Debido a que los algoritmos de cifrado modernos (como AES-256) son computacionalmente imposibles de romper por fuerza bruta, la información no podrá recuperarse de ninguna manera.

#### 6. ¿Qué ocurriría si una clave criptográfica se publica accidentalmente en GitLab?
La clave debe considerarse **comprometida de inmediato**. Atacantes y bots automatizados monitorean repositorios en tiempo real para capturar credenciales expuestas y desencriptar los datos. Además, la clave permanecerá en el historial de *commits* de Git (incluso si se elimina el archivo en un commit posterior), obligando a revocar la clave, re-cifrar los datos con una clave nueva y reescribir la historia del repositorio.

#### 7. ¿Por qué un repositorio privado no debe considerarse un almacén de secretos?
Un repositorio privado no cuenta con controles de acceso de grano fino para secretos individuales ni genera registros de auditoría de lectura por llave. Asimismo, el código se clona de forma local en las estaciones de trabajo de múltiples desarrolladores (ampliando la superficie de ataque) y mantiene un historial persistente donde los secretos quedan guardados de manera indefinida.

#### 8. ¿Qué diferencia existe entre proteger información almacenada y proteger información durante su transmisión?
* **Información almacenada (En reposo):** Protege archivos en disco, bases de datos y respaldos contra robo físico de equipos, lectura no autorizada en almacenamiento o exfiltración de archivos `.sql`/`.dump` (se resuelve con `pgp_sym_encrypt`, LUKS, GPG).
* **Información durante su transmisión (En tránsito):** Protege los paquetes de datos que viajan por el cable o Wi-Fi entre el cliente y el servidor contra intercepción, modificación o lectura no autorizada en la red (se resuelve con TLS/SSL).

#### 9. ¿Por qué la gestión de claves es tan importante como el algoritmo criptográfico?
De acuerdo con el **Principio de Kerckhoffs**, la seguridad de un sistema criptográfico radica exclusivamente en mantener en secreto la clave, no el algoritmo (algoritmos como AES o TLS 1.3 son públicos y estándar). Si la gestión de la clave falla (almacenamiento en texto plano, falta de rotación, claves débiles), la fortaleza matemática del algoritmo queda totalmente anulada.

#### 10. ¿Qué mecanismo adicional implementarías en una organización para proteger los respaldos de las bases de datos?
1. **Cifrado Asimétrico previo a la transferencia:** Cifrar los archivos de respaldo del lado del cliente utilizando `GPG` (AES-256) o llaves de servicios gestionados (AWS KMS) antes de enviarlos al almacenamiento de respaldo.
2. **Almacenamiento Inmutable / WORM (*Write Once, Read Many*):** Configurar políticas de retención bloqueadas (como S3 Object Lock) para prevenir la eliminación o sobreescritura por *ransomware*.
3. **Aislamiento de Entorno:** Guardar los respaldos en una cuenta o servidor físicamente separado del entorno de producción, limitando el acceso mediante roles estrictos con autenticación de doble factor (MFA).