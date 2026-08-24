# NIBSS NPS ISO 20022 Reference — pain.001 / pacs.008 / pacs.002 / camt.053

Sample XML + XSD for the four message types relevant to a NIBSS NPS simulator,
adapted for Nigerian rail conventions. These are **not** official ISO 20022 or
NIBSS-published schemas — NIBSS's actual NPS XSDs are gated behind sandbox
registration (see the companion `README.md` in this project for links). These
are built from the public ISO 20022 message structure, with fields renamed or
added where Nigerian domestic rail conventions genuinely diverge — each
deviation is called out with its reason. Verify against real NIBSS sandbox
payloads once you have access, before treating this as a certification
reference.

## Message flow this covers

```
Customer/originator system            pain.001  →  Sending bank
Sending bank                          pacs.008  →  NPS switch (simulated)
NPS switch (simulated)                pacs.002  →  Sending bank
NPS switch / receiving bank (EOD)     camt.053  →  Customer/originator system
```

- **pain.001** — Customer Credit Transfer Initiation. The instruction a
  customer-facing system (e.g. `credora-backend`) sends to its own bank/gateway
  to request an outbound transfer. JSON is fine for this hop in your own
  system (Credora ↔ its aggregator) — pain.001 XML matters once that hop is
  modeled as talking to an actual bank/ISO 20022 gateway.
- **pacs.008** — FI to FI Customer Credit Transfer. The interbank leg: bank to
  switch. This is what your simulator receives.
- **pacs.002** — Payment Status Report. What your simulator sends back:
  accepted / rejected / pending, referencing the original transaction.
- **camt.053** — Bank to Customer Statement. End-of-day (or on-demand)
  statement covering settled entries — useful if your simulator also needs to
  produce reconciliation statements.

## Naming deviations from generic ISO 20022, and why

| ISO 20022 convention | Change made here | Reason |
|---|---|---|
| `IBAN` | `NUBAN` (10-digit) in account ID | Nigerian bank accounts use NUBAN, not IBAN — keeping IBAN would be actively wrong, not just simplified |
| `BIC` / SWIFT-style `ClrSysMmbId` | `InstnCode` | NIBSS routes by its own bank institution/sort codes, not SWIFT BICs, for domestic transfers |
| *(no ISO 20022 equivalent)* | `SessionId` added to pacs.008/pacs.002 | Real NIP/NPS assigns a switch-level session reference distinct from the customer's own `EndToEndId` — a genuine NIP/NPS concept being carried forward, not a generic ISO 20022 field |
| *(no ISO 20022 equivalent)* | `NameEnquiryRef` added to pacs.008 | NIP/NPS requires a prior Name Enquiry call before a credit transfer; carrying that reference forward mirrors real switch behavior |
| `StsRsnInf/Rsn/Cd` only | Added parallel `NipRspCd` | Keeps the standard ISO 20022 external status-reason code *and* a raw switch response code field, matching how NIP/NPS responses actually carry both a generic code and a rail-specific one |

Everything else (`GrpHdr`, `PmtId`, `IntrBkSttlmAmt`, `Dbtr`/`Cdtr` structure,
`TxSts`, `RmtInf`, etc.) keeps standard ISO 20022 naming since it carries real
structural meaning worth preserving for interoperability and for anyone
reading the payloads who already knows ISO 20022.

---

## 1. pain.001 — Customer Credit Transfer Initiation

