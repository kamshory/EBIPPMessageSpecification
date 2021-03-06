# Topology

The following is sequence diagram for inquiry process

```mermaid
sequenceDiagram
Collection Agent ->> AltoPay Biller: Inquiry Request
AltoPay Biller->>Biller: Inquiry Request
Biller->> AltoPay Biller: Inquiry Response
AltoPay Biller->> Collection Agent: Inquiry Response
```

The following is sequence diagram for payment process

```mermaid
sequenceDiagram
Collection Agent ->> AltoPay Biller: Payment Request
AltoPay Biller->>Biller: Payment Request
Biller->> AltoPay Biller: Payment Response
AltoPay Biller->> Collection Agent: Payment Response
Note right of AltoPay Biller: If Biller has not <br>responded to a <br>request for a long <br>time, AltoPay Biller <br>will send an Advice<br>Request to Biller
AltoPay Biller->>Biller: Advice Request
Biller->> AltoPay Biller: Advice Response
AltoPay Biller->>Collection Agent : Payment Response
Note right of Collection Agent: If AltoPay Biller has <br>not responded to a <br>request for a long <br>time, Collection <br>Agent will send an <br>Advice Request to <br>Biller
Collection Agent ->> AltoPay Biller: Advice Request
AltoPay Biller->> Collection Agent: Advice Response

```

The following is sequence diagram for advice process

```mermaid
sequenceDiagram
Collection Agent ->> AltoPay Biller: Advice Request
AltoPay Biller->>Biller: Advice Request
Biller->> AltoPay Biller: Advice Response
AltoPay Biller->> Collection Agent: Advice Response
Note right of AltoPay Biller: If AltoPay Biller ha<br>no information<br>about payment <br>response, AltoPay <br>Biller will send an <br>Advice Request <br>to Biller
AltoPay Biller->>Biller: Advice Request
Biller->> AltoPay Biller: Advice Response
AltoPay Biller->>Collection Agent : Advice Response
```

# Message

AltoPay Biller uses the ISO 8583 message format for transactions. Because ISO 8583 is designed for asynchronous communication, it takes a message header to separate one message from another message sent sequentially in a single connection. **Message headers and messages must be sent as a package to avoid other _thread_ send other data so that causing message headers and messages not to be sent sequentially**.

## Message Header Definition

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

Note: message header is binary data and not always printable. Be careful when `copy` and `paste` it.

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
}
```

# ISO 8583 Data Element

| DE | Name | Type | Remark | 0200 | 0210 | 0220 | 0230 |
|---|---|---|---|---|---|---|---|
| - | Message Type | N 4 | ‘0200’ / ‘0220’ (Request) | M | M | M | M |
| - | Primary Bit Map | AN 16 | Mandatory for all Messages | M | M | M | M |
| P1 | Secondary Bit Map | AN 16 | Only present if any of DE 65 to DE 128 are present. | M | M | M | M |
| P2 | Primary Account Number | N..19 LLVAR | PAN | M | ME | ME | ME |
| P3 | Processing Code | N 6 | EBIPP: <br>‘37xxxx’ (Inquiry)<br>‘80xxxx’ (Payment) <br>‘80xxxx’ (Advice) | M | M | ME | ME |
| P4 | Transaction Amount | N 12 | Inquiry: set all zeroes Payment: Debited Amount Reversal: Debited Amount | M | M | ME | ME |
| | | |  | | | | |
| P7 | Transmission Date and | N 10 | In GMT | M | M | M | M |
| | Time | | Format: MMDDhhmmss | | | | |
| P11 | System Trace Audit Number | N 6 | Transaction Trace Number. This is Unique per message transmitted. | M | ME | M | ME |
| P12 | Local Transaction Time | N 6 | Format: hhmmss | M | ME | ME | ME |
| P13 | Local Transaction Date | N 4 | Format: MMDD | M | ME | ME | ME |
| P14 | Expiration Date | N 4 | For Future Use | C | CE | CE | CE |
| P15 | Settlement Date | N 4 | Format: MMDD | M | ME | ME | ME |
| P18 | Channel Type | N 4 |  | M | ME | ME | ME |
| P22 | Point of Service Entry Mode | N 3 |  | M | ME | ME | ME |
| P25 | Point of Service Condition Code | N 2 |  | C | CE | CE | CE |
| P32 | Acquiring Institution ID | N..11 LLVAR |  | M | ME | ME | ME |
| P33 | Forwarding Institution ID | N..11 LLVAR |  | M | ME | ME | ME |
| P35 | Track-2 Data | Z..37 LLVAR | Inquiry: OFF <br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF <br> | C | - | - | - |
| P37 | Retrieval Reference Number | AN 12 |  | M | ME | ME | ME |
| P39 | Response Code | AN 2 | Response Code | | M | | M |
| P41 | Card Acceptor Terminal Identification | ANS 16 |  | M | ME | ME | ME |
| P42 | Card Acceptor ID | ANS 15 |  | M | ME | ME | ME |
| P43 | Card Acceptor Name/Location | ANS 40 |  | M | ME | | |
| P48 | Additional Data | ANS..999 LLLVAR |  | M | M | ME | ME |
| P49 | Transaction Currency Code | N 3 |  | M | ME | ME | ME |
| P52 | Personal Identification Number (PIN) Data | AN 16 | Inquiry: OFF<br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF<br> | C | - | - | - |
| P55 | ICC Data | ANS..765 | Inquiry: OFF <br>Payment: Debit=ON, Credit=OFF Reversal: Debit=ON, Credit=OFF<br> | C | C | - | - |
| P57 | Data Payment – National | ANS..999 LLLVAR |  | ME | ME | ME | ME |
| S90 | Original Data Element | N--42 |  | | | M | ME |
| S100 | Receiving Institution Identification Code | N..11 LLVAR | Issuer Code <br> | M | ME | ME | ME |
| S120 | Key Management | N...999 LLLVAR | Key to make a transaction sent by key exchange response or new key sent by AltoPay Biller | M | ME | ME | ME |
| S125 | Transaction Indicator | N..255 LLLVAR |  | M | ME | ME | ME |
| S127 | Destination Institution Identification Code | N..11 LLLVAR | Biller Identification Code <br> | M | ME | ME | ME |

## Additional Data

AltoPay Biller use TLV (Tag-Length-Value) for some data elements. `Tag` is 2 first bytes represented by `alpha numeic`. `Length` is second 2 bytes represented by `decimal` padded with zero at left side. `Value` is 7 bit ASCII. For binary data, use _base 64 encoding_ or _hexa decimal encoding_ instead.

**Data Building**

For example, we have data bellow:

| Tag | Value                                |
| --- | ------------------------------------ |
| PI  | 987654                               |
| CN  | 081198765432                         |
| AT  | 65000                                |

So, we can write that data become:

`PI06987654CN12081198765432AT0565000`

**Data Parsing**

For example, he have data bellow:

`PI06987654CN12081198765432AT0565000`

We can parse that with algorithm:

```
CURRENT OFFSET = 0
REPEAT
	READ 2 BYTE FROM CURRENT OFFSET AND MAKE IS AS TAG
	ADD CURRENT OFFSET WITH 2
	READ 2 BYTE FROM CURRENT OFFSET AND MAKE IT AS DECIMAL NUMBER REPRESENT DATA LENGTH
	READ DATA ACCORDING TO ITS LENGTH AND PUT IT ON CURRENT TAG
	ADD CURRENT OFFSET WITH DATA LENGTH
