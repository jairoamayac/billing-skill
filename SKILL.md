---
name: facturacion-electronica-latam
description: Guía técnica y flujos de integración de facturación electrónica para Colombia (DIAN), Argentina (ARCA/ex-AFIP), Costa Rica (Hacienda/DGT) y Brasil (SEFAZ/SPED), orientada a la plataforma TrustBid. Usa esta skill siempre que el usuario pregunte por facturas electrónicas, CUFE, CAE/CAEA, clave numérica de 50 dígitos, chave de acesso, NF-e/NFC-e/NFS-e, validación de comprobantes fiscales, parsers/OCR de facturas, integración con DIAN/ARCA/Hacienda/SEFAZ, documentos soporte, tratamiento fiscal de ONG/ESAL, o facturación offline/en campo en cualquiera de estos países — incluso si no menciona la palabra "skill" o el país explícitamente pero el contexto es LatAm.
---

# Facturación Electrónica LatAm para TrustBid

Skill de referencia para diseñar, implementar y validar integraciones de facturación
electrónica en los 4 países soportados por TrustBid. Cada país tiene su carpeta con la
referencia técnica completa; la carpeta `flujos/` contiene la arquitectura de integración
y los flujos operativos comunes (diseñados con patrones de la skill
`arquitectura-de-software`: Adapter por país, Strategy, Outbox, Circuit Breaker, cola
offline con sincronización diferida).

## Cómo usar esta skill

1. **Identifica el país** de la transacción o integración. Lee SOLO la referencia del país
   relevante (progressive disclosure — no cargues las 4 si no hace falta):
   - [colombia/reference.md](colombia/reference.md) — DIAN, UBL 2.1, CUFE SHA-384, AttachedDocument
   - [argentina/reference.md](argentina/reference.md) — ARCA, SOAP WSFEv1, CAE/CAEA, QR Base64-JSON
   - [costa-rica/reference.md](costa-rica/reference.md) — Hacienda/DGT, XML v4.4, clave de 50, API REST OAuth2
   - [brasil/reference.md](brasil/reference.md) — SEFAZ/SPED, NF-e/NFC-e/NFS-e/CT-e/MDF-e, chave de 44
2. **Si la tarea es de diseño de sistema** (parser, validador, cola offline, multi-país),
   lee [flujos/arquitectura-integracion.md](flujos/arquitectura-integracion.md) primero.
3. **Si la tarea es operativa** (validar un gasto, flujo ONG, contingencia sin internet,
   cuadrillas en campo), lee [flujos/flujos-operativos.md](flujos/flujos-operativos.md).

## Mapa comparativo rápido

| Aspecto | Colombia | Argentina | Costa Rica | Brasil |
|---|---|---|---|---|
| Autoridad | DIAN | ARCA (ex AFIP) | DGT / Min. Hacienda | SEFAZ estatales + Receita Federal + municipios |
| Formato | XML UBL 2.1 | SOAP/XML propietario (WSFEv1) | XML propio v4.4 | XML propio (layout 4.00 / DPS NFS-e) |
| Protocolo | SOAP (WCF) | SOAP + WSAA (ticket 12h) | REST + OAuth 2.0 (token 12h) | SOAP (SEFAZ) + REST (NFS-e Nacional) |
| Identificador único | CUFE (96 chars, SHA-384) | CAE (14 dígitos) | Clave numérica (50 chars) | Chave de Acesso (44 dígitos, Módulo 11) |
| Firma | XAdES-EPES (cert. ONAC) | CMS/PKCS#7 en WSAA | XAdES-EPES (.p12 Hacienda o BCCR) | XMLDSig con cert. ICP-Brasil A1/A3 |
| Validación | Previa, sincrónica | Previa, sincrónica (CAE) | Asíncrona (POST 201 + polling) | Previa, sincrónica (Autorização de Uso) |
| Modo offline | Contingencia (tipo 03) | CAEA quincenal | Situación 3 (48 h para transmitir) | SVC / EPEC (168 h) / offline NFC-e (24 h) |
| Verificación pública | catalogo-vpfe.dian.gov.co | arca.gob.ar (QR → JSON Base64) | GET /recepcion/{clave} | nfe.fazenda.gov.br, nfse.gov.br, portales SEFAZ |

## Reglas transversales para TrustBid

- **El XML es el documento legal**, nunca el PDF. El PDF/DANFE/representación gráfica es
  solo un resumen visual. El parser debe priorizar XML; usar OCR solo cuando únicamente
  exista imagen/PDF, y en ese caso validar contra el portal público del país.
- **Toda factura entrante se verifica contra la autoridad fiscal** antes de aprobar el
  gasto en el pipeline financiero (ver flujo "Validación de gasto" en flujos-operativos.md).
- **Cross-check obligatorio**: identificador único + emisor + total extraídos del documento
  deben coincidir con la respuesta de la autoridad. Cualquier discrepancia → rechazo del
  gasto y alerta de auditoría.
- **Donaciones no se facturan en ningún país**; solo las actividades comerciales de una
  ONG generan factura. Cada referencia de país tiene su sección ONG con el detalle
  (certificados de donación en CO, Clase C en AR, EXONET en CR, inmunidad + CST en BR).
- **Montos**: usar decimales con punto, sin separador de miles, con la precisión que exige
  cada esquema (2 decimales CO/AR/BR, 5 decimales CR). Nunca floats binarios en cálculo
  de impuestos — usar decimal de precisión arbitraria.
- **Fechas**: ISO 8601 con offset horario local (CO -05:00, AR -03:00, CR -06:00, BR -03:00
  a -05:00 según estado).

## Estado normativo (verificado julio 2026)

- Colombia: Anexo Técnico 1.9 (Res. 000165/2023), Res. 000202/2025 y 000011/2026 vigentes.
- Argentina: ARCA reemplazó a AFIP; RG 5616/2024 (condición IVA receptor obligatoria),
  RG 5762/2025 y RG 5866/2026 vigentes.
- Costa Rica: versión 4.4 obligatoria desde 1-sep-2025; actualización técnica 4.4
  obligatoria el 1-nov-2026 (STAG y producción disponibles desde 22-abr-2026); alimenta
  declaraciones precargadas de TRIBU-CR.
- Brasil: NFS-e Nacional obligatoria para municipios desde 1-ene-2026 (LC 214/2025);
  ME/EPP del Simples Nacional emiten solo por el Emissor Nacional desde 1-sep-2026;
  NT 2025.002 v1.40 (grupo gIBSCBS de la reforma tributaria) obligatoria en producción
  desde 3-ago-2026.

Al usar cifras normativas (fechas, umbrales, resoluciones) en producción, verifica contra
la fuente oficial listada en cada referencia — los regímenes fiscales cambian cada año.
