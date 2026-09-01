# NIBSS-SIM ISO 20022 Learning Reference

This file defines a **small, learning-oriented subset** of four ISO 20022 message families that are useful for the NIBSS-SIM project:

- `pain.001` — Customer Credit Transfer Initiation
- `pacs.008` — FI-to-FI Customer Credit Transfer
- `pacs.002` — FI-to-FI Payment Status Report
- `camt.053` — Bank-to-Customer Statement

These are **not NIBSS production schemas** and are not intended for certification or interoperability with a real NIBSS endpoint. The schemas deliberately keep only the fields needed to understand the payment flow.

## 1. Correct mental model

Do not think of these four messages as four mandatory NIBSS API calls.

```text
Customer / Credora
       │
       │ pain.001-shaped instruction
       ▼
Sending Institution
       │
       │ pacs.008
       ▼
NIBSS / NIP switch
       │
       │ routes payment
       ▼
Receiving Institution
       │
       │ processing result
       ▼
NIBSS / NIP switch
       │
       └── pacs.002-shaped status

Later:
Bank ── camt.053 ──> Customer / account holder
```

For **NIBSS-SIM V1**, the most useful message to learn is `pacs.008`, because it represents the interbank customer-credit-transfer leg.

`pain.001` becomes useful when you model the boundary between a customer/payment platform and its sending institution.

`pacs.002` models payment status.

`camt.053` models account-statement/reconciliation data and belongs in a later phase.

## 2. What the four messages mean

| Message | ISO meaning | NIBSS-SIM learning meaning |
|---|---|---|
| `pain.001` | Customer Credit Transfer Initiation | An originator/customer-side instruction asking its bank or payment provider to make a credit transfer |
| `pacs.008` | FI-to-FI Customer Credit Transfer | The interbank payment message sent by the Sending Institution into the switch |
| `pacs.002` | FI-to-FI Payment Status Report | A status report referring to the original payment message |
| `camt.053` | Bank-to-Customer Statement | A statement containing booked account entries for reconciliation |

### Important terminology correction

`pacs.008` does **not** mean "NIBSS transfer request".

It is an ISO 20022 **FI-to-FI Customer Credit Transfer** message. NIBSS-SIM uses it as the conceptual model for the interbank leg.

Likewise, `pacs.002` is a **payment status report**, not simply an "ISO response". It refers back to the original payment.

`camt.053` is a **bank-to-customer statement**, not a NIBSS transfer response.

## 3. NIBSS-SIM namespace convention

Use simulator-owned namespaces so the project never looks like it is claiming to implement official NIBSS schemas:

```text
urn:nibss-sim:iso20022:pain.001.001.13
urn:nibss-sim:iso20022:pacs.008.001.08
urn:nibss-sim:iso20022:pacs.002.001.10
urn:nibss-sim:iso20022:camt.053.001.08
```

The message identifiers above describe the ISO message versions being studied. The namespace itself is owned by **NIBSS-SIM**.

For the ordinary REST/XML simulator API, use a separate namespace:

```text
urn:nibss-sim:nip:api:v1
```

That distinction is important:

```text
NIBSS-SIM REST/XML API
urn:nibss-sim:nip:api:v1

NIBSS-SIM ISO learning messages
urn:nibss-sim:iso20022:...
```

## 4. Nigerian account and institution identifiers

A previous version of this document incorrectly treated `NUBAN` as if it were an ISO 20022 XML element.

It is better to understand it this way:

- **NUBAN** is the Nigerian domestic bank-account convention.
- ISO 20022 does not require a literal `<NUBAN>` element.
- In a simplified ISO-style model, a domestic account can be represented using `Acct/Id/Othr/Id`, with a proprietary scheme such as `NUBAN`.
- The simulator can expose a simpler `<NUBAN>` element in its own API XML if that makes the exercise easier.

Similarly, `InstnCode` is a **NIBSS-SIM convenience field**. It should not be presented as a standard ISO 20022 element.

Example simulator-specific identifier:

```xml
<Acct>
  <Id>
    <Othr>
      <Id>1234567890</Id>
      <SchmeNm>
        <Prtry>NUBAN</Prtry>
      </SchmeNm>
    </Othr>
  </Id>
</Acct>
```

