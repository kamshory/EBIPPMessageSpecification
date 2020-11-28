
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
0200B23A400108818000000000001000000A370000000000000000112707241600000414241311281128601206360003000000000248KOI             070PI06013026CN12081228812348PM0212SB1112345678901SD1112345678901AC04250036007123456700009071234567
```

**Parsed Message**

```json
{
	"f3": "370000",
	"f4": "000000000000",
	"f7": "1127072416",
	"f11": "000004",
	"f12": "142413",
	"f13": "1128",
	"f15": "1128",
	"f18": "6012",
	"f32": "360003",
	"f37": "000000000248",
	"f41": "KOI ",
	"f48": "PI06013026CN12081228812348PM0212SB1112345678901SD1112345678901AC042500",
	"f49": "360",
	"f100": "1234567",
	"f125": "",
	"f127": "071234567"
}
```

### Inquiry Response

**ISO Message**

```
0210B23A40010A818080000000001000000A37000000000013898311270724160000041424131128112860120636000300000000024800KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360296S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN    : 081228812348S333NAMA               : KAMSHORY ROYS431TAGIHAN            : Rp 138.983S534PERIODE            : November 2020S631JATUH TEMPO        : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG07123456700009071234567
```

**Parse Message**

```json
{
	"f3": "370000",
	"f4": "000000138983",
	"f7": "1127072416",
	"f11": "000004",
	"f12": "142413",
	"f13": "1128",
	"f18": "6012",
	"f15": "1128",
	"f32": "360003",
	"f37": "000000000248",
	"f39": "00",
	"f41": "KOI ",
	"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
	"f49": "360",
	"f57": "S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN : 081228812348S333NAMA : KAMSHORY ROYS431TAGIHAN : Rp 138.983S534PERIODE : November 2020S631JATUH TEMPO : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG",
	"f100": "1234567",
	"f125": "",
	"f127": "071234567"
}
```

### Payment Request

**ISO Message**
```
0200B23A40010A818080000000001000000A80000000000013898311270724160000041424131128112860120636000300000000024803KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY36000007123456700009071234567
```

**Parsed Message**
```json
{
	"f3": "800000",
	"f4": "000000138983",
	"f7": "1127072416",
	"f11": "000004",
	"f12": "142413",
	"f13": "1128",
	"f18": "6012",
	"f15": "1128",
	"f32": "360003",
	"f37": "000000000248",
	"f39": "00",
	"f41": "KOI ",
	"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
	"f49": "360",
	"f57": "",
	"f100": "1234567",
	"f125": "",
	"f127": "071234567"
}
```

### Payment Response

**ISO Mesaage**
```
0210B23A40010A818080000000001000000A80000000000013898311270724160000041424131128112860120636000300000000024800KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360320R027STRUK PEMBAYARAN KARTU HALOR140========================================R233NOMOR PELANGGAN    : 081228812348R333NAMA               : KAMSHORY ROYR431TAGIHAN            : Rp 138.983R534PERIODE            : November 2020R646WAKTU PEMBAYARAN   : 28 November 2020 19:19:26R700R840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG07123456700009071234567
```

**Parsed Message**
```json
{
	"f3": "800000",
	"f4": "000000138983",
	"f7": "1127072416",
	"f11": "000004",
	"f12": "142413",
	"f13": "1128",
	"f15": "1128",
	"f18": "6012",
	"f32": "360003",
	"f37": "000000000248",
	"f39": "00",
	"f41": "KOI ",
	"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
	"f49": "360",
	"f57": "R027STRUK PEMBAYARAN KARTU HALOR140========================================R233NOMOR PELANGGAN : 081228812348R333NAMA : KAMSHORY ROYR431TAGIHAN : Rp 138.983R534PERIODE : November 2020R646WAKTU PEMBAYARAN : 28 November 2020 19:19:26R700R840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG",
	"f100": "1234567",
	"f125": "",
	"f127": "071234567"
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
| 41 | Lost Card                                     |
| 43 | Invalid Card                                  |
| 48 | No Payment Record 1                           |
| 54 | Bill Already Paid                             |
| 56 | No Card Record                                |
| 58 | Transaction Not Permit                        |
| 59 | Invalid Transaction Amount                    |
| 60 | Invalid Minimum Amount 1                      |
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