UNTIL CURRENT OFFSET < RAW DATA LENGTH
```

From algorithm above, we get:
```
P1=987654
CD=081198765432
AT=65000
```

**Data Building and Data Parsing Using Java**

```java
public class TLV {
	
private TLV()
{
}
public static JSONObject parse(String data)
{
	int remainingLength = 0;
	int nextOfset = 0;
	String remaining = data;
	JSONObject values = new JSONObject();
	String tag = "";
	String len = "";
	String val = "";
	int itemLength = 0;
	while(remaining.length() > 4)
	{
		tag = remaining.substring(0, 2);
		len = remaining.substring(2, 4);
		try
		{
			itemLength = Integer.parseInt(len);
		}
		catch(NumberFormatException e)
		{
			itemLength = 0;
		}
		remainingLength = remaining.length();
		nextOfset = 4+itemLength;
		if(nextOfset <= remainingLength)
		{
			val = remaining.substring(4, nextOfset);
			remaining = remaining.substring(nextOfset);
		}
		else
		{
			val = remaining.substring(4);
			remaining = "";
		}
		values.put(tag, val);
	}
	return values;		
}

public static String build(JSONObject jsonObj) 
{
	StringBuilder result = new StringBuilder();
	String tag = "";
	String value = "";
	int length = 0;
	for (String key : jsonObj.keySet()) 
	{
	        String keyStr = key;
	        Object keyvalue = jsonObj.get(keyStr);
	        tag = keyStr;
	        if(keyvalue instanceof String)
	        {
			value = keyvalue.toString();
			length = value.length();
			result.append(String.format("%-2s%02d%s", tag, length, value));	            
	        }
	}		
	return result.toString();
}
}
```

# Message Specification

## Network Management Message

Network Management Message only applied on asynchronous transaction which keep and maintain its connection. 

| DE  | Type               | Description                          | 0800 | 0810 |
| --- | ------------------ | ------------------------------------ | ---- | ---- |
| 7   | N 10               | Transmission Date and Time           | M    | ME   |
| 11  | N 6                | System Trace Audit Number            | M    | ME   |
| 32  | ANS .. 99 LLVAR    | Acquiring Institution Code           | M    | ME   |
| 39  | N 2                | Response Code                        | -    | M   |
| 48  | ANS ... 999 LLLVAR | Additional Data                      | C    | CE   |
| 70  | N 3                | Network Management Information Code  | M    | ME   |
| 120 | ANS ... 999 LLLVAR | Key Management                       | -    | C   |

Field 48 on Network Management Request sent by client contains
| Tag | Value                        | 0800 | 0810 |
| --- | ---------------------------- | ---- | ---- |
| AK  | API Key                      | M    | ME   |
| TS  | Timestamp in ISO Format      | M    | ME   |
| SN  | Signature                    | M    | ME   |

Credentials information required:
1. Client code (sent to server via field  32)
2. API Key (sent to server via field  48)
3. Validation Key (not sent to server, required to create signature)

Field 48 not required on echo test.

Field 70 is Network Management Information Code

Data element for network management  request

| Code | Description  | 7  | 11 | 32 | 39 | 48 | 70 | 120 | 
| ---- | ------------ | -- | -- | -- | -- | -- | -- | --- |
| 001  | Logon        | M  | M  | M  | -  | M  | M  | -   |
| 002  | Logoff       | M  | M  | M  | -  | M  | M  | -   |
| 161  | Key Exchange | M  | M  | M  | -  | M  | M  | -   |
| 162  | New Key      | M  | M  | M  | -  | M  | M  | M   |
| 201  | Cutover      | M  | M  | M  | -  | M  | M  | -   |
| 301  | Echo Test    | M  | M  | M  | -  | -  | M  | -   |
 
Data element for network management  response

| Code | Description  | 7  | 11 | 32 | 39 | 48 | 70 | 120 |
| ---- | ------------ | -- | -- | -- | -- | -- | -- | --- |
| 001  | Logon        | M  | M  | M  | M  | M  | M  | M   |
| 002  | Logoff       | M  | M  | M  | M  | M  | M  | -   |
| 161  | Key Exchange | M  | M  | M  | M  | M  | M  | M   |
| 162  | New Key      | M  | M  | M  | M  | M  | M  | -   |
| 201  | Cutover      | M  | M  | M  | M  | M  | M  | -   |
| 301  | Echo Test    | M  | M  | M  | M  | -  | M  | -   |
 

**API Key** is client API Key created by AltoPay Biller on registration.

**Timestamp** is ISO format of time stamp in UTC. Example `2020-12-31T23:59:59.999Z`.

**Signature** is hMac with SHA-256 of API Key and Timestamp. Validation key required as it password.

**Example:**

```java
String apiKey = "akey_7HgyugUyfuT";
String validationKey = "vkey_sidcsoi765djcos";
String timestamp = currentTime("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", "UTC");
String stringToSign = apiKey + ":" + timestamp;
String rawSignature = hMacSHA256(stringToSign, validationKey);
String signature = binToHex(rawSignature);
```

| Name             | Value                                                            |
| ---------------- | ---------------------------------------------------------------- |
| Client Code      | CA123                                                            |
| API Key          | akey_7HgyugUyfuT                                                 |
| Validation Key   | vkey_sidcsoi765djcos                                             |
| Time Stamp       | 2020-11-29T07:55:37.653Z                                         |
| String To Sign   | akey_7HgyugUyfuT:2020-11-29T07:55:37.653Z                        |
| Signature        | 871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285 |


Signature is hMac of API Key and Timestamp with Validation Key as its password.

### Logon
The client must send a `LOGON` request as soon as it connected. Server will respond the request.

Direction : from client to server

**Logon Request**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f48":"AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70":"001"
}
```

