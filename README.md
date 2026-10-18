# core-transfers-system

Capa de sistema del ecosistema de transferencias interbancarias. Encapsula la integración con el core bancario legado (simulación): transforma el modelo JSON interno a XML (formato del core), simula la respuesta del core para pruebas locales y devuelve el código de resultado al Process API.

---

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/api/transferencias` | Registra una transferencia en el core bancario. |. |

Puerto local: **8083**

---

## Arquitectura de tres capas

```
transfers-experience-api  :8081
     │
payment-order-process     :8082
     │  JSON interno
     ▼
core-transfers-system     :8083   ← este servicio
     │  XML (TransferenciaRequest)
     ▼
Core bancario legado
```

---

## Decisiones de diseño

**JSON → XML para el core legado.** El core bancario espera un XML con estructura fija (`TransferenciaRequest`). La transformación normaliza la glosa (mayúsculas, máximo 30 caracteres) y convierte `fechaSolicitud` a `yyyyMMddHHmmss` en UTC antes de armar el XML. Esto aísla al resto del sistema del formato del core.

**Simulación del core separada en un sub-flow mockeable.** `simulate-core-legado-sf` determina el `codigoCore` según el prefijo de `cuentaOrigen`. Al estar en un sub-flow propio, las pruebas MUnit pueden mockearlo directamente para inyectar cualquier código sin necesidad de cuentas reales.

**Magic accounts para pruebas end-to-end:**

| Prefijo de `cuentaOrigen` | Código core | Resultado en Experience |
|---|---|---|
| `51xxxx` | `51` | 422 `SALDO_INSUFICIENTE` |
| `62xxxx` | `62` | 422 `CUENTA_BLOQUEADA` |
| `91xxxx` | `91` | 503 `SERVICIO_NO_DISPONIBLE` |
| cualquier otro | `00` | 201 `APROBADA` |

---

## Cómo ejecutar localmente

**Requisitos:** Mule 4.12.2 EE, Java 17, Maven 3.9.x. Este servicio no depende de ninguna otra capa.

---

## Pipeline CI/CD

Mismo esquema que las otras capas: build único → GitHub Release inmutable → dev → qa → prod con aprobación manual. Cobertura MUnit ≥ 80 % requerida.

Al ser la capa más baja, en el primer despliegue debe iniciarse antes que Process y Experience.
