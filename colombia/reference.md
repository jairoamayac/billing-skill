# Colombia — Facturación Electrónica (DIAN)

Autoridad: **DIAN** (Dirección de Impuestos y Aduanas Nacionales).
Marco: Estatuto Tributario arts. 615, 616-1, 618 · Resolución 000165/2023 (Anexo Técnico
1.9) · Res. 000202/2025 (simplifica datos del comprador presencial) · Res. 000011/2026.

## Obligados
- Todas las personas jurídicas que vendan bienes/servicios.
- Responsables de IVA e INC sin importar ingresos.
- Personas naturales con ingresos brutos > 3.500 UVT (UVT 2026 = $52.374 → ~$183.309.000 COP).
- Régimen Simple de Tributación (RST); importadores; casos especiales.
- Exceptuados: naturales no responsables de IVA/INC bajo el umbral, Juntas de Acción
  Comunal sin devolución de IVA, prestadores desde el exterior sin residencia fiscal.

## Formato técnico
- **XML UBL 2.1** obligatorio (JSON solo en capas de integración privadas).
- 5 tipos de documento: `Invoice`, `CreditNote`, `DebitNote`, `ApplicationResponse`
  (respuesta/eventos DIAN), `AttachedDocument` (contenedor de entrega).
- Secciones clave del `Invoice`:
  - `cbc:UBLVersionID` = `2.1`, `cbc:CustomizationID` (escenario: 10 nacional, 11 exportación), `cbc:ProfileID` = `DIAN 2.1`
  - `ext:UBLExtensions` → `sts:InvoiceControl` (resolución de numeración, prefijo, rango),
    `sts:SoftwareProvider`, `sts:QRCode`, `ds:Signature`
  - `cac:AccountingSupplierParty` / `cac:AccountingCustomerParty`
  - `cac:InvoiceLine` (líneas), `cac:TaxTotal` (impuestos), `cac:LegalMonetaryTotal` (totales)

## Firma digital
- **XAdES-EPES enveloped** (ETSI TS 101 903 v1.2.2/1.3.2/1.4.1), certificado de entidad
  acreditada por **ONAC**.
- Política de firma: `https://facturaelectronica.dian.gov.co/politicadefirma/v2/politicadefirmav2.pdf`
- Ubicación: `/fe:Invoice/ext:UBLExtensions/ext:UBLExtension[2]/ext:ExtensionContent/ds:Signature`
  con cadena completa en `ds:X509Certificate` (Base64).

## CUFE (identificador único)
Hash **SHA-384** (96 chars hex) de la concatenación estricta sin separadores:

```
CUFE = SHA-384(NumFac + FecFac + HorFac + ValFac + CodImp1 + ValImp1 + CodImp2 + ValImp2
             + CodImp3 + ValImp3 + ValTot + NitOFE + NumAdq + ClTec + TipoAmbie)
```

| Campo | Formato / regla |
|---|---|
| NumFac | prefijo + consecutivo sin separadores (`FEP10023`) |
| FecFac | `YYYY-MM-DD` |
| HorFac | `HH:mm:ss-05:00` |
| ValFac | base antes de impuestos, `1500000.00` |
| CodImp1/2/3 | fijos `01` IVA, `02` INC, `03` ICA |
| ValImp1/2/3 | acumulado por impuesto, `0.00` si no aplica |
| ValTot | PayableAmount, 2 decimales |
| NitOFE / NumAdq | sin puntos, guiones ni DV; adquiriente anónimo = `222222222222` |
| ClTec | clave técnica de 40 chars hex que entrega la DIAN |
| TipoAmbie | `1` producción, `2` pruebas |

Se almacena en `cbc:UUID` con `@schemeName="CUFE-SHA384"`. Recalcular el hash detecta
alteraciones del XML.

## Campos obligatorios principales (XPath UBL 2.1)
- `cbc:ID` (prefijo+consecutivo) · `cbc:UUID` (CUFE) · `cbc:IssueDate` / `cbc:IssueTime`
- `cbc:InvoiceTypeCode` (01 venta, 02 exportación, 03 contingencia)
- `cbc:DocumentCurrencyCode` (ISO 4217, default `COP`)
- `sts:InvoiceAuthorization` + `sts:Prefix` + `sts:From`/`sts:To` (resolución de numeración)
- Emisor: `CompanyID` (NIT sin DV; DV en `@schemeID`), `PartyName/Name` (igual al RUT),
  código de responsabilidad fiscal (`O-48` responsable IVA…)
- Adquiriente: `CompanyID` (o `222222222222`), nombre (o "Consumidor Final"); dirección
  física NO exigible en venta presencial (Res. 0202/2025); email opcional
