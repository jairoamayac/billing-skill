# Flujos operativos TrustBid — multi-país

Cuatro flujos transversales. Los detalles normativos de cada paso están en la
referencia del país correspondiente.

## Flujo 1 — Validación de un gasto entrante (parser + verificador)

Objetivo: impedir que una factura falsa o alterada se apruebe como gasto.

```mermaid
flowchart TD
    R["Recepción<br/>(email / upload / OCR móvil)"] --> T{Tipo de archivo}
    T -->|XML| P["Parsear directo<br/>(siempre preferido)"]
    T -->|PDF / imagen| O["OCR + decodificar QR"]
    P --> ID
    O --> ID
    ID["Identificar país y tipo de documento<br/>CO: AttachedDocument/Invoice → CUFE ·<br/>AR: QR Base64-JSON → CAE ·<br/>CR: Clave 50 (506…) ·<br/>BR: chave 44 + modelo"] --> V
    V["Validación estructural local<br/>CO: XSD + recomputar CUFE ·<br/>AR: balance + QR vs OCR ·<br/>CR: dígitos clave + totales ·<br/>BR: Módulo 11 + CFOP/CST/NCM"] --> A
    A["Verificación contra la autoridad<br/>(online, obligatoria)<br/>CO: catalogo-vpfe · AR: constatación ·<br/>CR: GET /recepcion/clave · BR: portal NF-e/SEFAZ/NFS-e"] --> X
    X{"Cross-check:<br/>ID + emisor + total<br/>== respuesta autoridad?"}
    X -->|Coincide y estado válido| OK["✅ Gasto APROBABLE<br/>persistir XML + respuesta autoridad"]
    X -->|Discrepancia o estado inválido| NO["❌ RECHAZAR gasto<br/>+ alerta de auditoría"]
```

Reglas duras:
- Un tiquete CR (tipo 04) o una NFC-e brasileña de consumidor **no respaldan crédito
  fiscal** — marcarlos como no deducibles aunque sean auténticos.
- En CR el gasto solo es deducible si además se emitió el **Mensaje de Receptor**
  (tipo 05/06/07) dentro del plazo — TrustBid debe generarlo, no solo verificar.
- En CO, si el proveedor es informal (sin factura), el flujo correcto es emitir
  **Documento Soporte electrónico**, no aprobar el recibo a secas.

## Flujo 2 — Emisión de factura propia

```mermaid
flowchart TD
    S["Solicitud de emisión<br/>(venta / servicio del tenant)"] --> C
    C["1. Resolver contexto fiscal del emisor y receptor → tipo de comprobante<br/>AR: matriz condición × receptor → Clase A/B/C/E/M ·<br/>CR: ¿exonerado? → EXONET → bloque Exoneracion ·<br/>CO: CustomizationID + numeración vigente ·<br/>BR: mercancía→NF-e/NFC-e o servicio→NFS-e"] --> B
    B["2. Construir documento (Builder por país)<br/>+ asignar consecutivo transaccional"] --> F
    F["3. Firmar (SignerPort)<br/>— siempre antes de encolar —"] --> Q
    Q["4. Outbox → adaptador del país → autoridad"] --> M{Modo de validación}
    M -->|Sync CO/AR/BR| RES{Resultado}
    M -->|Async CR, NFS-e BR| POLL["201 + polling"] --> RES
    RES -->|AUTORIZADO| POST
    RES -->|RECHAZADO| REJ["Corregir y reemitir<br/>BR: mismo nNF · AR: mismo nro si no consumió CAE"]
    REJ --> B
    POST["5. Post-autorización<br/>CO: AttachedDocument → zip → email ·<br/>AR: PDF con CAE+QR ·<br/>CR: PDF + XML firmado + XML respuesta ·<br/>BR: DANFE/DANFSE con protocolo"] --> ADJ["Ajustes posteriores →<br/>Nota Crédito/Débito referenciando el original"]
```

## Flujo 3 — Operación en campo con conectividad intermitente