And a simplified institution identifier:

```xml
<FinInstnId>
  <Othr>
    <Id>990001</Id>
    <SchmeNm>
      <Prtry>NIBSS-SIM-INSTN-CODE</Prtry>
    </SchmeNm>
  </Othr>
</FinInstnId>
```

## 5. pain.001 — Customer Credit Transfer Initiation

### Simple definition

`pain.001` represents an instruction from a **customer/originator side** to its bank or payment service provider to initiate a credit transfer.

For NIBSS-SIM, this is useful for learning the upstream side of the payment chain.

### Minimal learning XSD

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:nibss-sim:iso20022:pain.001.001.13"
           xmlns="urn:nibss-sim:iso20022:pain.001.001.13"
           elementFormDefault="qualified">

  <xs:element name="Document">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="CstmrCdtTrfInitn">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="GrpHdr" type="GroupHeader"/>
              <xs:element name="PmtInf" type="PaymentInformation"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>

  <xs:complexType name="GroupHeader">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
      <xs:element name="NbOfTxs" type="xs:positiveInteger"/>
      <xs:element name="CtrlSum" type="xs:decimal"/>
      <xs:element name="InitgPty" type="Party"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PaymentInformation">
    <xs:sequence>
      <xs:element name="PmtInfId" type="xs:string"/>
      <xs:element name="PmtMtd" type="xs:string"/>
      <xs:element name="Dbtr" type="Party"/>
      <xs:element name="DbtrAcct" type="Account"/>
      <xs:element name="CdtTrfTxInf" type="CreditTransfer"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CreditTransfer">
    <xs:sequence>
      <xs:element name="PmtId" type="PaymentId"/>
      <xs:element name="Amt" type="Amount"/>
      <xs:element name="Cdtr" type="Party"/>
      <xs:element name="CdtrAcct" type="Account"/>
      <xs:element name="RmtInf" type="Remittance" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PaymentId">
    <xs:sequence>
      <xs:element name="InstrId" type="xs:string"/>
      <xs:element name="EndToEndId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Amount">
    <xs:sequence>
      <xs:element name="InstdAmt" type="CurrencyAmount"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmount">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

  <xs:complexType name="Party">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Account">
    <xs:sequence>
      <xs:element name="Id">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Othr">
              <xs:complexType>
                <xs:sequence>
                  <xs:element name="Id" type="xs:string"/>
                  <xs:element name="SchmeNm">
                    <xs:complexType>
                      <xs:sequence>
                        <xs:element name="Prtry" type="xs:string"/>
                      </xs:sequence>
                    </xs:complexType>
                  </xs:element>
                </xs:sequence>
              </xs:complexType>
            </xs:element>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Remittance">
    <xs:sequence>
      <xs:element name="Ustrd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss-sim:iso20022:pain.001.001.13">
  <CstmrCdtTrfInitn>
    <GrpHdr>
      <MsgId>PAIN001-20260901-001</MsgId>
      <CreDtTm>2026-09-01T09:00:00</CreDtTm>
      <NbOfTxs>1</NbOfTxs>
      <CtrlSum>50000.00</CtrlSum>
      <InitgPty>
        <Nm>Credora</Nm>
      </InitgPty>
    </GrpHdr>

    <PmtInf>
      <PmtInfId>PMTINF-001</PmtInfId>
      <PmtMtd>TRF</PmtMtd>

      <Dbtr>
        <Nm>Credora Customer</Nm>
      </Dbtr>
      <DbtrAcct>
        <Id>
          <Othr>
            <Id>1000000001</Id>
            <SchmeNm>
              <Prtry>NUBAN</Prtry>
            </SchmeNm>
          </Othr>
        </Id>
      </DbtrAcct>

      <CdtTrfTxInf>
        <PmtId>
          <InstrId>INSTR-001</InstrId>
          <EndToEndId>E2E-001</EndToEndId>
        </PmtId>
        <Amt>
          <InstdAmt Ccy="NGN">50000.00</InstdAmt>
        </Amt>
        <Cdtr>
          <Nm>John Doe</Nm>
        </Cdtr>
        <CdtrAcct>
          <Id>
            <Othr>
              <Id>1234567890</Id>
              <SchmeNm>
                <Prtry>NUBAN</Prtry>
              </SchmeNm>
            </Othr>
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

