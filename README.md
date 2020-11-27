
# Message

## Message Header Rule

Header is 2 bytes before the message. Server will read first 2 bytes, calculate the message length, then read the message according the length.

## Message Header Calculation

**Knowledge Base**

For example, length of the message is 8498 byte.
| Source                      | Decimal  | ASCII |
| --------------------------- | -------- | ----- |
| 8498 % 256                  | 50       | 2     |
| (8498 - (8498 % 256)) / 256 | 33       | !     |

**Byte Ordering**

|            | Most Significant | Least Significant |
| ---------- | ---------------- | ----------------- |
| Decimal    | 33               | 50                |
| Binary     | 00100001         | 00110010          |
| ASCII      | !                | 2                 |

Big endian = put **most** significant element first
Little endian = put **least** significant element first

If message header calculated with _big endian_, the header of the message will be `!2`.   If message header calculated with _little endian_, the header of the message will be `2!`.

Otherwise, if we will calculate message header from byte `3z`. `3` on ASCII table is `51` and `z` on ASCII table is `122`.

If we use _big endian_, we will get 51 * 256 + 122 = 13178. If we use _little endian_, we will get 122 * 256 + 51 = 31283. 

| Source                        | Decimal  | ASCII |
| ----------------------------- | -------- | ----- |
| 13178 % 256                   | 122      | z     |
| (13178 - (13178 % 256)) / 256 | 51       | 3     |

or

| Source                        | Decimal  | ASCII |
| ----------------------------- | -------- | ----- |
| 31283 % 256                   | 51       | 3     |
| (31283 - (31283 % 256)) / 256 | 122      | z     |

Note: message header is binary data and not always printable. Be careful to `copy and paste` it.

**Message Header Calculating Using Java**

```java
class HeaderTool
{
private HeaderTool()
{
}
/**
 * Create message header
 * @param messageLength Message length (in byte)
 * @param byteOrder Byte order (true = little endian, false = big endian)
 * @return Byte represent length of message
 */
public static byte[] createISOLength(long messageLength, boolean byteOrder)
{
    byte[] header = new byte[2];
    if(byteOrder)
    {
        header[0] = (byte) ((byte) messageLength & 0xFF);
        header[1] = (byte) ((byte) (messageLength >> 8) & 0xFF);
    }
    else
    {
        header[0] = (byte) ((byte) (messageLength >> 8) & 0xFF);
        header[1] = (byte) ((byte) messageLength & 0xFF);
    }
    return header;		
}
}
/**
 * Calculate messahe header
 * @param header Message header
 * @param byteOrder Byte order (true = little endian, false = big endian)
 * @return Actual message length
 * @throws NegativeLengthException if message length is lower than 0
 */
public static long getLength(byte[] header, boolean byteOrder) 
{
    int headerLength = header.length;
    int i;
    long result = 0;
    int x = 0;
    if(byteOrder)
    {
        for(i = headerLength-1; i >= 0; i--)
        {
            result *= 256;
            x = header[i];
            if(x < 0)
            {
                x = x+256;
            }
            if(x > 256)
            {
                x = x-256;
            }
            result += x;
        }          
    }
    else
    {
        for(i = 0; i < headerLength; i++)
        {
            result *= 256;
            x = header[i];
            if(x < 0)
            {
                x = x+256;
            }
            if(x > 256)
            {
                x = x-256;
            }
            result += x;
        }
    }
    if(result < 0)
    {
        result = 0;
    }
    return result;
}
```

# ISO 8385 Data Element