ISO Message = "080082200001000100000400000000000000112707241600000405CA123116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285001"

**Logon Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70": "001",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNTQ0fQ.oSHXM_Dh4_FCQWPMXXQzogml7NNgPE-UJxWYbNyTt9g"
}
```

ISO Message = "081082200001020100000400000000000100112707241600000405CA12300116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285001123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNjg1fQ.hhYZ53A1Q4yK6A4PeqIz23AfaGOgyzdH1Fare3eUNYM"

### Logoff
The client must send `LOGOFF` request when they will not send transaction again.

Direction : from client to server


**Logon Request**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f48":"AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70":"002"
}
```

ISO Message = "080082200001000100000400000000000000112707241600000405CA123116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285002"

**Logon Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70": "002",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNTQ0fQ.oSHXM_Dh4_FCQWPMXXQzogml7NNgPE-UJxWYbNyTt9g"
}
```

ISO Message = "081082200001020100000400000000000100112707241600000405CA12300116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285002123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNjg1fQ.hhYZ53A1Q4yK6A4PeqIz23AfaGOgyzdH1Fare3eUNYM"

### Echo Test

Echo Test is used to ensure that a socket connection can be used to send and receive information from both sides.

Direction : from client to server and from server to client

**Echo Test Request**
```json
{
	"f7":"1231232359",
	"f11":"123456",
	"f32":"CA123",
	"f70":"301"
}
```

ISO Message = "080082200001000000000400000000000000123123235912345605CA123301"

**Echo Test Response**
```json
{
	"f7":"1231232359",
	"f11":"123456",
	"f32":"CA123",
	"f39":"00",
	"f70":"301"
}
```

ISO Message = "081082200001020000000400000000000000123123235912345605CA12300301"

### Key Exchange

The client request key to the server. The key required to make financial transactions. Server only will process the transaction if key is valid.

Direction : from client to server 

**Key Exchange Request**

```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f48":"AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70":"161"
}
```

ISO Message = "080082200001000100000400000000000000112707241600000405CA123116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285161"

**Key Exchange Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70": "161",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNTQ0fQ.oSHXM_Dh4_FCQWPMXXQzogml7NNgPE-UJxWYbNyTt9g"
}
```

ISO Message = "081082200001020100000400000000000100112707241600000405CA12300116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285161123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNjg1fQ.hhYZ53A1Q4yK6A4PeqIz23AfaGOgyzdH1Fare3eUNYM"

### New Key

Server will send new key periodically. Client must use newest key on next transaction. Client must response it with response code **SUCCESS (00)** to make sure that client receive the key.

Direction: from server to client

**Key Exchange Request**

```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f48":"AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70":"162"
}
```

ISO Message = "080082200001000100000400000000000000112707241600000405CA123116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285162"