- `cac:PaymentMeans/cbc:ID` (1 contado, 2 crédito) + `PaymentMeansCode` (10 efectivo,
  42 consignación, 47 transferencia)
- `cac:TaxTotal/cac:TaxSubtotal` con `TaxScheme/cbc:ID` (01 IVA, 02 INC, 03 ICA) y
  `cbc:Percent` (19.00, 5.00, 0.00)
- `LegalMonetaryTotal`: `LineExtensionAmount`, `TaxExclusiveAmount`, `TaxInclusiveAmount`, `PayableAmount`
- Líneas: `cbc:ID` secuencial desde 1, `InvoicedQuantity@unitCode`, `Item/Description`,
  `Price/PriceAmount`, `TaxTotal` por línea

## Flujo de validación previa
1. Generar XML UBL 2.1 + CUFE + firma XAdES-EPES.
2. Comprimir `.zip`, codificar Base64, enviar sincrónicamente al SOAP de la DIAN.
3. DIAN aplica +400 reglas; rechazo inmediato o `ApplicationResponse` firmado
   ("Documento validado por la DIAN", `ValidationResultCode` = `02`).
4. Empaquetar factura + ApplicationResponse en **AttachedDocument**, zip, enviar al
   email del adquirente.

## Parser TrustBid — AttachedDocument
- Si el XML raíz es `<AttachedDocument>`: extraer la factura del CDATA en
  `/AttachedDocument/cac:ParentDocumentLineReference/cac:DocumentReference/cac:Attachment/cac:ExternalReference/cbc:Description`.
- Validar contra XSD oficiales `DIAN_UBL.xsd` y `DIAN_UBL_Structures.xsd`.
- Estado de validación en `cac:ResultOfVerification` (código `02` = validado).

## Verificación pública
- Portal: `https://catalogo-vpfe.dian.gov.co/document/search` (sin autenticación).
- Consulta directa por CUFE/CUDE:
  `https://catalogo-vpfe.dian.gov.co/document/searchqr?documentkey={CUFE}`
- El QR impreso codifica un enlace con metadatos (número, NITs, totales, CUFE) — el OCR
  debe decodificarlo y confirmar estado "Documento validado por la DIAN".

## Integración
- Solo **SOAP** (WCF, WS-Security, XML Base64). Sin API REST oficial.
  - Habilitación: `https://vpfe-hab.dian.gov.co/WcfDianCustomerServices.svc?wsdl`
  - Producción: `https://vpfe.dian.gov.co/WcfDianCustomerServices.svc?wsdl`
- Proveedores tecnológicos (REST JSON → UBL firmado): Alegra/Matias API, The Factory HKA,
  Siigo API. Costos referenciales: $10-30 mil COP/mes (≤120 docs/año) hasta ~$100 mil
  COP/mes (≤1.500 docs/año).

## ONG / ESAL (Régimen Tributario Especial)
- **Donaciones y cuotas de afiliación NO se facturan** — se legalizan con certificados de
  donación (Ley 2380/2024, arts. 257-258 ET; descuento 25%-37% en renta del donante).
- **Venta de bienes/servicios** (libros, seminarios, consultorías) SÍ obliga factura
  electrónica con validación previa.
- Donación en especie de una empresa a la ONG: el **donante** factura a nombre de la ONG
  y liquida IVA (art. 421 ET, Oficio 6107/2024); exentos alimentos/aseo a bancos de
  alimentos (art. 424-9 ET).
- Gastos con proveedores informales (jornaleros, transporte rural): la ONG genera y
  transmite el **Documento Soporte electrónico** en adquisiciones a no obligados a
  facturar. El OCR de TrustBid debe extraer los campos mínimos del recibo informal y
  estructurar ese XML.

## Fuentes oficiales
- Anexo Técnico 1.9: https://www.dian.gov.co/impuestos/factura-electronica/Documents/Anexo-Tecnico-Factura-Electronica-de-Venta-vr-1-9.pdf
- Res. 000165/2023: https://normograma.dian.gov.co/dian/compilacion/docs/resolucion_dian_0165_2023.htm
- Res. 000202/2025: https://normograma.dian.gov.co/dian/compilacion/docs/resolucion_dian_0202_2025.htm
- Portal de consulta: https://catalogo-vpfe.dian.gov.co/
- UBL 2.1: https://docs.oasis-open.org/ubl/os-UBL-2.1/
- Proveedores tecnológicos: https://micrositios.dian.gov.co/sistema-de-facturacion-electronica/documentacion-tecnica/