Principio común: **un dispositivo = una identidad de numeración exclusiva** (PV en AR,
Sucursal+Terminal en CR, série en BR, prefijo/rango en CO). Nunca compartir secuencias
entre terminales offline.

```mermaid
flowchart TD
    A["🔵 ANTES de salir (online, sincronización matutina)<br/>AR: descargar CAEA de la quincena → SQLite cifrado ·<br/>BR: certificado A1 + CSC en el dispositivo ·<br/>CR: .p12 + PIN + catálogo CAByS ·<br/>Común: catálogos + rangos de numeración"] --> B
    B["🟠 EN CAMPO sin señal (emisión local)<br/>AR: CAEA (tipoCodAut=A), consecutivo local, PDF+QR ·<br/>CR: clave Situación=3, firmar local, ticket provisional ·<br/>BR: NFC-e offline con CSC; NF-e → EPEC (tpEmis=4) o SVC (6/7) ·<br/>CO: contingencia (InvoiceTypeCode=03)"] --> C
    C["🟢 AL RECONECTAR (cola offline → outbox → autoridad)<br/>AR: reportar CAEA ≤ 8 días tras fin de quincena (+ PV sin uso) ·<br/>CR: transmitir ≤ 48 h ·<br/>BR: NFC-e ≤ 24 h · EPEC ≤ 168 h ·<br/>Común: monitorear deadline legal y alertar antes de vencer"]
```

### Traslado de materiales (cuadrillas)
| País | Documento de tránsito |
|---|---|
| AR | Remito Clase R (CAI); COT en provincias con control (ARBA ≥ $7.220.557 o ≥ 4.500 kg) |
| BR | NF-e de remesa (CFOP 5.554/6.554 o 5.949/6.949, CST 40/41) + MDF-e con encerramento al llegar |
| CR | No hay documento fiscal de tránsito interno — portar facturas de compra + albarán interno |
| CO | Sin guía fiscal específica de tránsito interno para este caso |

### Tributos territoriales a registrar por gasto/servicio
- AR: provincia de ejecución (sustento territorial) → Convenio Multilateral / SIFERE.
- BR: estado destino → DIFAL (fórmula base dupla); municipio del tomador → ISS vía ADN.
- CR: cantón → posibles patentes municipales si la operación se vuelve permanente.

## Flujo 4 — ONG / entidades sin fines de lucro

Clasificar SIEMPRE el ingreso antes de decidir si se factura:

```mermaid
flowchart TD
    I["Ingreso de la ONG"] --> T{Naturaleza del ingreso}
    T -->|Donación / cuota / aporte institucional| D["🚫 NO se factura en ningún país"]
    D --> DN["CO: certificado de donación (rep. legal + revisor fiscal; Ley 2380/2024) ·<br/>AR: recibo no fiscal (respaldo: certificado de exención ARCA) ·<br/>CR: recibo interno (exoneración de IVA en COMPRAS vía EXONET) ·<br/>BR: recibo; inmunidad art. 150 VI 'c' CF/88"]
    T -->|Venta de bien o servicio<br/>libros, seminarios, consultorías| V["🧾 SÍ se factura electrónicamente"]
    V --> VN["CO: factura con validación previa (el régimen especial no exime) ·<br/>AR: Clase C con CAE (condición IVA Exento) ·<br/>CR: factura normal; si el CLIENTE es exonerado → bloque Exoneracion ·<br/>BR: NF-e/NFS-e con CST 40/41 · CST 08 PIS/COFINS · ExigibilidadeISS 2/3"]
```

Egresos de la ONG en campo (proveedores informales):
- CO: generar **Documento Soporte electrónico** por cada compra a no obligados
  (jornaleros, transporte rural) — vía OCR del recibo + transmisión a la DIAN.
- AR/CR/BR: exigir comprobante fiscal del proveedor; sin él, el gasto no es deducible y
  debe marcarse en TrustBid como "sin soporte fiscal" para el reporte al donante.
