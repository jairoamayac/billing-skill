# Costa Rica — Comprobantes Electrónicos (DGT / Ministerio de Hacienda)

Autoridad: **Dirección General de Tributación (DGT)**, Ministerio de Hacienda.
Marco: Ley 9635 · Decreto 44739-H (8-nov-2024) · DGT-R-48-2016 · MH-DGT-RES-000-2024.

**Versión 4.4**: obligatoria desde 1-sep-2025; actualización técnica 4.4 obligatoria el
**1-nov-2026** (STAG y producción disponibles desde 22-abr-2026). Los XML alimentan las
declaraciones precargadas (D-150 IVA) del sistema Hacienda Digital / TRIBU-CR. XML en
versión anterior tras la fecha límite → rechazo inmediato.

## Obligados
- Toda persona física o jurídica con actividad lucrativa (Régimen Tradicional), sin
  umbral mínimo. Registro en el RUT obligatorio.
- Excepción: **Régimen de Tributación Simplificada** (sodas, peluquerías, transporte…)
  → factura física. El comprador del régimen general que quiera deducir esa compra debe
  emitir una **Factura Electrónica de Compra (tipo 08)** auto-declarando la transacción.

## Tipos de comprobante (código en el consecutivo)
| Código | Documento | Uso |
|---|---|---|
| 01 | Factura Electrónica | B2B, respalda crédito fiscal |
| 02 / 03 | Nota de Débito / Crédito | Ajustes +/- sobre factura previa (requieren `InformacionReferencia`) |
| 04 | Tiquete Electrónico | B2C; NO deducible |
| 05/06/07 | Mensajes de Receptor | Aceptación/rechazo del receptor |
| 08 | Factura de Compra | Compras a Régimen Simplificado o a extranjeros no domiciliados |
| 09 | Factura de Exportación | Ventas al exterior |
| 10 | Recibo Electrónico de Pago | Nuevo en 4.4; trazabilidad de cobros a crédito ≤90 días |

## Firma digital
- **XAdES-EPES enveloped** obligatoria ANTES de transmitir.
- Opciones: llave criptográfica de Hacienda (**.p12** + PIN de 4 dígitos, gratuita en ATV,
  automatizable) o Firma Digital del BCCR (smart card, no apta para servidores).
- v4.4: el nodo `ds:Signature` admite hasta **5 firmas** independientes.

## Clave numérica (50 caracteres — identificador único)
```
Clave = País(3) + Día(2) + Mes(2) + Año(2) + CédulaEmisor(12) + Consecutivo(20)
      + Situación(1) + CódigoSeguridad(8)
```
- País: `506`. Fecha: `DDMMYY`. Cédula: rellenar con ceros a la izquierda a 12.
- **Consecutivo (20)** = Sucursal(3) + Terminal(4) + TipoDoc(2) + Secuencial(11).
- **Situación (pos. 42)**: `1` normal · `2` contingencia por fuerza mayor · `3` sin internet.
- Código de seguridad: 8 chars aleatorios, obligatorio y de unicidad validada en 4.4.
- Desde 4.4 la clave es **alfanumérica** (cédulas jurídicas alfanuméricas del Registro Nacional).

## CAByS (clasificación obligatoria por línea)
Código de **13 dígitos** (BCCR + Hacienda) por cada línea de detalle. Hacienda rechaza
XML con CAByS inexistente/inactivo o cuya tarifa de IVA no concuerde con la máxima
autorizada del catálogo.

## Campos obligatorios principales (v4.4)
- `<Clave>` (50) · `<CodigoActividad>` (6, actividad económica del emisor)
- `<NumeroConsecutivo>` (20) · `<FechaEmision>` (ISO 8601 con -06:00)
- Emisor: nombre, identificación (Tipo 01 física / 02 jurídica / 03 DIMEX / 04 NITE),
  `Ubicacion` (códigos BCCR: provincia/cantón/distrito/barrio + señas), email
- Receptor (obligatorio en facturas): tipo 05 = extranjero no domiciliado (nuevo 4.4),
  06 = no contribuyente; número hasta 20 chars; `CodigoActividad` si deduce
- `<CondicionVenta>` (01 contado, 02 crédito → `<PlazoCredito>`, 03 consignación…)
- `<MedioPago>` (4.4 añade códigos SINPE Móvil y plataformas digitales)
- Líneas: `NumeroLinea`, `Codigo` (CAByS), `Cantidad` (12,5), `UnidadMedida`,
  `Detalle` (≤250), `PrecioUnitario` (18,5), `Descuento` (≤5 por línea, 10 tipos en 4.4),
  `SubTotal`, `Impuesto{Codigo (01 IVA), Tarifa (13/4/2/1/0), Monto}`
- Resumen: `CodigoMoneda` (ISO 4217) + `TipoCambio` (venta BCCR del día si extranjera),
  `TotalVenta`, `TotalImpuesto`, `TotalComprobante`