### XSD (`pain.001.simplified.xsd`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns="urn:nibss:nps:pain.001.001.13"
           targetNamespace="urn:nibss:nps:pain.001.001.13"
           elementFormDefault="qualified">

  <xs:element name="Document" type="DocumentType"/>

  <xs:complexType name="DocumentType">
    <xs:sequence>
      <xs:element name="CstmrCdtTrfInitn" type="CstmrCdtTrfInitnType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CstmrCdtTrfInitnType">
    <xs:sequence>
      <xs:element name="GrpHdr" type="GrpHdrType"/>
      <xs:element name="PmtInf" type="PmtInfType" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="GrpHdrType">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
      <xs:element name="NbOfTxs" type="xs:positiveInteger"/>
      <xs:element name="CtrlSum" type="xs:decimal"/>
      <xs:element name="InitgPty" type="PartyType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PmtInfType">
    <xs:sequence>
      <xs:element name="PmtInfId" type="xs:string"/>
      <xs:element name="PmtMtd" type="xs:string" fixed="TRF"/>
      <xs:element name="BtchBookg" type="xs:boolean"/>
      <xs:element name="NbOfTxs" type="xs:positiveInteger"/>
      <xs:element name="CtrlSum" type="xs:decimal"/>
      <xs:element name="PmtTpInf" type="PmtTpInfType"/>
      <xs:element name="ReqdExctnDt" type="xs:date"/>
      <xs:element name="Dbtr" type="PartyType"/>
      <xs:element name="DbtrAcct" type="AcctType"/>
      <xs:element name="DbtrAgt" type="AgtType"/>
      <xs:element name="CdtTrfTxInf" type="CdtTrfTxInfType" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PmtTpInfType">
    <xs:sequence>
      <!-- SvcLvl Cd: NURG = non-urgent/batch, INST = instant (NIP/NPS real-time rail) -->
      <xs:element name="SvcLvl">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Cd" type="xs:string"/>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CdtTrfTxInfType">
    <xs:sequence>
      <xs:element name="PmtId" type="PmtIdType"/>
      <xs:element name="Amt" type="AmtType"/>
      <!-- ChrgBr: SHAR / DEBT / CRED — who bears the transfer fee -->
      <xs:element name="ChrgBr" type="xs:string"/>
      <xs:element name="CdtrAgt" type="AgtType"/>
      <xs:element name="Cdtr" type="PartyType"/>
      <xs:element name="CdtrAcct" type="AcctType"/>
      <xs:element name="RmtInf" type="RmtInfType" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PmtIdType">
    <xs:sequence>
      <xs:element name="InstrId" type="xs:string"/>
      <xs:element name="EndToEndId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AmtType">
    <xs:sequence>
      <xs:element name="InstdAmt" type="CurrencyAmountType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmountType">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

  <xs:complexType name="PartyType">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctType">
    <xs:sequence>
      <xs:element name="Id" type="AcctIdType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctIdType">
    <xs:sequence>
      <!-- NUBAN replaces IBAN: Nigerian domestic accounts use a 10-digit NUBAN -->
      <xs:element name="NUBAN" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AgtType">
    <xs:sequence>
      <xs:element name="FinInstnId" type="FinInstnIdType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FinInstnIdType">
    <xs:sequence>
      <!-- InstnCode replaces BIC/ClrSysMmbId: NIBSS institution/sort code -->
      <xs:element name="InstnCode" type="xs:string"/>
      <xs:element name="Nm" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RmtInfType">
    <xs:sequence>
      <xs:element name="Ustrd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss:nps:pain.001.001.13">
  <CstmrCdtTrfInitn>
    <GrpHdr>
      <MsgId>PAIN-MSG-20260824-001</MsgId>
      <CreDtTm>2026-08-24T09:00:00</CreDtTm>
      <NbOfTxs>1</NbOfTxs>
      <CtrlSum>50000.00</CtrlSum>
      <InitgPty>
        <Nm>Credora</Nm>
      </InitgPty>
    </GrpHdr>
    <PmtInf>
      <PmtInfId>PMTINF-001</PmtInfId>
      <PmtMtd>TRF</PmtMtd>
      <BtchBookg>false</BtchBookg>
      <NbOfTxs>1</NbOfTxs>
      <CtrlSum>50000.00</CtrlSum>
      <PmtTpInf>
        <SvcLvl>
          <Cd>INST</Cd>
        </SvcLvl>
      </PmtTpInf>
      <ReqdExctnDt>2026-08-24</ReqdExctnDt>
      <Dbtr>
        <Nm>Efosa Enogie</Nm>
      </Dbtr>
      <DbtrAcct>
        <Id>
          <NUBAN>0123456789</NUBAN>
        </Id>
      </DbtrAcct>
      <DbtrAgt>
        <FinInstnId>
          <InstnCode>044</InstnCode>
          <Nm>Access Bank</Nm>
        </FinInstnId>
      </DbtrAgt>
      <CdtTrfTxInf>
        <PmtId>
          <InstrId>INSTR-001</InstrId>
          <EndToEndId>E2E-001</EndToEndId>
        </PmtId>
        <Amt>
          <InstdAmt Ccy="NGN">50000.00</InstdAmt>
        </Amt>
        <ChrgBr>SHAR</ChrgBr>
        <CdtrAgt>
          <FinInstnId>
            <InstnCode>058</InstnCode>
            <Nm>GTBank</Nm>
          </FinInstnId>
        </CdtrAgt>
        <Cdtr>
          <Nm>John Doe</Nm>
        </Cdtr>
        <CdtrAcct>
          <Id>
            <NUBAN>9876543210</NUBAN>
          </Id>
        </CdtrAcct>
        <RmtInf>
          <Ustrd>Payment for services</Ustrd>
        </RmtInf>
      </CdtTrfTxInf>
    </PmtInf>
  </CstmrCdtTrfInitn>