## 6. pacs.008 — FI-to-FI Customer Credit Transfer

### Simple definition

This is the **interbank payment message**.

In the simulator:

```text
Credora / Sending Institution
        │
        │ pacs.008-shaped message
        ▼
     NIBSS-SIM
        │
        │ route using destination institution code
        ▼
Ridgeway / Receiving Institution
```

The important financial concepts are:

- `Dbtr` — Debtor / originator
- `DbtrAcct` — Debtor account
- `DbtrAgt` — Debtor's financial institution
- `Cdtr` — Creditor / beneficiary
- `CdtrAcct` — Creditor account
- `CdtrAgt` — Creditor's financial institution
- `PmtId` — payment identifiers
- `IntrBkSttlmAmt` — interbank settlement amount
- `RmtInf` — remittance information

For NIBSS-SIM, keep `SessionId` as a **simulator correlation field** rather than pretending it is a standard ISO 20022 element.

### Minimal learning XSD

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:nibss-sim:iso20022:pacs.008.001.08"
           xmlns="urn:nibss-sim:iso20022:pacs.008.001.08"
           elementFormDefault="qualified">

  <xs:element name="Document">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="FIToFICstmrCdtTrf">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="GrpHdr" type="GroupHeader"/>
              <xs:element name="CdtTrfTxInf" type="CreditTransfer"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>

  <xs:complexType name="GroupHeader">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
      <xs:element name="NbOfTxs" type="xs:positiveInteger"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CreditTransfer">
    <xs:sequence>
      <xs:element name="PmtId" type="PaymentId"/>
      <xs:element name="SessionId" type="xs:string"/>
      <xs:element name="IntrBkSttlmAmt" type="CurrencyAmount"/>
      <xs:element name="Dbtr" type="Party"/>
      <xs:element name="DbtrAcct" type="Account"/>
      <xs:element name="DbtrAgt" type="Agent"/>
      <xs:element name="Cdtr" type="Party"/>
      <xs:element name="CdtrAcct" type="Account"/>
      <xs:element name="CdtrAgt" type="Agent"/>
      <xs:element name="RmtInf" type="Remittance" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="PaymentId">
    <xs:sequence>
      <xs:element name="InstrId" type="xs:string"/>
      <xs:element name="EndToEndId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmount">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

  <xs:complexType name="Party">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Agent">
    <xs:sequence>
      <xs:element name="FinInstnId">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Othr">
              <xs:complexType>
                <xs:sequence>
                  <xs:element name="Id" type="xs:string"/>
                  <xs:element name="SchmeNm">
                    <xs:complexType>
                      <xs:sequence>
                        <xs:element name="Prtry" type="xs:string"/>
                      </xs:sequence>
                    </xs:complexType>
                  </xs:element>
                </xs:sequence>
              </xs:complexType>
            </xs:element>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Account">
    <xs:sequence>
      <xs:element name="Id">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Othr">
              <xs:complexType>
                <xs:sequence>
                  <xs:element name="Id" type="xs:string"/>
                  <xs:element name="SchmeNm">
                    <xs:complexType>
                      <xs:sequence>
                        <xs:element name="Prtry" type="xs:string"/>
                      </xs:sequence>
                    </xs:complexType>
                  </xs:element>
                </xs:sequence>
              </xs:complexType>
            </xs:element>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Remittance">
    <xs:sequence>
      <xs:element name="Ustrd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss-sim:iso20022:pacs.008.001.08">
  <FIToFICstmrCdtTrf>
    <GrpHdr>
      <MsgId>PACS008-20260901-001</MsgId>
      <CreDtTm>2026-09-01T09:00:05</CreDtTm>
      <NbOfTxs>1</NbOfTxs>
    </GrpHdr>

    <CdtTrfTxInf>
      <PmtId>
        <InstrId>INSTR-001</InstrId>
        <EndToEndId>E2E-001</EndToEndId>
      </PmtId>

      <SessionId>NPS-SESS-20260901-000001</SessionId>

      <IntrBkSttlmAmt Ccy="NGN">50000.00</IntrBkSttlmAmt>

      <Dbtr>
        <Nm>Credora Customer</Nm>
      </Dbtr>
      <DbtrAcct>
        <Id>
          <Othr>
            <Id>1000000001</Id>
            <SchmeNm><Prtry>NUBAN</Prtry></SchmeNm>
          </Othr>
        </Id>
      </DbtrAcct>
      <DbtrAgt>
        <FinInstnId>
          <Othr>
            <Id>999999</Id>
            <SchmeNm><Prtry>NIBSS-SIM-INSTN-CODE</Prtry></SchmeNm>
          </Othr>
        </FinInstnId>
      </DbtrAgt>

      <Cdtr>
        <Nm>John Doe</Nm>
      </Cdtr>
      <CdtrAcct>
        <Id>
          <Othr>
            <Id>1234567890</Id>
            <SchmeNm><Prtry>NUBAN</Prtry></SchmeNm>
          </Othr>
        </Id>
      </CdtrAcct>
      <CdtrAgt>
        <FinInstnId>
          <Othr>
            <Id>990001</Id>
            <SchmeNm><Prtry>NIBSS-SIM-INSTN-CODE</Prtry></SchmeNm>
          </Othr>
        </FinInstnId>
      </CdtrAgt>

      <RmtInf>
        <Ustrd>Payment for invoice 4521</Ustrd>
      </RmtInf>
    </CdtTrfTxInf>
  </FIToFICstmrCdtTrf>
