# Argentina — Facturación Electrónica (ARCA, ex AFIP)

Autoridad: **ARCA** (Agencia de Recaudación y Control Aduanero, reemplazó a AFIP fines
de 2024). Marco: RG 4291 (esquema general) + RG 1415 · RG 5616/2024 (condición IVA del
receptor obligatoria; tipo de cambio BNA día hábil anterior si se cobra en moneda
extranjera) · RG 5762/2025 (simplifica habilitación de Clase A, deroga RG 1575) ·
RG 5866/2026 (incorpora sectores exceptuados; meta papel-cero al cierre de 2026).

## Obligados
- Responsables Inscriptos, Monotributistas (todas las categorías), Exentos de IVA,
  Exportadores (Clase E).
- Excepción: Monotributo Social / Régimen de Inclusión Social (talonario papel con CAI).
- Talonario manual solo como contingencia.

## Clases de comprobante
| Clase | Emisor | Receptor | IVA |
|---|---|---|---|
| A | Resp. Inscripto | Resp. Inscripto / Monotributista | Discriminado (crédito fiscal). Variantes: "Pago en CBU informada", "Operación sujeta a retención" |
| B | Resp. Inscripto | Consumidor final / Exento / No alcanzado | Implícito, no discriminado |
| C | Monotributista o Exento | Cualquiera | Sin IVA |
| E | Cualquiera | Exterior (exportación) | — |
| M | Resp. Inscripto con inconsistencias | — | Receptor retiene 100% IVA + 6% Ganancias |

## CAE / CAEA
- **CAE**: código de autorización de 14 dígitos, solicitado sincrónicamente por
  comprobante (o lotes de hasta 50 en WSFEv1). Vigencia 10 días corridos. Servicios
  requieren informar período (desde/hasta). Sin CAE válido → sin valor fiscal.
- **CAEA**: autorización anticipada **quincenal** para emitir offline (ver sección campo).

