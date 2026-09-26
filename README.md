# Hack The Box Writeups & Informes de Auditoría 🎯

Bienvenido a mi repositorio de resolución de máquinas y laboratorios de **Hack The Box**. Aquí documento el proceso de auditoría, análisis de vulnerabilidades, explotación y post-explotación de cada máquina completada.

---

## 📌 Estructura del Repositorio

Cada carpeta en este repositorio corresponde a un laboratorio específico e incluye un archivo `README.md` detallado con el informe técnico completo:

| Máquina | Sistema Operativo | Dificultad | Vector Principal | Escalación de Privilegios | Writeup |
|---|---|---|---|---|---|
| **Reactor** | Linux 🐧 | Fácil | CVE-2025-55182 (Next.js RCE) | Node.js V8 Inspector | [Ver Informe](./Reactor) |

---

## ⚙️ Metodología Aplicada

En cada reporte se aplican los estándares de auditoría de ciberseguridad:
1. **Reconocimiento y Escaneo:** Identificación de puertos, servicios y versiones activas.
2. **Análisis de Vulnerabilidades:** Búsqueda de fallos de seguridad (CVEs, malas configuraciones, fugas de información).
3. **Explotación:** Desarrollo/ejecución de PoCs y obtención de acceso inicial.
4. **Movimiento Lateral y Escalación:** Enumeración interna y elevación de privilegios a `root` / `SYSTEM`.
5. **Mitigación:** Recomendaciones de seguridad para parchear las vulnerabilidades encontradas.

---

> 🛠️ **Nota:** Todo el contenido tiene fines exclusivamente educativos e informativos en el marco de laboratorios éticos de pruebas de penetración.
