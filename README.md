# Auditoría de Seguridad en Base de Datos — DataCorp

![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white)
![NIST](https://img.shields.io/badge/Framework-NIST%20SP%20800--53-0033A0)
![CIS](https://img.shields.io/badge/Benchmark-CIS%20MySQL-2C5F8A)
![OWASP](https://img.shields.io/badge/OWASP-A01%20Broken%20Access%20Control-red)
![Académico](https://img.shields.io/badge/Tipo-Académico-green)

Auditoría técnica de seguridad sobre el motor MySQL de **DataCorp**, empresa que sufrió una filtración de información crítica atribuida a una configuración deficiente de usuarios y privilegios. El trabajo documenta el diagnóstico de vulnerabilidades, el script SQL de corrección, la validación mediante simulación de ataques y las recomendaciones ejecutivas.

---

## Vulnerabilidades identificadas

Se detectaron **6 vulnerabilidades** en la configuración original, distribuidas en tres niveles de gravedad:

| ID | Problema | Gravedad | Urgencia |
|----|----------|----------|----------|
| V-001 | Contraseñas triviales (`1234`, `vend123`, `cont123`) | **CRÍTICO** | Inmediata |
| V-002 | Host comodín `%` en todos los usuarios | **ALTO** | Inmediata |
| V-003 | Admin con `ALL PRIVILEGES ON *.*` + `GRANT OPTION` | **CRÍTICO** | Inmediata |
| V-004 | Permiso `DELETE` innecesario en usuario Vendedor | **MEDIO** | Corto plazo |
| V-005 | Contador con `ALL PRIVILEGES` (superusuario) | **ALTO** | Corto plazo |
| V-006 | Usuario `invitado` sin propósito de negocio definido | **MEDIO** | Corto plazo |

---

## Solución aplicada

Se implementó el **Principio de Menor Privilegio (PoLP)** + **Defensa en Profundidad** + **Separación de Roles**.

```sql
-- PASO 1: Eliminar todos los usuarios inseguros
DROP USER IF EXISTS 'admin'@'%';
DROP USER IF EXISTS 'vendedor'@'%';
DROP USER IF EXISTS 'contador'@'%';
DROP USER IF EXISTS 'invitado'@'%';

-- PASO 2: Admin — solo acceso local, contraseña robusta
CREATE USER 'admin_sec'@'localhost' IDENTIFIED BY 'K#p9$mL29z!Q_Xw8';
GRANT ALL PRIVILEGES ON *.* TO 'admin_sec'@'localhost' WITH GRANT OPTION;

-- PASO 3: Vendedor — IP interna, sin DELETE
CREATE USER 'vendedor_user'@'192.168.1.0/255.255.255.0'
    IDENTIFIED BY 'S3cur3_V3nd_2026!';
GRANT SELECT, INSERT ON empresa.ventas
    TO 'vendedor_user'@'192.168.1.0/255.255.255.0';

-- PASO 4: Contador — solo lectura, IP fija
CREATE USER 'contador_user'@'192.168.1.50' IDENTIFIED BY 'C0nt4_S4f3_#2026';
GRANT SELECT ON empresa.* TO 'contador_user'@'192.168.1.50';

-- PASO 5: Aplicar cambios
FLUSH PRIVILEGES;
```

---

## Comparativa antes / después

| Usuario | Config. insegura | Config. segura |
|---------|-----------------|----------------|
| `admin` | `@'%'` · pass: `1234` · `ALL PRIVILEGES + GRANT OPTION` | `@'localhost'` · pass robusta · sin acceso externo |
| `vendedor` | `@'%'` · `SELECT, INSERT, DELETE ON empresa.*` | `@'192.168.1.x'` · `SELECT, INSERT ON empresa.ventas` |
| `contador` | `@'%'` · `ALL PRIVILEGES ON empresa.*` | `@'192.168.1.50'` · `SELECT ON empresa.*` |
| `invitado` | `@'%'` · `SELECT ON empresa.*` | **Eliminado** (sin función de negocio) |

---

## Resultados de validación

Todas las pruebas de simulación de ataque pasaron correctamente:

| Test | Operación | Resultado |
|------|-----------|-----------|
| T-001 | Conexión `admin` desde IP externa | `Acceso denegado` ✓ |
| T-002 | Login con contraseña `1234` | `Error de autenticación` ✓ |
| T-003 | `DELETE FROM ventas` (vendedor_user) | `ERROR 1142 – permission denied` ✓ |
| T-004 | `DROP TABLE` (contador_user) | `ERROR 1142 – permission denied` ✓ |
| T-005 | Login usuario `invitado` | `ERROR 1045 – user does not exist` ✓ |
| T-006 | `SELECT ventas` (vendedor_user, red interna) | `Datos retornados correctamente` ✓ |

---

## Estructura del repositorio

```
datacorp-db-audit/
├── scripts/
│   ├── script_original_inseguro.sql   # Configuración vulnerable original
│   └── script_correccion_seguro.sql   # Corrección con PoLP aplicado
├── extras/
│   ├── vistas_contador.sql            # VIEW para acceso controlado a datos sensibles
│   ├── log_auditoria.sql              # Activación del log general de MySQL
│   └── procedimientos_almacenados.sql # Stored procedures para operaciones controladas
├── informe/
│   └── Informe_Auditoria_BD_DataCorp_FINAL.docx
└── README.md
```

---

## Principios de seguridad aplicados

- **PoLP** (Principio de Menor Privilegio): cada usuario recibe solo los permisos indispensables para su función.
- **Defensa en Profundidad**: tres capas independientes — permisos, contraseñas y restricción de red por IP.
- **Separación de Roles**: ningún usuario operacional tiene capacidad de administración.

---

## Marcos de referencia

| Framework | Aplicación |
|-----------|------------|
| NIST SP 800-53 Rev. 5 | Controles de acceso (AC) y auditoría (AU) |
| CIS Benchmark for MySQL 8.0 | Configuración segura de usuarios y contraseñas |
| OWASP Top 10 — A01 Broken Access Control | Clasificación del riesgo principal |
| ISO/IEC 27001:2022 | Metodología de evaluación y tratamiento de riesgos |

---

## Recomendaciones destacadas

1. **Política formal de contraseñas** — mínimo 16 caracteres + caducidad cada 90 días (`password_lifetime`).
2. **Eliminar `%` en todos los hosts** — solo IPs internas o `localhost` para admin.
3. **Activar log de auditoría** — `general_log` o plugin `audit_log` con alertas sobre DDL (`DROP`, `ALTER`).
4. **Implementar vistas (VIEW)** para acceso a tablas con datos sensibles (nóminas, clientes).
5. **SSL/TLS obligatorio** — `require_secure_transport = ON` para cifrar todas las conexiones.
6. **MFA para cuentas administrativas** — integración via PAM en MySQL Enterprise.

> ⚠️ **Aviso**: el script original inseguro se incluye únicamente como referencia diagnóstica. No debe ejecutarse en ningún entorno productivo.

---

## Autor

**Juan Zúñiga Carrasco** — Pentesting, Ciberseguridad y Control de Bases de Datos  
Asignatura: *Ciberseguridad, Control y Monitoreo de Bases de Datos* · Prof. Víctor Lucero · 2026

---

Este repositorio corresponde a un trabajo académico confidencial. Uso libre bajo licencia MIT con fines educativos.