**Key Exchange Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70": "162",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNTQ0fQ.oSHXM_Dh4_FCQWPMXXQzogml7NNgPE-UJxWYbNyTt9g"
}
```

ISO Message = "081082200001020100000400000000000100112707241600000405CA12300116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285162123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEyNjg1fQ.hhYZ53A1Q4yK6A4PeqIz23AfaGOgyzdH1Fare3eUNYM"

# Financial Transaction Message Format

## Data Elements

The following is a list of data elements

| DE   | Type and Length    | 200 I | 210 I | 200 P | 210 P | Description                                  |
| ---- | ------------------ |-------|-------|-------|-------| -------------------------------------------- |
| 3    | N 6                | M     | ME    | ME    | ME    | Processing Code                              |
| 4    | N 12               |       | M     | ME    | ME    | Amount                                       |
| 7    | N 10               | M     | ME    | ME    | ME    | Date and Time in GMT                         |
| 11   | N 6                | M     | ME    | ME    | ME    | System Trace Audit Number                    |
| 12   | N 6                | M     | ME    | ME    | ME    | Local Transaction Time                       |
| 13   | N 4                | M     | ME    | ME    | ME    | Local Transaction Date                       |
| 15   | N 4                | M     | ME    | ME    | ME    | Settlement Date                              |
| 18   | N 4                | M     | ME    | ME    | ME    | Channel Type                                 |
| 32   | AN .. 11 LLVAR     | M     | ME    | ME    | ME    | Acquiring Institution ID (Client Code)       |
| 37   | AN 12              | M     | ME    | ME    | ME    | Transaction Reference Number                 |
| 39   | N 2                |       | M     | M     | ME    | Response Code                                |
| 41   | AN 16              | M     | ME    | ME    | ME    | Card Acceptor Terminal Identification        |
| 48   | ANS ... 999 LLLVAR | M     | ME    | ME    | ME    | Additional Data                              |
| 49   | N 3                | M     | ME    | ME    | ME    | Transaction Currency Code                    |
| 57   | ANS ... 999 LLLVAR |       | M     |       | M     | Inquiry Screen and Payment Receipt           |
| 100  | AN .. 11 LLVAR     | M     | ME    | ME    | ME    | Receiving Institution Identification Code    |
| 120  | AN ... 999 LLLVAR  | M     |       | M     |       | Last Key Sent by Server                      |
| 125  | AN .. 255 LLLVAR   | M     | ME    | ME    | ME    | Transaction Indicator                        |
| 127  | AN .. 11 LLVAR     | M     | ME    | ME    | ME    | Destination Institution Identification Code  |

Note:

1. Field 32, 100, 125 and 127 will be informed
2. Field 48 and 57 are TLV
3. Field 39 only exists on response
4. Field 57 only exists on response
5. Field 120 only exists on request

### Field 3

Field 3 is Processing Code. See Appendix section.

### Field 4 

Field 4 is transaction amount. Transaction sent by server on inquiry response. Some product which does not have inquiry, amount sent by client on payment request or by server on payment response.

### Field 7 

Field 4 is Date and Time in GMT with format MMddHHmmss (on Java). For example, 27 December 2020 13:56:29 written `1227135629`.

### Field 11

Field 11 is System Trace Audit Number 6 digit with left padding.

### Field 12

Field 12 is Local Transaction Time with format HHmmss (on Java). For example, 27 December 2020 20:56:29 written `205629`. 

### Field 13

Field 12 is Local Transaction Date with format MMdd (on Java). For example, 27 December 2020 written `1227`. 

### Field 15

Field 12 is Settlement Date with format MMdd (on Java). For example, 28 December 2020 written `1228`. 

### Field 18

Field 18 is Channel Type. See Appendix section.

### Field 32 

Field 32 is filled by `Client Code`. Client code is generated by AltoPay Biller and will be informed to client.

### Field 37 

Field 37 is Transaction Reference Number. Use unix reference both in inquiry and payment. For advice, use reference number same with payment.

### Field 39

Field 39 is Response Code. See Appendix section.

### Field 41

Field 41 is Card Acceptor Terminal Identification.

### Field 48 
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| AC  | 12  | C     | CE    | CE    | CE    | Total admin fee                    |
| C1  | 24  | C     | CE    | CE    | CE    | Customer reference 1               |
| C2  | 24  | C     | CE    | CE    | CE    | Customer reference 2               |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

### Field 49

Field 49 is Transaction Currency Code according ISO 4217 in 3 digit number.

### Field 57 
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description     |
|-----|-----|-------|-------|-------|-------|-----------------|
| S0  | 50  |       | M     |       |       | Inquiry screen  |
| S1  | 50  |       | M     |       |       | Inquiry screen  |
| S2  | 50  |       | M     |       |       | Inquiry screen  |
| S3  | 50  |       | M     |       |       | Inquiry screen  |
| S4  | 50  |       | M     |       |       | Inquiry screen  |
| S5  | 50  |       | M     |       |       | Inquiry screen  |
| S6  | 50  |       | M     |       |       | Inquiry screen  |
| S7  | 50  |       | M     |       |       | Inquiry screen  |
| S8  | 50  |       | M     |       |       | Inquiry screen  |
| S9  | 50  |       | M     |       |       | Inquiry screen  |
| R0  | 50  |       |       |       | M     | Payment receipt |
| R1  | 50  |       |       |       | M     | Payment receipt |
| R2  | 50  |       |       |       | M     | Payment receipt |
| R3  | 50  |       |       |       | M     | Payment receipt |
| R4  | 50  |       |       |       | M     | Payment receipt |
| R5  | 50  |       |       |       | M     | Payment receipt |
| R6  | 50  |       |       |       | M     | Payment receipt |
| R7  | 50  |       |       |       | M     | Payment receipt |
| R8  | 50  |       |       |       | M     | Payment receipt |
| R9  | 50  |       |       |       | M     | Payment receipt |
| RA  | 50  |       |       |       | M     | Payment receipt |
| RB  | 50  |       |       |       | M     | Payment receipt |
| RC  | 50  |       |       |       | M     | Payment receipt |
| RD  | 50  |       |       |       | M     | Payment receipt |
| RE  | 50  |       |       |       | M     | Payment receipt |
| RF  | 50  |       |       |       | M     | Payment receipt |
| RG  | 50  |       |       |       | M     | Payment receipt |
| RH  | 50  |       |       |       | M     | Payment receipt |
| RI  | 50  |       |       |       | M     | Payment receipt |

### Field 100

Field 32 is filled by `Receiving Institution Code`. Receiving institution code will be informed to client.

### Field 120

Field 120 is key management. This value must be same with last key (field 120 on 0800 or 0810 message) sent by server. If client receive response code **Transaction Not Permit (58)**, client must send key exchange request to get new key.

### Field 125

Field 125 is transaction indicator. Transaction indicator format will be informed to client.

### Field 127

Field 127 is filled by `Destination Code`. Destination code will be informed to client.

## Sample Message

### Inquiry Request

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
"f32": "alto",
"f37": "000000000248",
"f41": "KOI ",
"f48": "PI06054501CN12522030064594AC042500",
"f49": "360",
"f100": "1234567",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzIyMTI5fQ.tyJ5WjfiuuG3AH5ugaXlNNXPlZocDJnPpsnC7x-12_M",
"f125": "",
"f127": "071234567"
}
```

ISO message: "0200B23A400108818000000000001000010A370000000000000000112707241600000414241311281128601204alto000000000248KOI             034PI06054501CN12522030064594AC042500360071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzIyMTI5fQ.tyJ5WjfiuuG3AH5ugaXlNNXPlZocDJnPpsnC7x-12_M00009071234567"

**TLV Data on Field 48 Request**
| TAG | VALUE        | 
| --- | ------------ | 
| AC  | 2500         | 
| PI  | 054501       | 
| CN  | 522030064594 | 

### Inquiry Response

```json
{
"f125": "071234567",
"f41": "KOI ",
"f32": "alto",
"f100": "1234567",
"f12": "142413",
"f120": "",
"f11": "000004",
"f13": "1128",
"f57": "S025INFORMASI TAGIHAN LISTRIKS100S220BILLER : PLN PREPAIDS326IDPEL : 522030064594S432NAMA : Customer 2 Bulan 4S525BL/TH : SEP20,OKT20S622TOTAL TGHN : 2 BULANS732RP TAG PLN : RP 690.000S832ADMIN BANK : RP 2.500S932TOTAL BAYAR : RP 692.500",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4",
"f18": "6012",
"f3": "370000",
"f39": "00",
"f4": "000000690000",
"f7": "1127072416",
"f127": ""
}
```

ISO message: "0210B23A40010A818080000000001000010A370000000000690000112707241600000414241311281128601204alto00000000024800KOI             082AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4360286S025INFORMASI TAGIHAN LISTRIKS100S220BILLER : PLN PREPAIDS326IDPEL       : 522030064594S432NAMA        : Customer 2 Bulan 4S525BL/TH       : SEP20,OKT20S622TOTAL TGHN  :  2 BULANS732RP TAG PLN  : RP         690.000S832ADMIN BANK  : RP           2.500S932TOTAL BAYAR : RP         692.50007123456700009071234567"

**TLV Data on Field 48 Response**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

**TLV Data on Field 57 Response**
| TAG | VALUE                            | 
| --- | -------------------------------- | 
| S0  | INFORMASI TAGIHAN LISTRIK        | 
| S1  |                                  | 
| S2  | BILLER : PLN PREPAID             | 
| S3  | IDPEL       : 522030064594       | 
| S4  | NAMA        : Customer 2 Bulan 4 | 
| S5  | BL/TH       : SEP20,OKT20        | 
| S6  | TOTAL TGHN  :  2 BULAN           | 
| S7  | RP TAG PLN  : RP         690.000 | 
| S8  | ADMIN BANK  : RP           2.500 | 
| S9  | TOTAL BAYAR : RP         692.500 | 