| DE | Name | Type | Remark | 0200 | 0210 | 0400 | 0420 |
|---|---|---|---|---|---|---|---|
| - | Message Type | N 4 | ‘0200’ / ‘0400’ (Request) | M | M | M | M |
| - | Primary Bit Map | AN 16 | Mandatory for all Messages | M | M | M | M |
| P1 | Secondary Bit Map | AN 16 | Only present if any of DE 65 to DE 128 are present. | M | M | M | M |
| P2 | Primary Account Number | N..19 LLVAR | PAN | M | ME | ME | ME |
| P3 | Processing Code | N 6 | EBIPP: <br>‘37xxxx’ (Inquiry)<br>‘80xxxx’ (Payment) <br>‘80xxxx’ (Reversal) | M | M | ME | ME |
| P4 | Transaction Amount | N 12 | Inquiry: set all zeroes Payment: Debited Amount Reversal: Debited Amount | M | M | ME | ME |
| | | | Refer to Section 4. Data Element and Name Definition | | | | |
| P7 | Transmission Date and | N 10 | In GMT | M | M | M | M |
| | Time | | Format: MMDDhhmmss | | | | |
| P11 | System Trace Audit Number | N 6 | Transaction Trace Number. This is Unique per message transmitted. | M | ME | M | ME |
| P12 | Local Transaction Time | N 6 | Format: hhmmss | M | ME | ME | ME |
| P13 | Local Transaction Date | N 4 | Format: MMDD | M | ME | ME | ME |
| P14 | Expiration Date | N 4 | For Future Use | C | CE | CE | CE |
| P15 | Settlement Date | N 4 | Format: MMDD | M | ME | ME | ME |
| P18 | Channel Type | N 4 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P22 | Point of Service Entry Mode | N 3 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P25 | Point of Service Condition Code | N 2 | Refer to Section 4. Data Element and Name Definition | C | CE | CE | CE |
| P32 | Acquiring Institution ID | N..11 LLVAR | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P33 | Forwarding Institution ID | N..11 LLVAR | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P35 | Track-2 Data | Z..37 LLVAR | Inquiry: OFF <br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF <br>Refer to Section 4. Data Element and Name Definition | C | - | - | - |
| P37 | Retrieval Reference Number | AN 12 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P39 | Response Code | AN 2 | Response Code | | M | | M |
| P41 | Card Acceptor Terminal Identification | ANS 16 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P42 | Card Acceptor ID | ANS 15 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P43 | Card Acceptor Name/Location | ANS 40 | Refer to Section 4. Data Element and Name Definition | M | ME | | |
| P48 | Additional Data | ANS..999 LLLVAR | Refer to Section 4. Data Element and Name Definition | M | M | ME | ME |
| P49 | Transaction Currency Code | N 3 | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| P52 | Personal Identification Number (PIN) Data | AN 16 | Inquiry: OFF<br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF<br>Refer to Section 4. Data Element and Name Definition | C | - | - | - |
| P55 | ICC Data | ANS..765 | Inquiry: OFF <br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF<br>Refer to Section 4. Data Element and Name Definition | C | C | - | - |
| P57 | Data Payment – National | ANS..999 LLLVAR | Refer to Section 4. Data Element and Name Definition | ME | ME | ME | ME |
| S90 | Original Data Element | N--42 | Refer to Section 4. Data Element and Name Definition | | | M | ME |
| S100 | Receiving Institution Identification Code | N..11 LLVAR | Issuer Code <br>Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| S125 | Transaction Indicator | N..255 LLLVAR | Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |
| S127 | Destination Institution Identification Code | N..11 LLLVAR | Biller Identification Code <br>Refer to Section 4. Data Element and Name Definition | M | ME | ME | ME |

## Additional Data

### Field 48 Inquiry and Payment
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Desc                               |
|-----|-----|---------|---------|---------|---------|------------------------------------|
| PI  | 6   | M       | ME      | ME      | ME      | Product ID                         |
| CN  | 24  | M       | ME      | ME      | ME      | Contract number or customer number |
| NM  | 50  |         | M       | ME      | ME      | Customer name                      |
| AT  | 12  |         | M       | ME      | ME      | Total amount without point decimal |
| AC  | 12  |         | C       | CE      | CE      | Total admin fee                    |
| C1  | 24  | C       | CE      | CE      | CE      | Customer reference 1               |
| C2  | 24  | C       | CE      | CE      | CE      | Customer reference 2               |
| FR  | 24  |         | M       | ME      | ME      | Forwarding reference number        |
| FS  | 24  |         | M       | ME      | ME      | Forwarding STAN                    |

### Field 57 Inquiry
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Desc           |
|-----|-----|---------|---------|---------|---------|----------------|
| S0  | 50  |         | M       |         |         | Inquiry screen |
| S1  | 50  |         | M       |         |         | Inquiry screen |
| S2  | 50  |         | M       |         |         | Inquiry screen |
| S3  | 50  |         | M       |         |         | Inquiry screen |
| S4  | 50  |         | M       |         |         | Inquiry screen |
| S5  | 50  |         | M       |         |         | Inquiry screen |
| S6  | 50  |         | M       |         |         | Inquiry screen |
| S7  | 50  |         | M       |         |         | Inquiry screen |
| S8  | 50  |         | M       |         |         | Inquiry screen |
| S9  | 50  |         | M       |         |         | Inquiry screen |