</Document>
```

## 7. pacs.002 — FI-to-FI Payment Status Report

### Simple definition

`pacs.002` reports the processing status of an earlier interbank payment.

The key relationship is:

```text
pacs.008
   │
   │ original payment
   ▼
NIBSS-SIM
   │
   │ pacs.002
   ▼
Sending Institution
```

The status belongs to the original transaction.

For the learning simulator, use:

- `ACSC` — accepted and settlement completed
- `RJCT` — rejected
- `PDNG` — pending

These are ISO-style status concepts. Keep the simulator's own `responseCode` separate when you want NIBSS-SIM-specific diagnostic behavior.

### Minimal learning XSD

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:nibss-sim:iso20022:pacs.002.001.10"
           xmlns="urn:nibss-sim:iso20022:pacs.002.001.10"
           elementFormDefault="qualified">

  <xs:element name="Document">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="FIToFIPmtStsRpt">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="GrpHdr" type="GroupHeader"/>
              <xs:element name="OrgnlGrpInfAndSts" type="OriginalGroup"/>
              <xs:element name="TxInfAndSts" type="TransactionStatus"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>

  <xs:complexType name="GroupHeader">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="OriginalGroup">
    <xs:sequence>
      <xs:element name="OrgnlMsgId" type="xs:string"/>
      <xs:element name="OrgnlMsgNmId" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="TransactionStatus">
    <xs:sequence>
      <xs:element name="OrgnlInstrId" type="xs:string"/>
      <xs:element name="OrgnlEndToEndId" type="xs:string"/>
      <xs:element name="SessionId" type="xs:string"/>
      <xs:element name="TxSts" type="xs:string"/>
      <xs:element name="NipRspCd" type="xs:string" minOccurs="0"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Accepted sample

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss-sim:iso20022:pacs.002.001.10">
  <FIToFIPmtStsRpt>
    <GrpHdr>
      <MsgId>PACS002-20260901-001</MsgId>
      <CreDtTm>2026-09-01T09:00:07</CreDtTm>
    </GrpHdr>

    <OrgnlGrpInfAndSts>
      <OrgnlMsgId>PACS008-20260901-001</OrgnlMsgId>
      <OrgnlMsgNmId>pacs.008.001.08</OrgnlMsgNmId>
    </OrgnlGrpInfAndSts>

    <TxInfAndSts>
      <OrgnlInstrId>INSTR-001</OrgnlInstrId>
      <OrgnlEndToEndId>E2E-001</OrgnlEndToEndId>
      <SessionId>NPS-SESS-20260901-000001</SessionId>
      <TxSts>ACSC</TxSts>
      <NipRspCd>00</NipRspCd>
    </TxInfAndSts>
  </FIToFIPmtStsRpt>
</Document>
```

