# billing-skill — Facturación Electrónica LatAm para TrustBid

Skill de [Claude Code](https://claude.com/claude-code) con la guía técnica y los flujos
de integración de facturación electrónica en los 4 países soportados por
**TrustBid**: Colombia, Argentina, Costa Rica y Brasil.

Cubre el ciclo completo: qué se necesita para habilitarse como facturador en cada país,
cómo emitir un comprobante válido, cómo verificar/parsear uno entrante para aprobar un
gasto, y cómo operar en campo con conectividad intermitente (cuadrillas, vendedores
itinerantes, contingencia offline).

## Estructura del repo

```
.
├── SKILL.md                              # punto de entrada de la skill (metadata + guía de uso)
├── colombia/reference.md                 # DIAN — UBL 2.1, CUFE (SHA-384), AttachedDocument
├── argentina/reference.md                # ARCA (ex AFIP) — WSFEv1/WSAA, CAE/CAEA, QR
├── costa-rica/reference.md               # Hacienda/DGT — XML v4.4, clave de 50, CAByS, OAuth2
├── brasil/reference.md                   # SEFAZ/SPED — NF-e/NFC-e/NFS-e/CT-e/MDF-e, chave de 44
└── flujos/
    ├── arquitectura-integracion.md       # diseño del módulo de facturación (patrones GoF/GRASP/distribuidos)
    ├── flujos-operativos.md              # validación de gasto, emisión, campo offline, ONG
    └── requisitos-habilitacion.md        # trámites/certificados necesarios antes de poder facturar
```

## Qué es esto

Es una **skill**, no una librería de código: un conjunto de archivos Markdown pensados
para que un modelo (Claude u otro agente) los cargue como contexto y responda con
precisión técnica sobre facturación electrónica en estos 4 países, sin necesidad de
volver a investigar la normativa cada vez.

Usa *progressive disclosure*: `SKILL.md` es liviano y decide qué archivo cargar según
la tarea — nunca se necesita meter las 4 referencias de país en contexto al mismo
tiempo.

| Si la tarea es... | Leer |
|---|---|
| Entender el formato/identificador/firma de un país específico | `<pais>/reference.md` |
| Diseñar el módulo de facturación (parser, adaptadores, cola offline) | `flujos/arquitectura-integracion.md` |
| Validar un gasto, emitir una factura, operar en campo, o el caso ONG | `flujos/flujos-operativos.md` |
| Dar de alta un tenant nuevo en un país (certificados, RUT, trámites) | `flujos/requisitos-habilitacion.md` |

## Cobertura por país

| País | Autoridad | Formato | Identificador único |
|---|---|---|---|
| 🇨🇴 Colombia | DIAN | XML UBL 2.1 | CUFE (SHA-384, 96 chars) |
| 🇦🇷 Argentina | ARCA (ex AFIP) | SOAP/XML propio (WSFEv1) | CAE (14 dígitos) / CAEA offline |
| 🇨🇷 Costa Rica | Ministerio de Hacienda / DGT | XML propio v4.4 | Clave numérica (50 chars) |
| 🇧🇷 Brasil | SEFAZ estatales + Receita Federal + municipios | XML propio (layout 4.00 / NFS-e Nacional) | Chave de Acesso (44 dígitos, Módulo 11) |

Ver la tabla comparativa completa (protocolo, firma digital, modo de validación,
contingencia offline, verificación pública) en [`SKILL.md`](SKILL.md).

## Cómo instalar la skill en Claude Code

```bash
git clone git@github.com:jairoamayac/billing-skill.git ~/.claude/skills/facturacion-electronica-latam
```

Una vez clonada en `~/.claude/skills/`, Claude Code la detecta automáticamente y la
invoca cuando la conversación trata sobre facturación electrónica en alguno de estos
países (CUFE, CAE, NF-e, DIAN, ARCA, Hacienda, SEFAZ, etc.), sin que sea necesario
pedirla por nombre.

## Diseño de la arquitectura

Los flujos de integración en `flujos/arquitectura-integracion.md` están diseñados
siguiendo patrones de arquitectura de software (Adapter por país, Strategy para
identificadores fiscales, Outbox, Circuit Breaker, State) para que el módulo de
facturación de TrustBid soporte los 4 países detrás de una interfaz estable, sin
condicionales de país esparcidos por el código.

## Mantenimiento

Los regímenes fiscales de estos países cambian con frecuencia (nuevas versiones de
esquema, resoluciones, fechas de obligatoriedad). El estado normativo documentado en
cada archivo indica la fecha de verificación — antes de usar una cifra normativa
(fecha límite, umbral, número de resolución) en producción, confirmarla contra la
fuente oficial citada al final de cada referencia de país.

## Licencia / uso

Material de referencia interno para el desarrollo de TrustBid. No constituye asesoría
legal o tributaria — para decisiones fiscales formales, consultar con un profesional
autorizado en cada jurisdicción.