### Field 57 Payment
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Desc            |
|-----|-----|---------|---------|---------|---------|-----------------|
| R0  | 50  |         |         |         | M       | Payment receipt |
| R1  | 50  |         |         |         | M       | Payment receipt |
| R2  | 50  |         |         |         | M       | Payment receipt |
| R3  | 50  |         |         |         | M       | Payment receipt |
| R4  | 50  |         |         |         | M       | Payment receipt |
| R5  | 50  |         |         |         | M       | Payment receipt |
| R6  | 50  |         |         |         | M       | Payment receipt |
| R7  | 50  |         |         |         | M       | Payment receipt |
| R8  | 50  |         |         |         | M       | Payment receipt |
| R9  | 50  |         |         |         | M       | Payment receipt |
| RA  | 50  |         |         |         | M       | Payment receipt |
| RB  | 50  |         |         |         | M       | Payment receipt |
| RC  | 50  |         |         |         | M       | Payment receipt |
| RD  | 50  |         |         |         | M       | Payment receipt |
| RE  | 50  |         |         |         | M       | Payment receipt |
| RF  | 50  |         |         |         | M       | Payment receipt |
| RG  | 50  |         |         |         | M       | Payment receipt |
| RH  | 50  |         |         |         | M       | Payment receipt |
| RI  | 50  |         |         |         | M       | Payment receipt |

# Transaction Message Format

### Inquiry Request

**ISO Message**

```
0200B23A400108818000000000001000000A370000000000000000112707241600000414241311271128601106360003000000000248KOI             089PI0501001CN1100000000000PM0212SB1112345678901SD1112345678901AC042500C1011C21252007776459436007123456700130071234567
```

**Parsed Message**

```json
{
    "3": "370000",
    "4": "000000000000",
    "7": "1127072416",
    "11": "000004",
    "12": "142413",
    "13": "1127",
    "15": "1128",
    "18": "6011",
    "32": "360003",
    "37": "000000000248",
    "41": "KOI             ",
    "48": "PI0501001CN1100000000000PM0212SB1112345678901SD1112345678901AC042500C1011C212520077764594",
    "49": "360",
    "100": "1234567",
    "125": "",
    "127": "071234567"
}
```

### Inquiry Response

**ISO Message**

```
0210B23A40010A818080000000001000010A37000000000000000011270724160000041424131127112860110636000300000000024800KOI             104NM11Wajan IrengSD1112345678901AC042500PI0501001CN1100000000000PM0212C1011SB1112345678901C212520077764594360291S326IDPEL       : 520077764594S425NAMA        : Wajan IrengS530TARIF/DAYA  : B1 /000001300 VAS632TKN UNSOLD1 : RP               0S732TKN UNSOLD2 : RP               0S832NOMINAL     : RP               0S932ADMIN BANK  : RP           2.500S022PEMBELIAN PLN PRABAYARS100S220BILLER : PLN PREPAID071234567032367a98661e10f4a51e1675bf4399fceb00130071234567
```

**Parse Message**

```json
{
    "3": "370000",
    "4": "000000000000",
    "7": "1127072416",
    "11": "000004",
    "12": "142413",
    "13": "1127",
    "15": "1128",
    "18": "6011",
    "32": "360003",
    "37": "000000000248",
    "39": "00",
    "41": "KOI             ",
    "48": "NM11Wajan IrengSD1112345678901AC042500PI0501001CN1100000000000PM0212C1011SB1112345678901C212520077764594",
    "49": "360",
    "57": "S326IDPEL       : 520077764594S425NAMA        : Wajan IrengS530TARIF/DAYA  : B1 /000001300 VAS632TKN UNSOLD1 : RP               0S732TKN UNSOLD2 : RP               0S832NOMINAL     : RP               0S932ADMIN BANK  : RP           2.500S022PEMBELIAN PLN PRABAYARS100S220BILLER : PLN PREPAID",
    "100": "1234567",
    "120": "367a98661e10f4a51e1675bf4399fceb",
    "125": "",
    "127": "071234567"
}
```

### Payment Request

**ISO Message**
```
0200B23A400108818080000000001000010A800000000000102500112707251600000414251311271128601106360003000000000248KOI             128NM11Wajan IrengPI0501001CN1100000000000PM0212SI102345678901SB1112345678901SD1112345678901AT06100000AC042500C1011C212520077764594360283S022PEMBELIAN PLN PRABAYARS220BILLER : PLN PREPAIDS326IDPEL       : 520077764594S425NAMA        : Wajan IrengS530TARIF/DAYA  : B1 /000001300 VAS631TKN UNSOLD1 : RP               S731TKN UNSOLD2 : RP               S831NOMINAL     : RP               S931ADMIN BANK  : RP           2.50071234567032367a98661e10f4a51e1675bf4399fceb00130071234567
```