- Exportación: `<PartidaArancelaria>` por línea.
- Precisión decimal: **5 decimales** (18,5).

## Flujo de validación (asíncrono)
1. Generar XML + clave 50 + firma XAdES-EPES (.p12).
2. `POST /recepcion/v1/recepcion` con JSON:
   `{clave, fecha, emisor{tipo,numero}, receptor{tipo,numero}, comprobanteXml(Base64)}`.
3. Respuesta `201 Created` + header `Location` → **polling** `GET /recepcion/{clave}`.
4. Validaciones de negocio en lote: firma, estado RUT, CAByS vs tarifas, coherencia de
   totales. `ind-estado`: `aceptado` / `aceptado parcialmente` / `rechazado`;
   `respuesta-xml` (Base64) = acuse firmado por la DGT (único respaldo legal).
5. Enviar al receptor: PDF + XML firmado + XML de respuesta.
6. El receptor emite **Mensaje de Receptor** (tipo 05/06/07) dentro de los 8 días hábiles
   del mes siguiente — sin él no puede acreditar IVA ni deducir el gasto.

## Autenticación (OAuth 2.0 / OIDC)
- Grant: `password`. Body: `grant_type=password`, `client_id` = `api-prod` | `api-stag`,
  `username` = cédula a 12 chars, `password` = contraseña de API generada en ATV.
- Token válido **12 h**, header `Authorization: Bearer {token}`.
- IDP producción: `https://idp.comprobanteselectronicos.go.cr/auth/realms/rut/protocol/openid-connect/token`
- IDP staging: `https://idp.comprobanteselectronicos.go.cr/auth/realms/rut-stag/protocol/openid-connect/token`
- API producción: `https://api.comprobanteselectronicos.go.cr/recepcion/v1/`
- API sandbox: `https://api.comprobanteselectronicos.go.cr/recepcion-sandbox/v1/`

## Verificación pública (para el validador TrustBid)
- `GET https://api.comprobanteselectronicos.go.cr/recepcion/v1/recepcion/{clave}` →
  `{clave, fecha, ind-estado, respuesta-xml}`.
- Portal ATV "Consulta de Comprobantes"; terceros: VerificatuFactura.com.
- Bibliotecas: `facturacr` (Ruby, open source), pasarelas comerciales JSON→XML.

## ONG / Fundaciones (EXONET)
- **No hay exención automática** por figura jurídica. La entidad tramita la exoneración
  de IVA en **EXONET** (Dirección General de Hacienda) → resolución con clave `AL-`/`EX-`.
- El **proveedor** que vende a la ONG exonerada debe incluir el bloque `<Exoneracion>`
  dentro de `<Impuesto>` en CADA línea:
  `TipoDocumentoEX1` (02 = EXONET), `NumeroDocumento` (ej. `AL-00460853-20`),
  `NombreInstitucion`, `FechaEmisionEX`, `TarifaExonerada`, `MontoExoneracion`.
- Total: `TarifaExonerada == Tarifa` y `MontoExoneracion == Monto` (IVA neto 0).
  Parcial (13% → 2%): `TarifaExonerada = 11.00`, monto proporcional, IVA remanente se cobra.
- El proveedor debe verificar vigencia y porcentaje de la resolución antes de vender.

## Operación en campo / offline
- **Sucursal(3) + Terminal(4)** en el consecutivo: hasta 999 sucursales × 9.999
  terminales. Asignar combinación exclusiva por dispositivo/cuadrilla; cada terminal
  incrementa su propio secuencial de 11 dígitos → sin colisiones. No se requiere
  timbraje por terminal; solo las sucursales físicas se registran en el RUT/ATV.
- **Sin internet → Situación 3** (pos. 42 de la clave): firmar localmente con el .p12
  (nunca diferir la firma), entregar ticket provisional (impresora térmica BT) con la
  leyenda de contingencia, y transmitir en lote dentro de **48 horas** tras recuperar
  conexión.
- El portal gratuito ATV no sirve en campo (sin offline, sin UI móvil) → app propia con
  SQLite + firmado XAdES embebido + cola cifrada.
- **No existe guía de despacho/remito fiscal para tránsito interno** — solo documentos
  aduaneros para comercio exterior. Recomendado portar facturas de compra de los equipos
  y albarán interno del ERP.
- Domicilio fiscal: obra prolongada con bodega/oficina fija → alta de sucursal en RUT;
  vigilar patentes municipales del cantón y coherencia del código de actividad económica.

## Fuentes oficiales
- Ministerio de Hacienda / DGT: https://www.hacienda.go.cr
- Esquemas XSD 4.4: https://tribunet.hacienda.go.cr
- BCCR (CAByS, tipo de cambio): https://www.bccr.fi.cr
- EXONET: portal del Ministerio de Hacienda
