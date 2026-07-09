# Requisitos de habilitación para facturar electrónicamente — por país

Checklist práctico de lo que un tenant de TrustBid necesita tener/tramitar ANTES de
poder emitir su primera factura electrónica en cada país. Complementa las referencias
técnicas (`colombia/reference.md`, etc.), que asumen que la habilitación ya existe.

## Colombia (DIAN)

**Requisitos legales/administrativos**
1. Estar inscrito en el **RUT** (Registro Único Tributario) con la responsabilidad
   fiscal correcta (IVA, INC, RST, etc.).
2. Solicitar y obtener autorización de **numeración/resolución de facturación** ante la
   DIAN (prefijo + rango de consecutivos + vigencia, hoy indefinida salvo pérdida de
   confianza).
3. Registrarse como **facturador electrónico** en el sistema de la DIAN (proceso de
   habilitación con set de pruebas: mínimo un lote de facturas de prueba aprobadas en
   el ambiente de habilitación antes de pasar a producción).

**Requisitos técnicos**
1. **Certificado digital** de persona jurídica/natural, vigente, emitido por entidad
   acreditada por la **ONAC** (no cualquier CA sirve).
2. Software o proveedor tecnológico (PT) capaz de generar XML UBL 2.1, calcular el
   CUFE (SHA-384) y firmar XAdES-EPES.
3. Conexión SOAP a los WSDL de habilitación/producción de la DIAN, o contrato con un
   PT (Alegra/Matias, TFHKA, Siigo) que abstraiga eso vía REST.
4. Clave técnica de 40 caracteres hex que la DIAN entrega al software habilitado (se
   usa en el cálculo del CUFE) — es distinta por software/NIT.

**Costos aproximados**: PT desde ~$10.000-30.000 COP/mes (≤120 docs/año) hasta
~$100.000 COP/mes (≤1.500 docs/año). Certificado digital: costo anual variable según
la CA (no incluido en el plan del PT normalmente).

**Tiempo típico de habilitación**: 1-3 semanas si el PT ya tiene la integración lista;
más si se construye la integración SOAP propia.

## Argentina (ARCA, ex AFIP)

**Requisitos legales/administrativos**
1. **CUIT** activo y **Clave Fiscal nivel 3** (la 1 y 2 no alcanzan para delegar
   servicios web).
2. Alta de al menos un **Punto de Venta** para el canal "Factura Electrónica" en el
   servicio "Administración de Puntos de Venta y Domicilios" (asociado a un domicilio
   comercial declarado en el RUT/domicilio fiscal).
3. Habilitación de Clase A (si corresponde) — simplificada desde la RG 5762/2025, ya
   no requiere las viejas garantías patrimoniales previas.

**Requisitos técnicos**
1. Generar un **CSR** (Certificate Signing Request) firmado con SHA-256 localmente.
2. Subirlo al servicio "Administración de Certificados Digitales" en el portal con
   Clave Fiscal → descargar el certificado `.crt` firmado por ARCA.
3. **Delegar la relación**: asociar el CUIT emisor con ese certificado y el alias
   impositivo para el servicio de negocio (`wsfe`, `wsfex`, `wsmtxca` según aplique).