**Parsed Message**
```json
{
    "3": "800000",
    "4": "000000102500",
    "7": "1127072516",
    "11": "000004",
    "12": "142513",
    "13": "1127",
    "15": "1128",
    "18": "6011",
    "32": "360003",
    "37": "000000000248",
    "41": "KOI             ",
    "48": "NM11Wajan IrengPI0501001CN1100000000000PM0212SI102345678901SB1112345678901SD1112345678901AT06100000AC042500C1011C212520077764594",
    "49": "360",
    "57": "S022PEMBELIAN PLN PRABAYARS220BILLER : PLN PREPAIDS326IDPEL       : 520077764594S425NAMA        : Wajan IrengS530TARIF/DAYA  : B1 /000001300 VAS631TKN UNSOLD1 : RP               S731TKN UNSOLD2 : RP               S831NOMINAL     : RP               S931ADMIN BANK  : RP           2.50",
    "100": "1234567",
    "120": "367a98661e10f4a51e1675bf4399fceb",
    "125": "",
    "127": "071234567"
}
```

### Payment Response

**ISO Mesaage**
```
0210B23A40010A818080000000001000010A80000000000010000011270725160000041425131127112860110636000300000000024800KOI             128NM11Wajan IrengSD1112345678901AC042500AT06100000SI102345678901PI0501001CN1100000000000PM0212C1011SB1112345678901C212520077764594360607R225NAMA        : Wajan IrengR330TARIF/DAYA  : B1 /000001300 VAR429JML KWH     :           84,62R525NO METER    : 10077764594R634NO REFF     : 0UAK2642C18E2E94501BR700R832NOMINAL     : RP         100.000R932ADMIN BANK  : RP           2.500RA32TOTAL BAYAR : RP         102.500RB32TOKEN : 5438 8233 9306 8891 3258RC32RP BAYAR    : RP         102.500RD32MATERAI     : RP        6.000,00RE32PPN         : RP        8.173,91RF32PPJ         : RP        4.086,96RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27  BUKTI PEMBAYARAN YANG SAHR020BILLER : PLN PREPAIDR126IDPEL       : 520077764594071234567032367a98661e10f4a51e1675bf4399fceb00130071234567
```

**Parsed Message**
```json
{
    "3": "800000",
    "4": "000000100000",
    "7": "1127072516",
    "11": "000004",
    "12": "142513",
    "13": "1127",
    "15": "1128",
    "18": "6011",
    "32": "360003",
    "37": "000000000248",
    "39": "00",
    "41": "KOI             ",
    "48": "NM11Wajan IrengSD1112345678901AC042500AT06100000SI102345678901PI0501001CN1100000000000PM0212C1011SB1112345678901C212520077764594",
    "49": "360",
    "57": "R225NAMA        : Wajan IrengR330TARIF/DAYA  : B1 /000001300 VAR429JML KWH     :           84,62R525NO METER    : 10077764594R634NO REFF     : 0UAK2642C18E2E94501BR700R832NOMINAL     : RP         100.000R932ADMIN BANK  : RP           2.500RA32TOTAL BAYAR : RP         102.500RB32TOKEN : 5438 8233 9306 8891 3258RC32RP BAYAR    : RP         102.500RD32MATERAI     : RP        6.000,00RE32PPN         : RP        8.173,91RF32PPJ         : RP        4.086,96RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27  BUKTI PEMBAYARAN YANG SAHR020BILLER : PLN PREPAIDR126IDPEL       : 520077764594",
    "100": "1234567",
    "120": "367a98661e10f4a51e1675bf4399fceb",
    "125": "",
    "127": "071234567"
}
```

# Appendix

## MTI

| MTI  | Description                     |
| ---- | ------------------------------- |
| 0200 | Financial Transaction Request   |
| 0210 | Financial Transaction Response  |
| 0220 | Advice Request                  |
| 0221 | Advice Request Repeat           |
| 0230 | Advice Response                 |
| 0240 | Notification                    |
| 0400 | Reversal Interactive            |
| 0401 | Reversal Interactive Repeat     |
| 0410 | Reversal Interactive Response   |
| 0420 | Reversal Advice                 |
| 0421 | Reversal Advice Repeat          |
| 0430 | Reversal Ack                    |
| 0800 | Network Management Request      |
| 0810 | Network Management Response     |