</Document>
```

---

## 2. pacs.008 — FI to FI Customer Credit Transfer

### XSD (`pacs.008.simplified.xsd`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns="urn:nibss:nps:pacs.008.001.08"
           targetNamespace="urn:nibss:nps:pacs.008.001.08"
           elementFormDefault="qualified">

  <xs:element name="Document" type="DocumentType"/>

  <xs:complexType name="DocumentType">
    <xs:sequence>
      <xs:element name="FIToFICstmrCdtTrf" type="FIToFICstmrCdtTrfType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FIToFICstmrCdtTrfType">
    <xs:sequence>
      <xs:element name="GrpHdr" type="GrpHdrType"/>
      <xs:element name="CdtTrfTxInf" type="CdtTrfTxInfType" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="GrpHdrType">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
      <xs:element name="NbOfTxs" type="xs:positiveInteger"/>
      <xs:element name="SttlmInf" type="SttlmInfType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="SttlmInfType">
    <xs:sequence>
      <!-- CLRG = clearing, real-time gross settlement style for NIP/NPS -->
      <xs:element name="SttlmMtd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CdtTrfTxInfType">
    <xs:sequence>
      <xs:element name="PmtId" type="PmtIdType"/>
      <!-- SessionId: NIP/NPS switch-assigned session reference, no direct
           ISO 20022 equivalent — imported from real NIP behavior -->
      <xs:element name="SessionId" type="xs:string"/>
      <!-- NameEnquiryRef: reference from the mandatory pre-transfer Name
           Enquiry call — a genuine NIP/NPS requirement, not generic ISO 20022 -->
      <xs:element name="NameEnquiryRef" type="xs:string"/>
      <xs:element name="IntrBkSttlmAmt" type="CurrencyAmountType"/>
      <xs:element name="IntrBkSttlmDt" type="xs:date"/>
      <xs:element name="Dbtr" type="PartyType"/>
      <xs:element name="DbtrAcct" type="AcctType"/>
      <xs:element name="DbtrAgt" type="AgtType"/>
      <xs:element name="Cdtr" type="PartyType"/>
      <xs:element name="CdtrAcct" type="AcctType"/>
      <xs:element name="CdtrAgt" type="AgtType"/>
      <xs:element name="RmtInf" type="RmtInfType" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PmtIdType">
    <xs:sequence>
      <xs:element name="InstrId" type="xs:string"/>
      <xs:element name="EndToEndId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmountType">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

  <xs:complexType name="PartyType">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctType">
    <xs:sequence>
      <xs:element name="Id" type="AcctIdType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctIdType">
    <xs:sequence>
      <xs:element name="NUBAN" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AgtType">
    <xs:sequence>
      <xs:element name="FinInstnId" type="FinInstnIdType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FinInstnIdType">
    <xs:sequence>
      <xs:element name="InstnCode" type="xs:string"/>
      <xs:element name="Nm" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RmtInfType">
    <xs:sequence>
      <xs:element name="Ustrd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss:nps:pacs.008.001.08">
  <FIToFICstmrCdtTrf>
    <GrpHdr>
      <MsgId>PACS008-MSG-20260824-001</MsgId>
      <CreDtTm>2026-08-24T09:00:05</CreDtTm>
      <NbOfTxs>1</NbOfTxs>
      <SttlmInf>
        <SttlmMtd>CLRG</SttlmMtd>
      </SttlmInf>
    </GrpHdr>
    <CdtTrfTxInf>
      <PmtId>
        <InstrId>INSTR-001</InstrId>
        <EndToEndId>E2E-001</EndToEndId>
      </PmtId>
      <SessionId>NPS-SESS-9f3a7c21</SessionId>
      <NameEnquiryRef>NE-8821b04e</NameEnquiryRef>
      <IntrBkSttlmAmt Ccy="NGN">50000.00</IntrBkSttlmAmt>
      <IntrBkSttlmDt>2026-08-24</IntrBkSttlmDt>
      <Dbtr>
        <Nm>Efosa Enogie</Nm>
      </Dbtr>
      <DbtrAcct>
        <Id>
          <NUBAN>0123456789</NUBAN>
        </Id>
      </DbtrAcct>
      <DbtrAgt>
        <FinInstnId>
          <InstnCode>044</InstnCode>
          <Nm>Access Bank</Nm>
        </FinInstnId>
      </DbtrAgt>
      <Cdtr>
        <Nm>John Doe</Nm>
      </Cdtr>
      <CdtrAcct>
        <Id>
          <NUBAN>9876543210</NUBAN>
        </Id>
      </CdtrAcct>
      <CdtrAgt>
        <FinInstnId>
          <InstnCode>058</InstnCode>
          <Nm>GTBank</Nm>
        </FinInstnId>
      </CdtrAgt>
      <RmtInf>
        <Ustrd>Payment for services</Ustrd>
      </RmtInf>
    </CdtTrfTxInf>
  </FIToFICstmrCdtTrf>
</Document>
```