### Rejected sample

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss-sim:iso20022:pacs.002.001.10">
  <FIToFIPmtStsRpt>
    <GrpHdr>
      <MsgId>PACS002-20260901-002</MsgId>
      <CreDtTm>2026-09-01T09:05:00</CreDtTm>
    </GrpHdr>

    <OrgnlGrpInfAndSts>
      <OrgnlMsgId>PACS008-20260901-002</OrgnlMsgId>
      <OrgnlMsgNmId>pacs.008.001.08</OrgnlMsgNmId>
    </OrgnlGrpInfAndSts>

    <TxInfAndSts>
      <OrgnlInstrId>INSTR-002</OrgnlInstrId>
      <OrgnlEndToEndId>E2E-002</OrgnlEndToEndId>
      <SessionId>NPS-SESS-20260901-000002</SessionId>
      <TxSts>RJCT</TxSts>
      <NipRspCd>51</NipRspCd>
    </TxInfAndSts>
  </FIToFIPmtStsRpt>
</Document>
```

## 8. camt.053 — Bank-to-Customer Statement

### Simple definition

`camt.053` is a **bank-to-customer account statement**.

It is useful for the later reconciliation side of NIBSS-SIM:

```text
Ridgeway / participant bank
          │
          │ account statement
          ▼
Customer / Credora
```

It is not a transfer initiation message.

A statement contains things such as:

- account identification
- opening/closing balances
- booked entries
- transaction references
- remittance information

### Minimal learning XSD

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="urn:nibss-sim:iso20022:camt.053.001.08"
           xmlns="urn:nibss-sim:iso20022:camt.053.001.08"
           elementFormDefault="qualified">

  <xs:element name="Document">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="BkToCstmrStmt">
          <xs:complexType>
            <xs:sequence>
              <xs:element name="GrpHdr" type="GroupHeader"/>
              <xs:element name="Stmt" type="Statement"/>
            </xs:sequence>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>

  <xs:complexType name="GroupHeader">
    <xs:sequence>
      <xs:element name="MsgId" type="xs:string"/>
      <xs:element name="CreDtTm" type="xs:dateTime"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Statement">
    <xs:sequence>
      <xs:element name="Id" type="xs:string"/>
      <xs:element name="Acct" type="Account"/>
      <xs:element name="Bal" type="Balance" maxOccurs="2"/>
      <xs:element name="Ntry" type="Entry" minOccurs="0" maxOccurs="unbounded"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Account">
    <xs:sequence>
      <xs:element name="Id">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Othr">
              <xs:complexType>
                <xs:sequence>
                  <xs:element name="Id" type="xs:string"/>
                  <xs:element name="SchmeNm">
                    <xs:complexType>
                      <xs:sequence>
                        <xs:element name="Prtry" type="xs:string"/>
                      </xs:sequence>
                    </xs:complexType>
                  </xs:element>
                </xs:sequence>
              </xs:complexType>
            </xs:element>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
      <xs:element name="Ownr" type="Party"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Balance">
    <xs:sequence>
      <xs:element name="Tp">
        <xs:complexType>
          <xs:sequence>
            <xs:element name="Cd" type="xs:string"/>
          </xs:sequence>
        </xs:complexType>
      </xs:element>
      <xs:element name="Amt" type="CurrencyAmount"/>
      <xs:element name="CdtDbtInd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="Entry">
    <xs:sequence>
      <xs:element name="NtryRef" type="xs:string"/>
      <xs:element name="Amt" type="CurrencyAmount"/>
      <xs:element name="CdtDbtInd" type="xs:string"/>
      <xs:element name="Sts" type="xs:string"/>
      <xs:element name="BkTxCd" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

  <xs:complexType name="CurrencyAmount">
    <xs:simpleContent>
      <xs:extension base="xs:decimal">
        <xs:attribute name="Ccy" type="xs:string" use="required"/>
      </xs:extension>
    </xs:simpleContent>
  </xs:complexType>

  <xs:complexType name="Party">
    <xs:sequence>
      <xs:element name="Nm" type="xs:string"/>
    </xs:sequence>
  </xs:complexType>

</xs:schema>
```

