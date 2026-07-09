# Flujos operativos TrustBid — multi-país

Cuatro flujos transversales. Los detalles normativos de cada paso están en la
referencia del país correspondiente.

## Flujo 1 — Validación de un gasto entrante (parser + verificador)

Objetivo: impedir que una factura falsa o alterada se apruebe como gasto.

```
Recepción (email/upload/OCR móvil)
   │
   ▼
1. Detectar tipo de archivo
   ├─ XML → parsear directo (siempre preferido)
   └─ PDF/imagen → OCR + decodificar QR
   │
   ▼
2. Identificar país y tipo de documento
   ├─ CO: raíz <AttachedDocument> o <Invoice> UBL → extraer CUFE (cbc:UUID)
   ├─ AR: QR https://www.arca.gob.ar/fe/qr/?p={Base64 JSON} → CAE, CUIT, importe
   ├─ CR: <Clave> de 50 chars (empieza por 506)
   └─ BR: chave de 44 dígitos (DANFE barcode/QR) + modelo (55/65/NFS-e)
   │
   ▼
3. Validación estructural local
   ├─ CO: XSD DIAN + recomputar CUFE (SHA-384) si se tiene la clave técnica
   ├─ AR: balance impTotal = neto + IVA + tributos; JSON del QR vs texto OCR
   ├─ CR: dígitos de la clave (fecha, cédula, situación) + coherencia totales (5 dec.)
   └─ BR: Módulo 11 del cDV; coherencia CFOP/CST/NCM
   │
   ▼
4. Verificación contra la autoridad (online, obligatoria antes de aprobar)
   ├─ CO: GET catalogo-vpfe.dian.gov.co/document/searchqr?documentkey={CUFE}
   ├─ AR: Constatación de Comprobantes (arca.gob.ar)
   ├─ CR: GET /recepcion/v1/recepcion/{clave} → ind-estado == "aceptado"
   └─ BR: portal NF-e (nacional) / SEFAZ estatal (NFC-e) / nfse.gov.br (servicios)
   │
   ▼
5. Cross-check: ID único + emisor + total del documento == respuesta autoridad
   ├─ OK → gasto APROBABLE (persistir evidencia: XML + respuesta de la autoridad)
   └─ Discrepancia o estado inválido → RECHAZAR gasto + alerta de auditoría
```

Reglas duras:
- Un tiquete CR (tipo 04) o una NFC-e brasileña de consumidor **no respaldan crédito
  fiscal** — marcarlos como no deducibles aunque sean auténticos.
- En CR el gasto solo es deducible si además se emitió el **Mensaje de Receptor**
  (tipo 05/06/07) dentro del plazo — TrustBid debe generarlo, no solo verificar.
- En CO, si el proveedor es informal (sin factura), el flujo correcto es emitir
  **Documento Soporte electrónico**, no aprobar el recibo a secas.

## Flujo 2 — Emisión de factura propia

```
Solicitud de emisión (venta/servicio del tenant)
   │
   ▼
1. Resolver contexto fiscal del tenant y del receptor
   (país, régimen, condición IVA/exoneración del cliente → tipo de comprobante)
   ├─ AR: matriz condición emisor × receptor → Clase A/B/C/E/M (RG 5616: condición
   │      IVA del receptor es obligatoria)
   ├─ CR: ¿receptor exonerado? → exigir resolución EXONET vigente → bloque <Exoneracion>
   ├─ CO: escenario (CustomizationID) + resolución de numeración vigente
   └─ BR: ¿mercancía o servicio? → NF-e/NFC-e (SEFAZ) o NFS-e (ADN); CFOP/CST/NCM
   │
   ▼
2. Construir documento (Builder por país) + asignar consecutivo transaccional
   │
   ▼
3. Firmar (SignerPort) — siempre antes de encolar
   │
   ▼
4. Outbox → adaptador del país → autoridad
   ├─ Sync (CO/AR/BR): respuesta inmediata AUTORIZADO/RECHAZADO
   └─ Async (CR, NFS-e BR): 201 + polling hasta estado final
   │
   ▼
5. Post-autorización
   ├─ CO: construir AttachedDocument (factura + ApplicationResponse) → zip → email
   ├─ AR: PDF con CAE + QR Base64 → email
   ├─ CR: enviar PDF + XML firmado + XML respuesta de Hacienda al receptor
   └─ BR: DANFE/DANFSE con protocolo de autorización
   │
   ▼
6. Rechazo → corregir y reemitir (BR: mismo nNF; AR: mismo nro si no consumió CAE);
   ajustes posteriores → Nota Crédito/Débito referenciando el documento original
```