---

## 3. pacs.002 — Payment Status Report

### XSD (`pacs.002.simplified.xsd`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns="urn:nibss:nps:pacs.002.001.10"
           targetNamespace="urn:nibss:nps:pacs.002.001.10"
           elementFormDefault="qualified">

  <xs:element name="Document" type="DocumentType"/>

  <xs:complexType name="DocumentType">
    <xs:sequence>
      <xs:element name="FIToFIPmtStsRpt" type="FIToFIPmtStsRptType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FIToFIPmtStsRptType">
    <xs:sequence>
      <xs:element name="GrpHdr" type="GrpHdrType"/>
      <xs:element name="OrgnlGrpInfAndSts" type="OrgnlGrpInfAndStsType"/>
      <xs:element name="TxInfAndSts" type="TxInfAndStsType" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="GrpHdrType">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="OrgnlGrpInfAndStsType">
    <xs:sequence>
      <xs:element name="OrgnlMsgId" type="xs:string"/>
      <xs:element name="OrgnlMsgNmId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="TxInfAndStsType">
    <xs:sequence>
      <xs:element name="OrgnlInstrId" type="xs:string"/>
      <xs:element name="OrgnlEndToEndId" type="xs:string"/>
      <!-- SessionId echoes the pacs.008 switch session reference -->
      <xs:element name="SessionId" type="xs:string"/>
      <!-- TxSts: standard ISO 20022 external status codes, e.g.
           ACSC = AcceptedSettlementCompleted, RJCT = Rejected, PDNG = Pending -->
      <xs:element name="TxSts" type="xs:string"/>
      <xs:element name="StsRsnInf" type="StsRsnInfType" minOccurs="0"/>
      <xs:element name="AccptncDtTm" type="xs:dateTime" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="StsRsnInfType">
    <xs:sequence>
      <!-- Rsn/Cd: standard ISO 20022 external reason code (AM04, AC04, etc.) -->
      <xs:element name="Rsn" type="RsnType"/>
      <!-- NipRspCd: raw NIP/NPS switch response code carried alongside the
           standard ISO code — real NIP responses carry both -->
      <xs:element name="NipRspCd" type="xs:string"/>
      <xs:element name="AddtlInf" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RsnType">
    <xs:sequence>
      <xs:element name="Cd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample XML — accepted

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss:nps:pacs.002.001.10">
  <FIToFIPmtStsRpt>
    <GrpHdr>
      <MsgId>PACS002-MSG-20260824-001</MsgId>
      <CreDtTm>2026-08-24T09:00:07</CreDtTm>
    </GrpHdr>
    <OrgnlGrpInfAndSts>
      <OrgnlMsgId>PACS008-MSG-20260824-001</OrgnlMsgId>
      <OrgnlMsgNmId>pacs.008.001.08</OrgnlMsgNmId>
    </OrgnlGrpInfAndSts>
    <TxInfAndSts>
      <OrgnlInstrId>INSTR-001</OrgnlInstrId>
      <OrgnlEndToEndId>E2E-001</OrgnlEndToEndId>
      <SessionId>NPS-SESS-9f3a7c21</SessionId>
      <TxSts>ACSC</TxSts>
      <AccptncDtTm>2026-08-24T09:00:07</AccptncDtTm>
    </TxInfAndSts>
  </FIToFIPmtStsRpt>