4. Software capaz de hablar SOAP/WSDL, obtener el Ticket de Acceso vía WSAA (firma
   CMS/PKCS#7) y llamar `FECAESolicitar` en WSFEv1.
5. Probar primero en homologación (`wsaahomo`/`wswhomo`) con un certificado de testing
   tramitado en el autoservicio "WSASS" — **no es el mismo certificado de producción**.
6. Si va a operar offline: tramitar la habilitación de **CAEA** para los puntos de
   venta de campo (trámite adicional, no automático).

**Tiempo típico**: el alta del certificado y la delegación pueden resolverse en
horas/días; la habilitación de Clase A puede tardar más si ARCA pide validación
adicional del domicilio.

## Costa Rica (Ministerio de Hacienda / DGT)

**Requisitos legales/administrativos**
1. Inscripción en el **RUT** costarricense (persona física o jurídica) con la
   actividad económica correcta (`CodigoActividad`, 6 dígitos).
2. Registrarse en el portal **ATV** (Administración Tributaria Virtual).
3. Generar en ATV el usuario y contraseña de **API de comprobantes electrónicos**
   (distintos del usuario/clave de login web de ATV).

**Requisitos técnicos**
1. Elegir mecanismo de firma:
   - **Llave criptográfica de Hacienda** (.p12, gratis, se descarga desde ATV, PIN de
     4 dígitos) — la opción práctica para automatización en servidor.
   - o Firma Digital del BCCR (smart card) — requiere lector físico, no apta para
     backend.
2. Software capaz de: construir el XML v4.4, calcular la clave de 50 caracteres,
   firmar XAdES-EPES, obtener token OAuth2 (`grant_type=password`) y hacer
   `POST /recepcion` + polling `GET /recepcion/{clave}`.
3. Tener actualizado el **catálogo CAByS** vigente (Hacienda lo publica y actualiza
   periódicamente; rechaza códigos inactivos).
4. Probar primero en el ambiente **Staging** (`recepcion-sandbox`, IDP `rut-stag`)
   antes de producción.
5. Si la operación es multi-sucursal: dar de alta cada sucursal física en el RUT/ATV
   (asignación de código de sucursal 001-999); los terminales/dispositivos NO
   requieren alta individual.

**Nota de calendario**: si el desarrollo empieza después de nov-2026, construir
directamente contra el esquema v4.4 actualizado — la versión anterior será rechazada.

## Brasil (SEFAZ / Receita Federal / Municipios)

**Requisitos legales/administrativos**
1. **CNPJ** (o CPF para productor rural/persona física) activo ante la Receita
   Federal.
2. **Inscrição Estadual (IE)** ante la SEFAZ del estado, si va a emitir NF-e/NFC-e
   (venta de mercancías).
3. **Inscrição Municipal (IM)** ante la prefeitura, si va a emitir NFS-e (servicios).
4. Definir el **CRT** (régimen tributario: Simples Nacional, Simples excedido, o
   Régimen Normal) — determina el CST/CSOSN a usar en cada operación.
5. Si va a transportar carga propia entre estados: verificar que no necesite alta
   adicional específica de transportista (solo aplica a servicios de flete a terceros).

**Requisitos técnicos**
1. **Certificado digital ICP-Brasil** e-CNPJ o e-CPF:
   - Tipo **A1** (.pfx/.p12, 12 meses) para automatización en servidor/API — es lo
     que necesita casi cualquier integración TrustBid.
   - Tipo A3 (token físico) solo si hay operación manual en un puesto fijo.
2. Determinar la arquitectura de endpoints: cada estado tiene su propia SEFAZ
   (standalone) o delega en SVRS/SVAN — el software necesita la tabla de WSDL por
   `cUF` antes de emitir la primera NF-e.
3. Para NFS-e: verificar si el municipio del emisor ya migró al **ADN nacional**
   (obligatorio desde ene-2026) o todavía mantiene emisor propio que reenvía al ADN —
   afecta el endpoint a usar.
4. Probar en **homologação** de la SEFAZ correspondiente antes de producción (el
   certificado real se usa también en homologación, pero los documentos no tienen
   valor fiscal).
5. Si emite NFC-e/MDF-e con operación móvil: definir el CSC (Código de Segurança do
   Contribuinte) usado para el QR offline — se solicita en el portal de la SEFAZ.

**Complejidad adicional**: a diferencia de los otros tres países, Brasil no tiene un
único "botón de habilitación" — la combinación de IE + IM + certificado + endpoint por
estado hace que la mayoría de integradores usen un middleware (Focus NFe, TecnoSpeed,
Oobj) para no mantener 27 configuraciones de SEFAZ a mano.

## Checklist genérico para el onboarding de un tenant en TrustBid

Antes de activar facturación para un tenant nuevo, el flujo de alta debe capturar:

| Dato | CO | AR | CR | BR |
|---|---|---|---|---|
| Identificación fiscal | NIT | CUIT | Cédula jur./física | CNPJ/CPF |
| Registro tributario | RUT | Clave Fiscal nivel 3 | RUT + ATV | IE (estatal) + IM (municipal) |
| Certificado digital | X.509 (ONAC) | X.509 (CSR SHA-256 vía ARCA) | .p12 Hacienda o BCCR | ICP-Brasil A1/A3 |
| Credencial de API | — (solo SOAP o vía PT) | Ticket WSAA (renovable c/12h) | Usuario+password API → OAuth token | — (certificado firma cada request) |
| Numeración | Prefijo + rango DIAN | Punto de Venta | Sucursal + Terminal | Série + nNF |
| Ambiente de pruebas | Habilitación DIAN | Homologación (wswhomo) | Staging (recepcion-sandbox) | Homologação por SEFAZ |
| Vigencia típica cert. | Según CA | — | — | A1: 12 meses / A3: 36 meses |

No activar el interruptor de "producción" para un tenant hasta que estos 7 campos
estén completos y se haya emitido al menos un comprobante de prueba aprobado en el
ambiente correspondiente.

## Fuentes de referencia para trámites de habilitación
- DIAN — Registro y habilitación: https://micrositios.dian.gov.co/sistema-de-facturacion-electronica/proceso-de-registro-y-habilitacion-como-facturador-electronico/
- ARCA — Clave Fiscal y certificados: https://www.afip.gob.ar/ws/
- Hacienda CR — ATV y API: https://www.hacienda.go.cr
- Receita Federal / SEFAZ — habilitación por estado: portales estatales de cada SEFAZ (no hay portal único)