### Sample

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:nibss-sim:iso20022:camt.053.001.08">
  <BkToCstmrStmt>
    <GrpHdr>
      <MsgId>CAMT053-20260901-001</MsgId>
      <CreDtTm>2026-09-01T23:59:59</CreDtTm>
    </GrpHdr>

    <Stmt>
      <Id>STMT-20260901-001</Id>

      <Acct>
        <Id>
          <Othr>
            <Id>1000000001</Id>
            <SchmeNm><Prtry>NUBAN</Prtry></SchmeNm>
          </Othr>
        </Id>
        <Ownr>
          <Nm>Credora Customer</Nm>
        </Ownr>
      </Acct>

      <Bal>
        <Tp><Cd>OPBD</Cd></Tp>
        <Amt Ccy="NGN">150000.00</Amt>
        <CdtDbtInd>CRDT</CdtDbtInd>
      </Bal>

      <Bal>
        <Tp><Cd>CLBD</Cd></Tp>
        <Amt Ccy="NGN">100000.00</Amt>
        <CdtDbtInd>CRDT</CdtDbtInd>
      </Bal>

      <Ntry>
        <NtryRef>NTRY-0001</NtryRef>
        <Amt Ccy="NGN">50000.00</Amt>
        <CdtDbtInd>DBIT</CdtDbtInd>
        <Sts>BOOK</Sts>
        <BkTxCd>NIP-OUT-TRF</BkTxCd>
      </Ntry>
    </Stmt>
  </BkToCstmrStmt>
</Document>
```

## 9. How these map to the NIBSS-SIM project

Keep the Java application simpler than the message standard.

```text
controller/
    receives REST request

dto/
    represents the API contract

xml/
    validates XML against the simplified XSD
    maps XML ↔ Java DTO

service/
    performs NIBSS-SIM business logic

domain/
    Bank
    Transfer
    CallbackAttempt

repository/
    persists switch state

client/
    calls participant-bank services
```

The key learning flow is:

```text
1. Name Enquiry
      ↓
2. Transfer request
      ↓
3. NIBSS-SIM creates Transfer
      ↓
4. Route to Receiving Institution
      ↓
5. Participant bank processes credit
      ↓
6. NIBSS-SIM records final state
      ↓
7. pacs.002-shaped status / REST response
      ↓
8. Later: camt.053 statement for reconciliation
```

## 10. What should actually be implemented first

Do not implement all four message families at once.

### V1

Implement the simulator REST API:

```text
POST /name-enquiry
POST /transfers
GET  /transfers/{reference}
```

Support JSON first.

### V1.5

Add XML to the same API contract:

```text
Content-Type: application/xml
Accept: application/xml
```

Use:

```text
urn:nibss-sim:nip:api:v1
```

### V2

Introduce the ISO-shaped interbank message:

```text
pacs.008
```

Use it internally or as an optional wire representation of the transfer message.

### V2.5

Add:

```text
pacs.002
```

for payment status.

### V3

Add:

```text
camt.053
```

for statement/reconciliation exercises.

## Final rule for this project

Use **real financial terminology**, but do not pretend the simulator is an official NIBSS implementation.

That means it is good to learn:

```text
Originator
Debtor
Creditor
Beneficiary
Sending Institution
Receiving Institution
Financial Institution
Customer Credit Transfer
Payment Status Report
Account Statement
Settlement
Reconciliation
SessionId
EndToEndId
InstructionId
Response Code
```

But keep clearly marked simulator concepts:

```text
NIBSS-SIM institution codes
NIBSS-SIM API namespace
NIBSS-SIM SessionId format
NIBSS-SIM response-code mapping
NIBSS-SIM XML extensions
```

This gives the project authentic **financial-system structure and vocabulary** without teaching you that the simplified XML is an official NIBSS specification.
