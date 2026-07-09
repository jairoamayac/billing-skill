# Brasil — Documentos Fiscales Electrónicos (SEFAZ / SPED / Receita Federal)

Gobernanza en 3 niveles: **Federal** (Receita Federal), **Estatal** (27 SEFAZ, ICMS,
coordinadas por CONFAZ/ENCAT vía Ajustes SINIEF) y **Municipal** (5.570 prefeituras, ISS).
No hay un único documento: coexisten varios DF-e según la operación.

## Taxonomía de documentos
| Doc | Modelo | Autoridad | Uso |
|---|---|---|---|
| **NF-e** | 55 | SEFAZ | Venta B2B de mercancías, transferencias, devoluciones, exportación |
| **NFC-e** | 65 | SEFAZ | Venta minorista B2C (sustituye impresoras fiscales ECF) |
| **NFS-e** | DPS/estándar nacional | Municipal → **Portal Nacional (ADN)** | Servicios (LC 116/2003) |
| **CT-e** | 57 | SEFAZ | Flete de carga por terceros (intermunicipal/interestatal) |
| **MDF-e** | 58 | SEFAZ | Manifiesto del vehículo: consolida las NF-e/CT-e del viaje |

**NFS-e Nacional (LC 214/2025)**: municipios obligados a adherirse al ADN desde
**1-ene-2026**; MEI obligados desde sep-2023; **ME/EPP del Simples Nacional emiten solo
por el Emissor Nacional desde 1-sep-2026**. Portal: nfse.gov.br.

## Formatos y versiones
- NF-e/NFC-e: XML layout **4.00**; **NT 2025.002 v1.40** añade grupo `<gIBSCBS>` (IVA
  dual IBS/CBS de la Reforma Tributaria) — obligatorio en producción desde **3-ago-2026**.
- NFS-e: XML **DPS** (esquema NFSe XSD v1.01) → el ADN genera la NFS-e definitiva.
- CT-e/MDF-e: layout 4.00 con validaciones cruzadas contra las NF-e transportadas.

## Certificado digital (ICP-Brasil)
- X.509 v3 de AC autorizada: **e-CNPJ** (empresa) o e-CPF.
- **A1** (.pfx/.p12, software, 12 meses) → estándar para servidores/APIs/móvil.
- **A3** (token/smartcard, 36 meses) → no automatizable.

## Chave de Acesso (44 dígitos)
```
cUF(2) + AAMM(4) + CNPJ(14) + mod(2) + série(3) + nNF(9) + tpEmis(1) + cNF(8) + cDV(1)
```
- `cUF`: código IBGE del estado (35 SP, 33 RJ). `tpEmis`: 1 normal, 4 EPEC, 6 SVC-AN, 7 SVC-RS.
- `cNF`: 8 dígitos aleatorios anti-predicción. `cDV`: **Módulo 11**.
- **CNPJ Alfanumérico (2026)**: convertir cada carácter a número restando 48 a su valor
  ASCII antes del Módulo 11; pesos 2-9 de derecha a izquierda;
  DV = 0 si resto ∈ {0,1}, si no 11 − resto.

## Documentos auxiliares (sin valor fiscal — solo el XML es vinculante)
DANFE (NF-e, código de barras 44 + QR) · DANFE NFC-e (térmico, QR obligatorio, Ley
12.741/2012) · DANFSE · DACTE · **DAMDFE** (porte obligatorio del conductor en retenes).

## Flujo de autorización (validación previa)
1. Generar XML + firma con certificado A1/A3.
2. SOAP sobre TLS ≥1.2 a la SEFAZ del estado (o REST al ADN para NFS-e).
3. Validaciones: esquema XSD, firma/cadena ICP-Brasil, catastro activo (CNPJ + Inscrição
   Estadual/Municipal), reglas de negocio (CFOP, alícuotas, totales).
4. **Autorização de Uso** (protocolo) o **Rejeição** (código de 3 dígitos, ej. 225 =
   fallo de esquema); corregir y retransmitir con el mismo nNF.

## Campos obligatorios principales
- `CNPJ/CPF` (14/11, admite CNPJ alfa 2026) · `IE` (Inscrição Estadual) · `IM` (municipal)
- `CRT`: 1 Simples Nacional, 2 Simples excedido, 3 Régimen Normal
- `chNFe` (44) · `nNF` + `serie` · `dhEmi` (ISO 8601)
- `NCM` (8 dígitos, Mercosur, obligatorio en bienes) · `CFOP` (naturaleza de la operación)
- `CST`/`CSOSN` (situación tributaria ICMS) · `vProd`, `vNF`
- Reforma: `gIBSCBS` (CST IBS/CBS, alícuotas, valores) + `cClassTrib` (6 dígitos, LC 214/25)
- `qrCode` (NFC-e/NFS-e) · `placa` (transporte)