### Payment Request

```json
{
"f125": "071234567",
"f41": "KOI ",
"f32": "alto",
"f100": "1234567",
"f12": "142413",
"f11": "000004",
"f13": "1128",
"f57": "",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4",
"f18": "6012",
"f3": "800000",
"f4": "000000300000",
"f7": "1127072416",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzI2MzMxfQ.zuX4w0HSPVrJS7zLZMETsjQr1a7J1YGbdVZDECrPieE",
"f127": ""
}
```

ISO message: "0200B23A400108818080000000001000010A800000000000300000112707241600000414241311281128601204alto000000000248KOI             082AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4360000071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzI2MzMxfQ.zuX4w0HSPVrJS7zLZMETsjQr1a7J1YGbdVZDECrPieE09071234567000"

**TLV Data on Field 48 Request**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

### Payment Response

```json
{
"f125": "",
"f41": "KOI ",
"f32": "alto",
"f100": "1234567",
"f12": "142413",
"f120": "",
"f11": "000004",
"f13": "1128",
"f57": "R021BILLER : PLN POSTPAIDR126IDPEL : 522030064594R232NAMA : Customer 2 Bulan 4R325BL/TH : SEP20,OKT20R421STAND METER : 100-330R500R622TOTAL TGHN : 2 BULANR730TARIF/DAYA : B1 /000001300 VAR834NO REFF : 0UAK73F968AFA20E3B24R900RA32RP TAG PLN : RP 300.000RB00RC32ADMIN BANK : RP 2.500RD32TOTAL BAYAR : RP 302.500RE00RF00RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27 BUKTI PEMBAYARAN YANG SAH",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4",
"f18": "6012",
"f3": "800000",
"f39": "00",
"f4": "000000300000",
"f7": "1127072416",
"f127": ""
}
```

ISO message: "0210B23A40010A818080000000001000010A800000000000300000112707241600000414241311281128601204alto00000000024800KOI             082AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4360469R021BILLER : PLN POSTPAIDR126IDPEL       : 522030064594R232NAMA        : Customer 2 Bulan 4R325BL/TH       : SEP20,OKT20R421STAND METER : 100-330R500R622TOTAL TGHN  :  2 BULANR730TARIF/DAYA  : B1 /000001300 VAR834NO REFF     : 0UAK73F968AFA20E3B24R900RA32RP TAG PLN  : RP         300.000RB00RC32ADMIN BANK  : RP           2.500RD32TOTAL BAYAR : RP         302.500RE00RF00RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27  BUKTI PEMBAYARAN YANG SAH07123456709071234567000"

**TLV Data on Field 48 Response**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

**TLV Data on Field 57 Response**
| TAG | VALUE                              | 
| --- | ---------------------------------- | 
| R0  | BILLER : PLN POSTPAID              | 
| R1  | IDPEL       : 522030064594         | 
| R2  | NAMA        : Customer 2 Bulan 4   | 
| R3  | BL/TH       : SEP20,OKT20          | 
| R4  | STAND METER : 100-330              | 
| R5  |                                    | 
| R6  | TOTAL TGHN  :  2 BULAN             | 
| R7  | TARIF/DAYA  : B1 /000001300 VA     | 
| R8  | NO REFF     : 0UAK73F968AFA20E3B24 | 
| R9  |                                    | 
| RA  | RP TAG PLN  : RP         300.000   | 
| RB  |                                    | 
| RC  | ADMIN BANK  : RP           2.500   | 
| RD  | TOTAL BAYAR : RP         302.500   | 
| RE  |                                    | 
| RF  |                                    | 
| RG  | PERUSAHAAN LISTRIK NEGARA(PLN)     | 
| RH  |  MENYATAKAN STRUK INI SEBAGAI      | 
| RI  |   BUKTI PEMBAYARAN YANG SAH        | 


### Advice Request

```json
{
"f125": "071234567",
"f41": "KOI ",
"f32": "alto",
"f100": "1234567",
"f12": "142413",
"f11": "000004",
"f13": "1128",
"f57": "",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4",
"f18": "6012",
"f3": "800000",
"f4": "000000300000",
"f7": "1127072416",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzI2MzMxfQ.zuX4w0HSPVrJS7zLZMETsjQr1a7J1YGbdVZDECrPieE",
"f127": ""
}
```

ISO message: "0220B23A400108818080000000001000010A800000000000300000112707241600000414241311281128601204alto000000000248KOI             082AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4360000071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzI2MzMxfQ.zuX4w0HSPVrJS7zLZMETsjQr1a7J1YGbdVZDECrPieE09071234567000"

**TLV Data on Field 48 Request **
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

### Advice Response

```json
{
"f125": "",
"f41": "KOI ",
"f32": "alto",
"f100": "1234567",
"f12": "142413",
"f120": "",
"f11": "000004",
"f13": "1128",
"f57": "R021BILLER : PLN POSTPAIDR126IDPEL : 522030064594R232NAMA : Customer 2 Bulan 4R325BL/TH : SEP20,OKT20R421STAND METER : 100-330R500R622TOTAL TGHN : 2 BULANR730TARIF/DAYA : B1 /000001300 VAR834NO REFF : 0UAK73F968AFA20E3B24R900RA32RP TAG PLN : RP 300.000RB00RC32ADMIN BANK : RP 2.500RD32TOTAL BAYAR : RP 302.500RE00RF00RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27 BUKTI PEMBAYARAN YANG SAH",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4",
"f18": "6012",
"f3": "800000",
"f39": "00",
"f4": "000000300000",
"f7": "1127072416",
"f127": ""
}
```

ISO message: "0230B23A40010A818080000000001000010A800000000000300000112707241600000414241311281128601204alto00000000024800KOI             082AC042500PI06054501CN12522030064594FR12000000000016FS06000016NM18Customer 2 Bulan 4360469R021BILLER : PLN POSTPAIDR126IDPEL       : 522030064594R232NAMA        : Customer 2 Bulan 4R325BL/TH       : SEP20,OKT20R421STAND METER : 100-330R500R622TOTAL TGHN  :  2 BULANR730TARIF/DAYA  : B1 /000001300 VAR834NO REFF     : 0UAK73F968AFA20E3B24R900RA32RP TAG PLN  : RP         300.000RB00RC32ADMIN BANK  : RP           2.500RD32TOTAL BAYAR : RP         302.500RE00RF00RG30PERUSAHAAN LISTRIK NEGARA(PLN)RH29 MENYATAKAN STRUK INI SEBAGAIRI27  BUKTI PEMBAYARAN YANG SAH07123456709071234567000"