</Document>
```

### Sample XML — rejected

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss:nps:pacs.002.001.10">
  <FIToFIPmtStsRpt>
    <GrpHdr>
      <MsgId>PACS002-MSG-20260824-002</MsgId>
      <CreDtTm>2026-08-24T09:05:00</CreDtTm>
    </GrpHdr>
    <OrgnlGrpInfAndSts>
      <OrgnlMsgId>PACS008-MSG-20260824-002</OrgnlMsgId>
      <OrgnlMsgNmId>pacs.008.001.08</OrgnlMsgNmId>
    </OrgnlGrpInfAndSts>
    <TxInfAndSts>
      <OrgnlInstrId>INSTR-002</OrgnlInstrId>
      <OrgnlEndToEndId>E2E-002</OrgnlEndToEndId>
      <SessionId>NPS-SESS-2b71ae90</SessionId>
      <TxSts>RJCT</TxSts>
      <StsRsnInf>
        <Rsn>
          <Cd>AC04</Cd>
        </Rsn>
        <NipRspCd>07</NipRspCd>
        <AddtlInf>Beneficiary account closed</AddtlInf>
      </StsRsnInf>
    </TxInfAndSts>
  </FIToFIPmtStsRpt>
</Document>
```

---

## 4. camt.053 — Bank to Customer Statement

### XSD (`camt.053.simplified.xsd`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns="urn:nibss:nps:camt.053.001.08"
           targetNamespace="urn:nibss:nps:camt.053.001.08"
           elementFormDefault="qualified">

  <xs:element name="Document" type="DocumentType"/>

  <xs:complexType name="DocumentType">
    <xs:sequence>
      <xs:element name="BkToCstmrStmt" type="BkToCstmrStmtType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="BkToCstmrStmtType">
    <xs:sequence>
      <xs:element name="GrpHdr" type="GrpHdrType"/>
      <xs:element name="Stmt" type="StmtType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="GrpHdrType">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="StmtType">
    <xs:sequence>
      <xs:element name="Id" type="xs:string"/>
      <xs:element name="ElctrncSeqNb" type="xs:positiveInteger"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
      <xs:element name="FrToDt" type="FrToDtType"/>
      <xs:element name="Acct" type="AcctType"/>
      <xs:element name="Bal" type="BalType" maxOccurs="unbounded"/>
      <xs:element name="Ntry" type="NtryType" minOccurs="0" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FrToDtType">
    <xs:sequence>
      <xs:element name="FrDtTm" type="xs:dateTime"/>
      <xs:element name="ToDtTm" type="xs:dateTime"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctType">
    <xs:sequence>
      <xs:element name="Id" type="AcctIdType"/>
      <xs:element name="Ownr" type="PartyType"/>
      <xs:element name="Svcr" type="AgtType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AcctIdType">
    <xs:sequence>
      <xs:element name="NUBAN" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PartyType">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="AgtType">
    <xs:sequence>
      <xs:element name="FinInstnId" type="FinInstnIdType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="FinInstnIdType">
    <xs:sequence>
      <xs:element name="InstnCode" type="xs:string"/>
      <xs:element name="Nm" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="BalType">
    <xs:sequence>
      <!-- Tp/Cd: OPBD = opening booked, CLBD = closing booked -->
      <xs:element name="Tp" type="BalTpType"/>
      <xs:element name="Amt" type="CurrencyAmountType"/>
      <!-- CdtDbtInd: CRDT / DBIT -->
      <xs:element name="CdtDbtInd" type="xs:string"/>
      <xs:element name="Dt" type="xs:date"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="BalTpType">
    <xs:sequence>
      <xs:element name="Cd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="NtryType">
    <xs:sequence>
      <xs:element name="NtryRef" type="xs:string"/>
      <xs:element name="Amt" type="CurrencyAmountType"/>
      <xs:element name="CdtDbtInd" type="xs:string"/>
      <!-- Sts: BOOK = booked/settled, PDNG = pending -->
      <xs:element name="Sts" type="xs:string"/>
      <xs:element name="BookgDt" type="xs:date"/>
      <xs:element name="ValDt" type="xs:date"/>
      <!-- BkTxCd: domestic bank transaction code, kept generic/string here -->
      <xs:element name="BkTxCd" type="xs:string"/>
      <xs:element name="NtryDtls" type="NtryDtlsType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="NtryDtlsType">
    <xs:sequence>
      <xs:element name="TxDtls" type="TxDtlsType"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="TxDtlsType">
    <xs:sequence>
      <xs:element name="Refs" type="RefsType"/>
      <xs:element name="RmtInf" type="RmtInfType" minOccurs="0"/>
      <xs:element name="RltdPties" type="RltdPtiesType" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RefsType">
    <xs:sequence>
      <xs:element name="EndToEndId" type="xs:string"/>
      <!-- SessionId ties the statement entry back to its originating
           pacs.008/pacs.002 switch session -->
      <xs:element name="SessionId" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RmtInfType">
    <xs:sequence>
      <xs:element name="Ustrd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="RltdPtiesType">
    <xs:sequence>
      <xs:element name="Dbtr" type="PartyType" minOccurs="0"/>
      <xs:element name="Cdtr" type="PartyType" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmountType">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

