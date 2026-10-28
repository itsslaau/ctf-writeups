# Security Labs — Write-ups

Colección personal de write-ups de máquinas de CTF y laboratorios de pentesting.
Material de estudio: cada carpeta documenta el proceso de reconocimiento, explotación
y escalada de una máquina, con foco en entender el *porqué* de cada técnica.

> ⚠️ Todo el contenido tiene credenciales, hashes y datos sensibles **redactados**.
> Publicado con fines educativos.

---

## Índice

### HackSmarter

| Máquina | Dificultad | SO | Temas principales | Write-up |
|---|---|---|---|---|
| Samurai | Easy | Linux | Joomla, CVE-2023-23752, credential reuse, RCE por plantilla, SUID + PATH hijack | [Ver](./hacksmarter/samurai/README.md) |

<!-- Añade nuevas filas/plataformas según vayas resolviendo máquinas -->

---

## Estructura del repo

```
.
├── README.md                  ← este índice
├── .gitignore
└── <plataforma>/
    └── <maquina>/
        ├── README.md          ← write-up (GitHub lo renderiza al entrar)
        ├── guia-tecnica.md    ← notas conceptuales reutilizables
        └── img/               ← capturas
```

## Convenciones

- El write-up de cada máquina es el `README.md` de su carpeta.
- Credenciales y secretos siempre como `[REDACTED]`.
- Antes de publicar un write-up nuevo: comprobar que la plataforma permite su difusión.

---

## Disclaimer

Este repositorio es material educativo sobre seguridad ofensiva realizado en entornos
**autorizados** (laboratorios y máquinas diseñadas para practicar). Las técnicas aquí
descritas solo deben aplicarse sobre sistemas para los que se tiene permiso explícito.