## Processing Code

This describes the type of transaction and the type of accounts affected by the transaction.

Positions 1 to 2 (Transaction Code) 
| Transaction Code | Description |
| ---------------- | ----------- |
| 37               | Inquiry     |
| 80               | Payment     |

Positions 3 to 4 (From Account Type)
| From Account Type | Description        |
| ----------------- | ------------------ |
| 00                | Unspecified *      |
| 10                | Savings *          |
| 20                | Checking *         |
| 30                | Credit *           |
| 40                | Universal Account  |
| 50                | Investment Account |
| 60                | E Money*           |

Note: * is allowable account type

Positions 5 to 6 (To Account Type) 

| To Account Type | Description |
| --------------- | ----------- |
| 00              | Unspecified |

## Channel Type

| Code | Description                      |
| ---- | -------------------------------- |
| 6010 | Teller                           |
| 6011 | ATM                              |
| 6012 | Point of Sales                   |
| 6013 | Auto Debet / Giralisasi          |
| 6014 | Internet Banking                 |
| 6015 | KIOSK (keliling)                 |
| 6016 | Phone Banking                    |
| 6017 | Mobile Banking / SMS Banking     |
| 6018 | EDC                              |
| 6019 | Automatic Deposit Machine        |
| 6021 | PC PoS (PPOB)                    |
| 6022 | EDC PoS (PPOB)                   |
| 6023 | SMS PoS (PPOB)                   |
| 6024 | Modern Channel Pos (PPOB)        |
| 6025 | Mobile PoS (PPOB)                |

## Response Code

| Code | Description                                 |
| -- | ----------------------------------------------|
| 00 | Success                                       |
| 01 | Need To Sign On                               |
| 02 | Internal To Provider Down                     |
| 03 | Request Time Out                              |
| 05 | Do Not Honor                                  |
| 07 | Invalid Mandatory Field                       |
| 08 | Invalid Message Type Indicator                |
| 09 | Incomplete Message                            |
| 10 | Preset Maximum Transaction Loaded Exceeded    |
| 12 | Failed To Reverse Transaction Committed 1     |
| 13 | Invalid Amount                                |
| 14 | Invalid Card Number                           |
| 21 | Advice Failed                                 |
| 24 | Invalid Processing Code                       |
| 25 | Invalid Merchant Type                         |
| 26 | Invalid Institution Identification Code       |
| 30 | Invalid Product Code                          |
| 31 | Invalid Customer Id 1                         |
| 32 | Invalid Network Management Information Code   |
| 33 | Invalid Customer Id 2                         |
| 34 | Invalid Customer Id 3                         |
| 35 | Invalid Customer Id 4                         |
| 36 | Invalid Customer Id 5                         |
| 41 | Lost Card                                     |
| 43 | Invalid Card                                  |
| 48 | No Payment Record 1                           |
| 49 | Invalid Reversal No Payment Record 2          |
| 50 | Failed To Reverse Transaction Committed 2     |
| 51 | Failed To Reverse Transaction Committed 3     |
| 54 | Bill Already Paid                             |
| 55 | Bill Not Available 2                          |
| 56 | No Card Record                                |
| 58 | Transaction Not Permit                        |
| 59 | Invalid Transaction Amount                    |
| 60 | Invalid Minimum Amount 1                      |
| 61 | Invalid Minimum Amount 2                      |
| 62 | Invalid District Code                         |
| 65 | Exceeded Transaction Frequency Limit          |
| 68 | Response Receive Too Late                     |
| 69 | Unable Decrypt Track 2                        |
| 70 | MLPO Reference Number Already Exist           |
| 71 | Bill Amount Le Zero                           |
| 72 | Over quota                                    |
| 74 | Customer Id Suspended                         |
| 75 | Pin Tries Exceeded                            |
| 76 | Insufficient Fund 2                           |
| 81 | Cut Off                                       |
| 85 | Card Is Inactive                              |
| 86 | Card Is Blocked                               |
| 88 | Bill Not Available 1                          |
| 89 | Link To Service Provider Down                 |
| 90 | Cut Off In Progress                           |
| 91 | Payment Without Success Inquiry               |
| 92 | Transaction Not Found                         |
| 94 | Duplicated Transmission                       |
| 96 | System Malfunction                            |
| 97 | Invalid ARQC                                  |
| 98 | Internal Provider System Error                |
| 99 | Internal Ba Server Error                      |