</xs:schema>
```

### Sample XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss:nps:camt.053.001.08">
  <BkToCstmrStmt>
    <GrpHdr>
      <MsgId>CAMT053-MSG-20260824-001</MsgId>
      <CreDtTm>2026-08-24T23:59:59</CreDtTm>
    </GrpHdr>
    <Stmt>
      <Id>STMT-20260824-0123456789</Id>
      <ElctrncSeqNb>1</ElctrncSeqNb>
      <CreDtTm>2026-08-24T23:59:59</CreDtTm>
      <FrToDt>
        <FrDtTm>2026-08-24T00:00:00</FrDtTm>
        <ToDtTm>2026-08-24T23:59:59</ToDtTm>
      </FrToDt>
      <Acct>
        <Id>
          <NUBAN>0123456789</NUBAN>
        </Id>
        <Ownr>
          <Nm>Efosa Enogie</Nm>
        </Ownr>
        <Svcr>
          <FinInstnId>
            <InstnCode>044</InstnCode>
            <Nm>Access Bank</Nm>
          </FinInstnId>
        </Svcr>
      </Acct>
      <Bal>
        <Tp>
          <Cd>OPBD</Cd>
        </Tp>
        <Amt Ccy="NGN">150000.00</Amt>
        <CdtDbtInd>CRDT</CdtDbtInd>
        <Dt>2026-08-24</Dt>
      </Bal>
      <Bal>
        <Tp>
          <Cd>CLBD</Cd>
        </Tp>
        <Amt Ccy="NGN">100000.00</Amt>
        <CdtDbtInd>CRDT</CdtDbtInd>
        <Dt>2026-08-24</Dt>
      </Bal>
      <Ntry>
        <NtryRef>NTRY-0001</NtryRef>
        <Amt Ccy="NGN">50000.00</Amt>
        <CdtDbtInd>DBIT</CdtDbtInd>
        <Sts>BOOK</Sts>
        <BookgDt>2026-08-24</BookgDt>
        <ValDt>2026-08-24</ValDt>
        <BkTxCd>NIP-OUT-TRF</BkTxCd>
        <NtryDtls>
          <TxDtls>
            <Refs>
              <EndToEndId>E2E-001</EndToEndId>
              <SessionId>NPS-SESS-9f3a7c21</SessionId>
            </Refs>
            <RmtInf>
              <Ustrd>Payment for services</Ustrd>
            </RmtInf>
            <RltdPties>
              <Cdtr>
                <Nm>John Doe</Nm>
              </Cdtr>
            </RltdPties>
          </TxDtls>
        </NtryDtls>
      </Ntry>
    </Stmt>
  </BkToCstmrStmt>
</Document>
```

---

## Notes on using these for the simulator

- All four schemas use custom namespaces (`urn:nibss:nps:...`) rather than the
  official ISO 20022 namespaces, since the element sets here are simplified
  and include Nigerian-specific renames — don't present these as literal
  ISO 20022-conformant payloads to any real validator or counterparty.
- `SessionId` is the thread that ties a `pacs.008` request, its `pacs.002`
  response, and the eventual `camt.053` statement entry together — useful as
  your simulator's internal correlation key even before you know NIBSS's real
  equivalent field name.
- `NipRspCd` is a placeholder pattern (2-digit string here) for whatever
  NIBSS's actual raw switch response code list turns out to be — treat the
  value `"07"` in the rejected pacs.002 sample as illustrative only.
- Once sandbox access is available, the highest-value check is: do request/
  response field names match what NIBSS actually sends, and does the
  `InstnCode` field line up with real NIBSS bank codes (CBN-assigned 3-digit
  codes, e.g. 044 = Access Bank, 058 = GTBank) or does NPS use a different
  code list for the new stack.