## Verificación pública
- NF-e: https://www.nfe.fazenda.gov.br (44 dígitos → XML completo, estado
  Autorizado/Cancelado/Denegado/Inutilizado + eventos).
- CT-e: https://www.cte.fazenda.gov.br · NFS-e: https://www.nfse.gov.br
- **NFC-e solo se verifica en la SEFAZ del estado emisor** (ej. nfce.fazenda.sp.gov.br),
  vía QR del tique.

## Integración
- SEFAZ standalone (SP, MG, RS) con endpoints propios vs. **SVRS/SVAN** (estados
  delegados) → el software necesita una **matriz dinámica de endpoints WSDL** por estado
  emisor + certificados raíz de cada SEFAZ.
- NFS-e Nacional: API **REST** JSON/XML — `POST /dps` (DPS firmada con A1) → ADN valida,
  genera XML definitivo firmado por el gobierno, retorna 201 + chave + DANFSE.
- Homologación en todas las SEFAZ (DANFE con marca "SEM VALOR FISCAL").
- Middlewares recomendados: **Focus NFe** (REST JSON minimalista), **TecnoSpeed**,
  **Oobj** — unifican NF-e/NFC-e/NFS-e/CT-e/MDF-e y absorben las notas técnicas.

## ONG / Tercer sector
- **Inmunidad constitucional** (art. 150 VI "c" CF/88 + art. 14 CTN): solo cubre
  *impostos* (ISS, ICMS), NO *contribuições* (PIS/COFINS/seguridad social).
- Exención plena de contribuciones federales requiere certificado **CEBAS**.
- La ONG SÍ emite documentos fiscales; la inmunidad se declara en el XML:
  - ICMS: **CST 40** (isenta) o **41** (não tributada).
  - PIS/COFINS: **CST 08** (sin incidencia); sin CEBAS → PIS 1% sobre nómina (MP 2.158-35).
  - NFS-e: `<ExigibilidadeISS>` = 2 (inmunidad) o 3 (exención) + cita constitucional en `<infAdic>`.
- OSCIP (Ley 9.790/1999) habilita convenios públicos pero no crea inmunidades nuevas.

## Operación en campo / interestatal
### DIFAL (ICMS interestatal)
- Venta/traslado entre estados: alícuota interestatal (7% Sur/Sureste→Norte/Nordeste,
  12% inversa) para el origen; la diferencia con la alícuota interna del destino
  (17-22%) es el **DIFAL** (EC 87/2015, LC 190/2022).
- Receptor contribuyente (B2B) → paga el comprador. Receptor no contribuyente (B2C) →
  el emisor retiene y paga con guía GNRE adjunta al tránsito. Aplica también a Simples
  Nacional (STJ).
- Fórmula "base dupla":
  `Base = (ValorOp − ICMS_interestatal) / (1 − alíquota_interna_destino)`;
  `DIFAL = Base × alíquota_destino − ValorOp × alíquota_interestatal`.

### NFS-e multi-municipio
Con el estándar nacional, el ADN identifica el municipio del tomador (código IBGE),
calcula la alícuota de destino y distribuye la retención de ISS automáticamente —
elimina los registros CPOM municipio por municipio.

### Contingencias offline
| Mecanismo | Cuándo | tpEmis | Regularización |
|---|---|---|---|
| **SVC** (SVC-AN / SVC-RS) | SEFAZ de origen caída | 6 / 7 | Automática (sincronización interna) |
| **EPEC** | Sin SEFAZ ni SVC, internet mínimo | 4 | XML completo en **168 h** (7 días), si no → bloqueo de inscripción estatal |
| **Offline NFC-e / MDF-e** | Venta móvil sin conexión | — | Firmar local con A1 + CSC en QR; transmitir en **24 h** — el retraso se tipifica como evasión |

### Traslado de materiales/maquinaria propia a obras
- **CT-e: NO** (solo aplica a fletes de terceros; camión propio no lo genera).
- **NF-e: SÍ** — "Remessa para Prestação de Serviços Fora do Estabelecimento":
  CFOP **5.554/6.554** (activo inmovilizado) o **5.949/6.949** (insumos consumibles);
  sin ICMS (Súmula 166 STJ) → CST 40/41.
- **MDF-e: SÍ** para transporte interestatal E intermunicipal de carga propia (Ajuste
  SINIEF 23/19), vinculando las chaves de las NF-e de remesa. Conductor porta DAMDFE.
- **Encerramento do MDF-e** al llegar a destino es obligatorio e inmediato — omitirlo
  bloquea la placa del vehículo para nuevos manifiestos.

## Fuentes oficiales
- Portal NF-e: https://www.nfe.fazenda.gov.br · Portal NFS-e/ADN: https://www.nfse.gov.br
- Portal CT-e: https://www.cte.fazenda.gov.br
- CONFAZ (Ajustes SINIEF): https://www.confaz.fazenda.gov.br
- NFS-e Nacional 2026 (contexto): https://blog.tecnospeed.com.br/nfse-nacional-tudo/