**TLV Data on Field 48 Response**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

**TLV Data on Field 57 Response**
| TAG | VALUE                              | 
| --- | ---------------------------------- | 
| R0  | BILLER : PLN POSTPAID              | 
| R1  | IDPEL       : 522030064594         | 
| R2  | NAMA        : Customer 2 Bulan 4   | 
| R3  | BL/TH       : SEP20,OKT20          | 
| R4  | STAND METER : 100-330              | 
| R5  |                                    | 
| R6  | TOTAL TGHN  :  2 BULAN             | 
| R7  | TARIF/DAYA  : B1 /000001300 VA     | 
| R8  | NO REFF     : 0UAK73F968AFA20E3B24 | 
| R9  |                                    | 
| RA  | RP TAG PLN  : RP         300.000   | 
| RB  |                                    | 
| RC  | ADMIN BANK  : RP           2.500   | 
| RD  | TOTAL BAYAR : RP         302.500   | 
| RE  |                                    | 
| RF  |                                    | 
| RG  | PERUSAHAAN LISTRIK NEGARA(PLN)     | 
| RH  |  MENYATAKAN STRUK INI SEBAGAI      | 
| RI  |   BUKTI PEMBAYARAN YANG SAH        | 

# Product Message Specification

In this session, we will only discuss field  48 and field  57 of ISO 8583. Other fields are adjusted according to the specifications we discussed in the Transaction Message Format section. Some products do not have field 57. Billing information is available in field 48.

## Prepaid Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| AC  | 12  | M     | ME    | ME    | ME    | Total admin fee                    |
| C1  | 24  | C     | CE    | CE    | CE    | Customer reference 1               |
| C2  | 24  | C     | CE    | CE    | CE    | Customer reference 2               |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description     |
|-----|-----|-------|-------|-------|-------|-----------------|
| S0  | 50  |       | M     |       |       | Inquiry screen  |
| S1  | 50  |       | M     |       |       | Inquiry screen  |
| S2  | 50  |       | M     |       |       | Inquiry screen  |
| S3  | 50  |       | M     |       |       | Inquiry screen  |
| S4  | 50  |       | M     |       |       | Inquiry screen  |
| S5  | 50  |       | M     |       |       | Inquiry screen  |
| S6  | 50  |       | M     |       |       | Inquiry screen  |
| S7  | 50  |       | M     |       |       | Inquiry screen  |
| S8  | 50  |       | M     |       |       | Inquiry screen  |
| S9  | 50  |       | M     |       |       | Inquiry screen  |
| R0  | 50  |       |       |       | M     | Payment receipt |
| R1  | 50  |       |       |       | M     | Payment receipt |
| R2  | 50  |       |       |       | M     | Payment receipt |
| R3  | 50  |       |       |       | M     | Payment receipt |
| R4  | 50  |       |       |       | M     | Payment receipt |
| R5  | 50  |       |       |       | M     | Payment receipt |
| R6  | 50  |       |       |       | M     | Payment receipt |
| R7  | 50  |       |       |       | M     | Payment receipt |
| R8  | 50  |       |       |       | M     | Payment receipt |
| R9  | 50  |       |       |       | M     | Payment receipt |
| RA  | 50  |       |       |       | M     | Payment receipt |
| RB  | 50  |       |       |       | M     | Payment receipt |
| RC  | 50  |       |       |       | M     | Payment receipt |
| RD  | 50  |       |       |       | M     | Payment receipt |
| RE  | 50  |       |       |       | M     | Payment receipt |
| RF  | 50  |       |       |       | M     | Payment receipt |
| RG  | 50  |       |       |       | M     | Payment receipt |
| RH  | 50  |       |       |       | M     | Payment receipt |
| RI  | 50  |       |       |       | M     | Payment receipt |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

**Field 48 Inquiry Request Example**
| TAG | VALUE        | 
| --- | ------------ | 
| AC  | 2500         | 
| PI  | 054501       | 
| CN  | 522030064594 | 

**Field 48 Inquiry Response Example**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000001       | 
| FS  | 000001             | 
| NM  | Customer 2 Bulan 4 | 

**Field 57 Inquiry Response Example**
| TAG | VALUE                            | 
| --- | -------------------------------- | 
| S0  | INFORMASI TAGIHAN LISTRIK        | 
| S1  |                                  | 
| S2  | BILLER : PLN PREPAID             | 
| S3  | IDPEL       : 522030064594       | 
| S4  | NAMA        : Customer 2 Bulan 4 | 
| S5  | BL/TH       : SEP20,OKT20        | 
| S6  | TOTAL TGHN  :  2 BULAN           | 
| S7  | RP TAG PLN  : RP         690.000 | 
| S8  | ADMIN BANK  : RP           2.500 | 
| S9  | TOTAL BAYAR : RP         692.500 | 

**Field 48 Payment Request Example**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

**Field 48 Payment Response Example**
| TAG | VALUE              | 
| --- | ------------------ | 
| AC  | 2500               | 
| PI  | 054501             | 
| CN  | 522030064594       | 
| FR  | 000000000016       | 
| FS  | 000016             | 
| NM  | Customer 2 Bulan 4 | 

**Field 57 Payment Response Example**
| TAG | VALUE                              | 
| --- | ---------------------------------- | 
| R0  | BILLER : PLN POSTPAID              | 
| R1  | IDPEL       : 522030064594         | 
| R2  | NAMA        : Customer 2 Bulan 4   | 
| R3  | BL/TH       : SEP20,OKT20          | 
| R4  | STAND METER : 100-330              | 
| R5  |                                    | 
| R6  | TOTAL TGHN  :  2 BULAN             | 
| R7  | TARIF/DAYA  : B1 /000001300 VA     | 
| R8  | NO REFF     : 0UAK73F968AFA20E3B24 | 
| R9  |                                    | 
| RA  | RP TAG PLN  : RP         300.000   | 
| RB  |                                    | 
| RC  | ADMIN BANK  : RP           2.500   | 
| RD  | TOTAL BAYAR : RP         302.500   | 
| RE  |                                    | 
| RF  |                                    | 
| RG  | PERUSAHAAN LISTRIK NEGARA(PLN)     | 
| RH  |  MENYATAKAN STRUK INI SEBAGAI      | 
| RI  |   BUKTI PEMBAYARAN YANG SAH        | 

