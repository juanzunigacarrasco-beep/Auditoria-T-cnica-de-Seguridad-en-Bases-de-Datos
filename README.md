# Laboratorio Avanzado — Cifrado en Plataformas de Streaming

![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Kali Linux](https://img.shields.io/badge/Kali-Linux-557C94?logo=kalilinux&logoColor=white)
![AES](https://img.shields.io/badge/AES-CTR%20%7C%20GCM%20%7C%20CBC-6c3483)
![Académico](https://img.shields.io/badge/Tipo-Académico-green)

Evaluación empírica de los modos de operación AES (**CTR**, **GCM**, **CBC**) en un entorno de streaming simulado sobre Kali Linux. Se mide rendimiento, se demuestra la vulnerabilidad de reutilización de nonce, se analiza la propagación de errores y se valida la protección AEAD con GCM.

---

## Estructura del laboratorio

| Fase | Script | Objetivo |
|------|--------|----------|
| 1 | `01_benchmark.py` | Benchmark de latencia CTR / GCM / CBC sobre 50 MB de datos |
| 2 | `02_attack.py` | Ataque de reutilización de nonce — recuperación de texto plano via XOR |
| 3 | `03_resilience.py` | Propagación de errores ante bytes corrompidos en red |
| 4 | `04_gcm.py` | Autenticación AEAD con GCM — protección de metadatos (AAD) |

---

## Requisitos

- Docker >= 20.x
- Docker Compose >= 2.x
- Python 3.x y PyCryptodome (incluidos en el contenedor)

---

## Instalación y ejecución

```bash
# 1. Clonar el repositorio
git clone https://github.com/usuario/crypto-streaming-lab.git
cd crypto-streaming-lab

# 2. Levantar el contenedor
docker-compose up -d

# 3. Entrar al contenedor
docker exec -it crypto-lab-crypto-lab-1 bash

# 4. Ejecutar los scripts en orden
python3 scripts/01_benchmark.py
python3 scripts/02_attack.py
python3 scripts/03_resilience.py
python3 scripts/04_gcm.py
```

---

## Resultados obtenidos

| Modo | Tiempo (50 MB) | Paralelismo | Autenticación | Propagación de errores |
|------|---------------|-------------|---------------|------------------------|
| `CTR` | **0.0941 s** ★ | Sí | No | 1 byte |
| `GCM` | 0.1100 s | Sí | **AEAD ★** | Rechazo de paquete |
| `CBC` | 0.1276 s | No | No | 2 bloques |

---

## Estructura del repositorio

```
crypto-streaming-lab/
├── docker-compose.yml
├── scripts/
│   ├── 01_benchmark.py      # Fase 1 — benchmark CTR/GCM/CBC
│   ├── 02_attack.py         # Fase 2 — ataque nonce reuse
│   ├── 03_resilience.py     # Fase 3 — propagación de errores
│   └── 04_gcm.py            # Fase 4 — integridad AEAD con GCM
├── informe/
│   └── Informe_Cifrado_Streaming.docx
└── README.md
```

---

## Conceptos clave

- **AES-CTR**: modo de flujo, completamente paralelizable, vulnerable a reutilización de nonce.
- **AES-GCM**: esquema AEAD, recomendado por TLS 1.3, autentica datos asociados (AAD) sin cifrarlos.
- **AES-CBC**: modo secuencial, propaga errores en 2 bloques, no recomendado para nuevas implementaciones.
- **Nonce reuse attack**: si `nonce` se repite con la misma clave en CTR → `C1 XOR C2 XOR P1 = P2` sin romper AES.
- **AEAD / AAD**: permite proteger la integridad de metadatos (User-ID, nivel de suscripción) visibles en red.

> ⚠️ **Aviso**: el ataque de la Fase 2 se incluye únicamente con fines educativos. Demuestra una vulnerabilidad real de implementación sin comprometer ningún sistema externo.

---

## Autor

**Juan Zúñiga Carrasco** — Técnico en Nivel Superior en Informática, mención Ciberseguridad  
Asignatura: *Criptografía Aplicada a la Ciberseguridad* · Prof. Gerardo Calquín

---

Este repositorio corresponde a un trabajo académico. El uso del código es libre bajo los términos de la licencia MIT.