## Flujo 3 — Operación en campo con conectividad intermitente

Principio común: **un dispositivo = una identidad de numeración exclusiva** (PV en AR,
Sucursal+Terminal en CR, série en BR, prefijo/rango en CO). Nunca compartir secuencias
entre terminales offline.

```
ANTES de salir a campo (online, sincronización matutina)
   ├─ AR: descargar CAEA de la quincena (solicitable 5 días antes) → SQLite cifrado
   ├─ BR: certificado A1 + CSC cargados en el dispositivo
   ├─ CR: .p12 + PIN cargados; catálogo CAByS actualizado
   └─ Común: catálogos (clientes, tarifas, tipo de cambio) y rangos de numeración
   │
   ▼
EN CAMPO sin señal
   ├─ AR: emitir con CAEA (tipoCodAut="A"), consecutivo local, PDF+QR, térmica BT
   ├─ CR: clave con Situación=3, firmar local (nunca diferir), ticket provisional
   │      con leyenda de contingencia
   ├─ BR: NFC-e offline firmada local con CSC en QR; NF-e → EPEC (tpEmis=4) si hay
   │      internet mínimo, SVC (6/7) si cayó la SEFAZ
   └─ CO: factura de contingencia (InvoiceTypeCode=03)
   │
   ▼
AL RECUPERAR CONEXIÓN (cola offline → outbox → autoridad)
   ├─ AR: reportar comprobantes CAEA ≤ 8 días corridos tras fin de quincena
   │      (+ declarar PV CAEA sin uso); si no → ARCA suspende CAEA futuros
   ├─ CR: transmitir ≤ 48 h desde que volvió la conexión
   ├─ BR: NFC-e ≤ 24 h (el retraso se tipifica como evasión); EPEC: XML completo ≤ 168 h
   └─ Común: monitorear deadline legal por comprobante y alertar antes de vencer
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

```
Ingreso de la ONG
   ├─ Donación / cuota de afiliación / aporte institucional
   │     → NO se factura en ningún país
   │     ├─ CO: certificado de donación (rep. legal + revisor fiscal; Ley 2380/2024)
   │     ├─ AR: recibo de donación no fiscal (respaldo: certificado de exención ARCA)
   │     ├─ CR: recibo interno; la exoneración de IVA en COMPRAS va vía EXONET
   │     └─ BR: recibo; inmunidad art. 150 VI "c" CF/88
   │
   └─ Venta de bien o servicio (libros, seminarios, consultorías…)
         → SÍ se factura electrónicamente:
         ├─ CO: factura con validación previa (régimen tributario especial no exime)
         ├─ AR: Clase C con CAE (condición IVA Exento)
         ├─ CR: factura normal; si el CLIENTE es el exonerado, bloque <Exoneracion>
         └─ BR: NF-e/NFS-e con CST 40/41 (ICMS), CST 08 (PIS/COFINS),
                ExigibilidadeISS=2/3 + cita legal en infAdic
```

Egresos de la ONG en campo (proveedores informales):
- CO: generar **Documento Soporte electrónico** por cada compra a no obligados
  (jornaleros, transporte rural) — vía OCR del recibo + transmisión a la DIAN.
- AR/CR/BR: exigir comprobante fiscal del proveedor; sin él, el gasto no es deducible y
  debe marcarse en TrustBid como "sin soporte fiscal" para el reporte al donante.