## Postpaid Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| AC  | 12  | M     | ME    | ME    | ME    | Total admin fee                    |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description     |
|-----|-----|-------|-------|-------|-------|-----------------|
| S0  | 50  |       | M     |       |       | Inquiry screen  |
| S1  | 50  |       | M     |       |       | Inquiry screen  |
| S2  | 50  |       | M     |       |       | Inquiry screen  |
| S3  | 50  |       | M     |       |       | Inquiry screen  |
| S4  | 50  |       | M     |       |       | Inquiry screen  |
| S5  | 50  |       | M     |       |       | Inquiry screen  |
| S6  | 50  |       | M     |       |       | Inquiry screen  |
| S7  | 50  |       | M     |       |       | Inquiry screen  |
| S8  | 50  |       | M     |       |       | Inquiry screen  |
| S9  | 50  |       | M     |       |       | Inquiry screen  |
| R0  | 50  |       |       |       | M     | Payment receipt |
| R1  | 50  |       |       |       | M     | Payment receipt |
| R2  | 50  |       |       |       | M     | Payment receipt |
| R3  | 50  |       |       |       | M     | Payment receipt |
| R4  | 50  |       |       |       | M     | Payment receipt |
| R5  | 50  |       |       |       | M     | Payment receipt |
| R6  | 50  |       |       |       | M     | Payment receipt |
| R7  | 50  |       |       |       | M     | Payment receipt |
| R8  | 50  |       |       |       | M     | Payment receipt |
| R9  | 50  |       |       |       | M     | Payment receipt |
| RA  | 50  |       |       |       | M     | Payment receipt |
| RB  | 50  |       |       |       | M     | Payment receipt |
| RC  | 50  |       |       |       | M     | Payment receipt |
| RD  | 50  |       |       |       | M     | Payment receipt |
| RE  | 50  |       |       |       | M     | Payment receipt |
| RF  | 50  |       |       |       | M     | Payment receipt |
| RG  | 50  |       |       |       | M     | Payment receipt |
| RH  | 50  |       |       |       | M     | Payment receipt |
| RI  | 50  |       |       |       | M     | Payment receipt |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## Nontaglis Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| AC  | 12  | M     | ME    | ME    | ME    | Total admin fee                    |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description     |
|-----|-----|-------|-------|-------|-------|-----------------|
| S0  | 50  |       | M     |       |       | Inquiry screen  |
| S1  | 50  |       | M     |       |       | Inquiry screen  |
| S2  | 50  |       | M     |       |       | Inquiry screen  |
| S3  | 50  |       | M     |       |       | Inquiry screen  |
| S4  | 50  |       | M     |       |       | Inquiry screen  |
| S5  | 50  |       | M     |       |       | Inquiry screen  |
| S6  | 50  |       | M     |       |       | Inquiry screen  |
| S7  | 50  |       | M     |       |       | Inquiry screen  |
| S8  | 50  |       | M     |       |       | Inquiry screen  |
| S9  | 50  |       | M     |       |       | Inquiry screen  |
| R0  | 50  |       |       |       | M     | Payment receipt |
| R1  | 50  |       |       |       | M     | Payment receipt |
| R2  | 50  |       |       |       | M     | Payment receipt |
| R3  | 50  |       |       |       | M     | Payment receipt |
| R4  | 50  |       |       |       | M     | Payment receipt |
| R5  | 50  |       |       |       | M     | Payment receipt |
| R6  | 50  |       |       |       | M     | Payment receipt |
| R7  | 50  |       |       |       | M     | Payment receipt |
| R8  | 50  |       |       |       | M     | Payment receipt |
| R9  | 50  |       |       |       | M     | Payment receipt |
| RA  | 50  |       |       |       | M     | Payment receipt |
| RB  | 50  |       |       |       | M     | Payment receipt |
| RC  | 50  |       |       |       | M     | Payment receipt |
| RD  | 50  |       |       |       | M     | Payment receipt |
| RE  | 50  |       |       |       | M     | Payment receipt |
| RF  | 50  |       |       |       | M     | Payment receipt |
| RG  | 50  |       |       |       | M     | Payment receipt |
| RH  | 50  |       |       |       | M     | Payment receipt |
| RI  | 50  |       |       |       | M     | Payment receipt |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## Postpaid Airtime

**Field 48 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## Prepaid Airtime

**Field 48 Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   |       |       | M     | ME    | Product ID                         |
| CN  | 24  |       |       | M     | ME    | Contract number or customer number |
| AT  | 12  |       |       |       | M     | Total amount without point decimal |
| FR  | 24  |       |       |       | M     | Forwarding reference number        |
| FS  | 24  |       |       |       | M     | Forwarding STAN                    |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

**Field 48 Payment Request Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 

**Field 48 Payment Response Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 
| FR  | 8988757309534470    | 
| FS  | 000002              | 

## Mobile Data

**Field 48 Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   |       |       | M     | ME    | Product ID                         |
| CN  | 24  |       |       | M     | ME    | Contract number or customer number |
| AT  | 12  |       |       |       | M     | Total amount without point decimal |
| FR  | 24  |       |       |       | M     | Forwarding reference number        |
| FS  | 24  |       |       |       | M     | Forwarding STAN                    |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

**Field 48 Payment Request Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 

**Field 48 Payment Response Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 
| FR  | 8988757309534470    | 
| FS  | 000002              | 

## Top Up E-Money

**Field 48 Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   |       |       | M     | ME    | Product ID                         |
| CN  | 24  |       |       | M     | ME    | Contract number or customer number |
| AT  | 12  |       |       |       | M     | Total amount without point decimal |
| FR  | 24  |       |       |       | M     | Forwarding reference number        |
| FS  | 24  |       |       |       | M     | Forwarding STAN                    |

**Field 48 Payment Request Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 

**Field 48 Payment Response Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| PI  | 050001              | 
| CN  | 081277777777        | 
| FR  | 8988757309534470    | 
| FS  | 000002              | 

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## PBB (Pajak Bumi dan Bangunan)