## Flujo técnico
1. **WSAA**: mensaje firmado CMS (PKCS#7) → Ticket de Acceso (token+sign) válido **12 h**.
2. **WSFEv1** `FECAESolicitar` vía SOAP con el TA + metadatos del comprobante.
3. Respuesta sincrónica: `Resultado=A` (aprobado) con CAE + vencimiento, o `R` con
   códigos de error. El emisor genera el PDF con CAE + QR y lo envía al cliente.

Respuesta ejemplo (campos clave): `FeCabResp{Cuit, PtoVta, CbteTipo, Resultado}` y
`FECAEDetResponse{CbteDesde/Hasta, CbteFch, CAE, CAEFchVto, Resultado}`.

## Campos obligatorios principales
| Campo | Regla |
|---|---|
| cuitEmisor | 11 dígitos, igual al CUIT del certificado |
| ptoVta | punto de venta habilitado (1-5 dígitos) |
| tipoCmp | 1=Factura A, 6=B, 11=C, 19=E |
| nroCmp | consecutivo por PV (hasta 8 dígitos) |
| condicionIvaReceptor | obligatorio (RG 5616/2024); determina la clase |
| tipoDocRec / nroDocRec | 80=CUIT, 96=DNI, 99=consumidor final anónimo (`22222222222`) |
| concepto | 1 productos, 2 servicios, 3 ambos |
| fechaServDesde/Hasta, fechaVtoPago | obligatorios si concepto 2 o 3 |
| impNeto, impIVA, impTrib, impOpEx, impOpNoGrav | 13.2 decimales; impTotal debe balancear |
| moneda / ctz | `PES` (ctz 1.000000) o `DOL` etc. con cotización |
| codAut / tipoCodAut / fechaVtoCod | CAE (`E`) o CAEA (`A`) + vencimiento |
| alicuotaIva | por ítem (0.21, 0.105) — requerido en Clase A |

## QR obligatorio (validación clave para TrustBid)
El PDF lleva un QR con la URI:

```
https://www.arca.gob.ar/fe/qr/?p={BASE64(JSON)}
```

JSON nivel 1: `{ver, fecha, cuit, ptoVta, tipoCmp, nroCmp, importe, moneda, ctz,
tipoDocRec, nroDocRec, tipoCodAut ("E"/"A"), codAut}`.

Algoritmo del validador:
1. Decodificar QR (ZXing o similar), aislar el string tras `?p=`, decodificar Base64 → JSON.
2. Cross-check contra el texto del PDF (OCR): importe, CUIT, CAE.
3. Discrepancia → anular aprobación del gasto y alertar auditoría.
4. Constatación oficial en el portal público de ARCA ("Constatación de Comprobantes",
   sin clave fiscal): https://www.arca.gob.ar

## Endpoints
- WSAA producción: `https://wsaa.afip.gov.ar/ws/services/LoginCms?WSDL`
- WSFEv1 producción: `https://servicios1.afip.gob.ar/wsfev1/service.asmx?WSDL`
- WSAA homologación: `https://wsaahomo.afip.gov.ar/ws/services/LoginCms?WSDL`
- WSFEv1 homologación: `https://wswhomo.afip.gov.ar/wsfev1/service.asmx?WSDL`
- WSMTXCA (detalle por ítem, RG 2904); WSFEXv1 (exportación, RG 2758).
- Alta: Clave Fiscal nivel 3 → CSR SHA-256 → "Administración de Certificados Digitales"
  → delegar relación CUIT↔certificado↔servicio (`wsfe`). TLS 1.2.

## ONG / Fundaciones
- Con certificado de exención ARCA: exentas de Ganancias e IVA para servicios ligados a
  su objeto social.
- Condición "IVA Exento" → emiten **Clase C** obligatoriamente (sin alícuota de IVA en
  el request; neto declarado exento).
- Compras: reciben Factura B; el IVA implícito es costo del proyecto, no crédito fiscal.
- **Donaciones y aportes NO se facturan ni llevan CAE** — recibos de donación no
  fiscales respaldados por el certificado de exención (deducibles para el donante).
- Actividades comerciales conexas (venta de libros, capacitaciones aranceladas) → sí,
  Clase C con CAE.

## Operación en campo (cuadrillas / conectividad intermitente)
### Puntos de venta
- Un CUIT puede tener múltiples PV sin límite práctico. **Un PV exclusivo por
  dispositivo/cuadrilla** — compartir PV entre terminales offline produce colisiones de
  numeración que invalidan la facturación.
- Alta vía "Administración de Puntos de Venta y Domicilios" (Clave Fiscal 3).

### CAEA offline (3 fases)
1. **Sincronización (online)**: el servidor central solicita el CAEA por quincena
   (Q1: 1-15; Q2: 16-fin de mes), hasta 5 días corridos antes del inicio. Se distribuye
   cifrado a las bases locales (SQLite) de los dispositivos.
2. **Emisión (offline)**: PV exclusivo modo CAEA, consecutivo local sin saltos, código
   CAEA de 14 dígitos con `tipoCodAut="A"`, PDF+QR generados localmente (impresora
   térmica Bluetooth o email diferido).
3. **Regularización (online)**: reportar todos los comprobantes CAEA (fecha/hora local +
   CAEA) al Régimen de Información dentro de los **8 días corridos** posteriores al fin
   de la quincena. PV CAEA sin uso → presentar "Puntos de Venta no utilizados" en el
   mismo plazo. Incumplir suspende la emisión de nuevos CAEA.

### Traslados y tributos provinciales
- **Remito Clase R** (con CAI) para todo movimiento de mercadería; **COT** en provincias
  con control (ARBA: obligatorio si carga ≥ $7.220.557 o ≥ 4.500 kg; API de ARBA
  disponible). Carta de Porte Electrónica para agro.
- **Convenio Multilateral (IIBB)**: registrar en cada comprobante el "sustento
  territorial" (CP / provincia de ejecución) para distribuir bases imponibles en
  SIFERE Web y evitar doble imposición.

## Fuentes oficiales
- Facturación: https://www.afip.gob.ar/facturacion/ · Web services: https://www.afip.gob.ar/ws/
- Manual WSFEv1 v4.5 y homologación: https://www.afip.gob.ar/fe/ayuda/homologacion_externa.asp
- Especificación QR: https://www.afip.gob.ar/fe/qr/documentos/QRespecificaciones.pdf
- ONG: https://www.afip.gob.ar/entidades-sin-fines-de-lucro/
- Convenio Multilateral: https://www.ca.gob.ar/convenio-multilateral