**Field 48 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description                        |
|-----|-----|-------|-------|-------|-------|------------------------------------|
| PI  | 6   | M     | ME    | ME    | ME    | Product ID                         |
| CN  | 24  | M     | ME    | ME    | ME    | Contract number or customer number |
| NM  | 50  |       | M     | ME    | ME    | Customer name                      |
| AT  | 12  |       | M     | ME    | ME    | Total amount without point decimal |
| AC  | 12  | M     | ME    | ME    | ME    | Total admin fee                    |
| FR  | 24  |       | M     | ME    | ME    | Forwarding reference number        |
| FS  | 24  |       | M     | ME    | ME    | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 I | 210 I | 200 P | 210 P | Description     |
|-----|-----|-------|-------|-------|-------|-----------------|
| S0  | 50  |       | M     |       |       | Inquiry screen  |
| S1  | 50  |       | M     |       |       | Inquiry screen  |
| S2  | 50  |       | M     |       |       | Inquiry screen  |
| S3  | 50  |       | M     |       |       | Inquiry screen  |
| S4  | 50  |       | M     |       |       | Inquiry screen  |
| S5  | 50  |       | M     |       |       | Inquiry screen  |
| S6  | 50  |       | M     |       |       | Inquiry screen  |
| S7  | 50  |       | M     |       |       | Inquiry screen  |
| S8  | 50  |       | M     |       |       | Inquiry screen  |
| S9  | 50  |       | M     |       |       | Inquiry screen  |
| SA  | 50  |       | M     |       |       | Inquiry screen  |
| R0  | 50  |       |       |       | M     | Payment receipt |
| R1  | 50  |       |       |       | M     | Payment receipt |
| R2  | 50  |       |       |       | M     | Payment receipt |
| R3  | 50  |       |       |       | M     | Payment receipt |
| R4  | 50  |       |       |       | M     | Payment receipt |
| R5  | 50  |       |       |       | M     | Payment receipt |
| R6  | 50  |       |       |       | M     | Payment receipt |
| R7  | 50  |       |       |       | M     | Payment receipt |
| R8  | 50  |       |       |       | M     | Payment receipt |
| R9  | 50  |       |       |       | M     | Payment receipt |

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

**Field 48 Inquiry Request Example**
| TAG | VALUE            | 
| --- | ---------------- | 
| AC  | 2500             | 
| PI  | 050001           | 
| CN  | 0110017600183432 | 

**Field 48 Inquiry Response Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| AC  | 2500                | 
| PI  | 050001              | 
| CN  | 0110017600183432    | 
| FR  | 8988757309534470    | 
| FS  | 000001              | 
| NM  | WISHNU EKA SIDHARTA | 

**Field 57 Inquiry Response Example**
| TAG | VALUE                                 | 
| --- | ------------------------------------- | 
| S0  | NAME            : WISHNU EKA SIDHARTA | 
| S1  | CUSTOMER ID     : 0110017600183432    | 
| S2  | OPERATOR        : PBB Jakarta         | 
| S3  | SUBDISTRICT     : Poncokusumo         | 
| S4  | VILLAGE         : Gondokusuman        | 
| S5  | PAYMENT PERIOD  :                     | 
| S6  | SURFACE AREA    : 1519 m2             | 
| S7  | BUILDING AREA   : 119 m2              | 
| S8  | BILL AMOUNT     : 190000              | 
| S9  | DUE DATE        : 1231-01-01          | 
| SA  | NJOP BUMI/ M2   : 10000000            | 


**Field 48 Payment Request Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| AC  | 2500                | 
| PI  | 050001              | 
| CN  | 0110017600183432    | 
| FR  | 8988757309534470    | 
| FS  | 000002              | 
| NM  | WISHNU EKA SIDHARTA | 

**Field 48 Payment Response Example**
| TAG | VALUE               | 
| --- | ------------------- | 
| AC  | 2500                | 
| PI  | 050001              | 
| CN  | 0110017600183432    | 
| FR  | 8988757309534470    | 
| FS  | 000002              | 
| NM  | WISHNU EKA SIDHARTA | 

**Field 57 Payment Response Example**
| TAG | VALUE                                 | 
| --- | ------------------------------------- | 
| R0  | NAME            : WISHNU EKA SIDHARTA | 
| R1  | CUSTOMER ID     : 0110017600183432    | 
| R2  | OPERATOR        : PBB Jakarta         | 
| R3  | SUBDISTRICT     : Poncokusumo         | 
| R4  | VILLAGE         : Gondokusuman        | 
| R5  | PAYMENT PERIOD  : 2018                | 
| R6  | SURFACE AREA    : 1519 m2             | 
| R7  | BUILDING AREA   : 119 m2              | 
| R8  | BILL AMOUNT     : 190000              | 
| R9  | NJOP BUMI/ M2   : 10000000            | 

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
| ---- | ----------------------------------------------|
| 00   | Success                                       |
| 01   | Need To Sign On                               |
| 02   | Internal To Provider Down                     |
| 03   | Request Time Out                              |
| 05   | Do Not Honor                                  |
| 07   | Invalid Mandatory Field                       |
| 08   | Invalid Message Type Indicator                |
| 09   | Incomplete Message                            |
| 10   | Preset Maximum Transaction Loaded Exceeded    |
| 12   | Failed To Reverse Transaction Committed 1     |
| 13   | Invalid Amount                                |
| 14   | Invalid Card Number                           |
| 21   | Advice Failed                                 |
| 24   | Invalid Processing Code                       |
| 25   | Invalid Merchant Type                         |
| 26   | Invalid Institution Identification Code       |
| 30   | Invalid Product Code                          |
| 31   | Invalid Customer Id 1                         |
| 32   | Invalid Network Management Information Code   |
| 41   | Lost Card                                     |
| 43   | Invalid Card                                  |
| 48   | No Payment Record 1                           |
| 54   | Bill Already Paid                             |
| 56   | No Card Record                                |
| 58   | Transaction Not Permit                        |
| 59   | Invalid Transaction Amount                    |
| 60   | Invalid Minimum Amount 1                      |
| 62   | Invalid District Code                         |
| 65   | Exceeded Transaction Frequency Limit          |
| 68   | Response Receive Too Late                     |
| 69   | Unable Decrypt Track 2                        |
| 70   | MLPO Reference Number Already Exist           |
| 71   | Bill Amount Le Zero                           |
| 72   | Over quota                                    |
| 74   | Customer Id Suspended                         |
| 75   | Pin Tries Exceeded                            |
| 76   | Insufficient Fund 2                           |
| 81   | Cut Off                                       |
| 85   | Card Is Inactive                              |
| 86   | Card Is Blocked                               |
| 88   | Bill Not Available 1                          |
| 89   | Link To Service Provider Down                 |
| 90   | Cut Off In Progress                           |
| 91   | Payment Without Success Inquiry               |
| 92   | Transaction Not Found                         |
| 94   | Duplicated Transmission                       |
| 96   | System Malfunction                            |
| 97   | Invalid ARQC                                  |
| 98   | Internal Provider System Error                |
| 99   | Internal Ba Server Error                      |
