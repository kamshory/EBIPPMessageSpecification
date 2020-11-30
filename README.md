


# Topology

The following is a topological overview of the AltoPay Biller.

```mermaid
sequenceDiagram
Collection Agent ->> AltoPay Biller: Payment Request
AltoPay Biller->>Biller: Payment Request
Biller->> AltoPay Biller: Payment Response
AltoPay Biller->> Collection Agent: Payment Response
Note right of AltoPay Biller: If Biller has not <br>responded to a <br>request for a long <br>time, AltoPay Biller <br>will send an Advice
AltoPay Biller->>Biller: Advice Request
Biller->> AltoPay Biller: Advice Response
AltoPay Biller->>Collection Agent : Payment Response
Note right of Collection Agent: If AltoPay Biller has <br>not responded to a <br>request for a long <br>time, Collection <br>Agent will send an <br>Advice
Collection Agent ->> AltoPay Biller: Advice Request
AltoPay Biller->> Collection Agent: Advice Response

```
![AltoPay Biller Topology](https://raw.githubusercontent.com/kamshory/EBIPPMessageSpecification/b4aec0e43ae6a7f8c23dc8f2902b28b9a9c37534/ap-biller-topology.svg)

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

# ISO 8385 Data Element

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

We can parse that data become:

```
CURRENT OFFSET = 0
REPEAT
	READ 2 BYTE FROM CURRENT OFFSET
	MAKE IS AS TAG
	ADD CURRENT OFFSET WITH 2
	READ 2 BYTE FROM CURRENT OFFSET
	MAKE IT AS DECIMAL NUMBER REPRESENT DATA LENGTH
	READ DATA ACCORDING TO ITS LENGTH
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

Field 48 on Network Management Request sent by client contains
| Tag | Value                        | 0800 | 0810 |
| --- | ---------------------------- | ---- | ---- |
| AK  | API Key                      | M    | ME   |
| TS  | Timestamp in ISO Format      | M    | ME   |
| SN  | Signature                    | M    | ME   |
| KM  | Key Management               | -    | M    |
| EI  | Expires In (Second)          | -    | M    |
| EA  | Expires At (Unix Timestamp)  | -    | M    |

Credentials information required:
1. Client code (sent to server via field  32)
2. API Key (sent to server via field  48)
3. Validation Key (not sent to server, required to create signature)

Field 48 not required on echo test.

Field 70 is Network Management Information Code

Data element for network management  request

| Code | Description  | 7  | 11 | 32 | 39 | 48 | 70 | 
| ---- | ------------ | -- | -- | -- | -- | -- | -- |
| 001  | Logon        | M  | M  | M  | -  | M  | M  |
| 002  | Logoff       | M  | M  | M  | -  | M  | M  |
| 161  | Key Exchange | M  | M  | M  | -  | M  | M  |
| 162  | New Key      | M  | M  | M  | -  | M  | M  |
| 201  | Cutover      | M  | M  | M  | -  | M  | M  |
| 301  | Echo Test    | M  | M  | M  | -  | -  | M  |
 
Data element for network management  response

| Code | Description  | 7  | 11 | 32 | 39 | 48 | 70 | 
| ---- | ------------ | -- | -- | -- | -- | -- | -- |
| 001  | Logon        | M  | M  | M  | M  | M  | M  |
| 002  | Logoff       | M  | M  | M  | M  | M  | M  |
| 161  | Key Exchange | M  | M  | M  | M  | M  | M  |
| 162  | New Key      | M  | M  | M  | M  | M  | M  |
| 201  | Cutover      | M  | M  | M  | M  | M  | M  |
| 301  | Echo Test    | M  | M  | M  | M  | -  | M  |
 

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
"f48": "KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z",
"f70": "001"
}
```

ISO Message = "081082200001020100000400000000000000112707241600000405CA12300266KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z001"

### Logoff
The client must send `LOGOFF` request when they will not send transaction again.

Direction : from client to server

**Logoff Request**
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

**Logoff Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z",
"f70": "002"
}
```

ISO Message = "081082200001020100000400000000000000112707241600000405CA12300266KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z002"

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
"f48": "KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z",
"f70": "161"
}
```

ISO Message = "081082200001020100000400000000000000112707241600000405CA12300266KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z161"

### New Key

Server will send new key periodically. Client must use newest key on next transaction. Client must response it with response code **SUCCESS (00)** to make sure that client receive the key.

Direction: from server to client


**New Key Request**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f48":"AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285",
"f70":"161"
}
```

ISO Message = "080082200001000100000400000000000000112707241600000405CA123116AK16akey_7HgyugUyfuTTS242020-11-29T07:55:37.653ZSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285162"

**New Key Response**
```json
{
"f7": "1127072416",
"f11": "000004",
"f32": "CA123",
"f39": "00",
"f48": "KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z",
"f70": "162"
}
```

ISO Message = "081082200001020100000400000000000000112707241600000405CA12300266KM123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NzEwNDg2fQ.oCPVl9htbj5W8pBWkyXj0YkhUJ84WpsLN8ilgVnCBw0EI043610AK16akey_7HgyugUyfuTSN64871f7de75762871769c7ec7b0b3f62d154c6c8f244914c076d25cb60972a9285EA101606710486TS242020-11-29T07:55:37.653Z162"

# Transaction Message Format

## Data Elements

The following is a list of data elements

| DE   | Type and Length    | Description                                  |
| ---- | ------------------ | -------------------------------------------- |
| 3    | N 6                | Processing Code                              |
| 4    | N 12               | Amount                                       |
| 7    | N 10               | Date and Time in GMT                         |
| 11   | N 6                | System Trace Audit Number                    |
| 12   | N 6                | Local Transaction Time                       |
| 13   | N 4                | Local Transaction Date                       |
| 15   | N 4                | Settlement Date                              |
| 18   | N 4                | Channel Type                                 |
| 32   | AN .. 11 LLVAR     | Acquiring Institution ID (Client Code)       |
| 37   | AN 12              | Transaction Reference Number                 |
| 39   | N 2                | Response Code                                |
| 41   | AN 16              | Card Acceptor Terminal Identification        |
| 48   | ANS ... 999 LLLVAR | Additional Data                              |
| 49   | N 3                | Transaction Currency Code                    |
| 57   | ANS ... 999 LLLVAR | Inquiry Screen and Payment Receipt           |
| 100  | AN .. 11 LLVAR     | Receiving Institution Identification Code    |
| 120  | AN ... 999 LLLVAR  | Last Key Sent by Server                      |
| 125  | AN .. 255 LLLVAR   | Transaction Indicator                        |
| 127  | AN .. 11 LLVAR     | Destination Institution Identification Code  |

Note:

1. Field 32, 100, 125 and 127 will be informed
2. Field 48 and 57 are TLV
3. Field 39 only exists on response

### Field 32 

Field 32 is filled by `Client Code`. Client code is generated by AltoPay Biller and will be informed to client.

### Field 48 
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description                        |
|-----|-----|---------|---------|---------|---------|------------------------------------|
| PI  | 6   | M       | ME      | ME      | ME      | Product ID                         |
| CN  | 24  | M       | ME      | ME      | ME      | Contract number or customer number |
| NM  | 50  |         | M       | ME      | ME      | Customer name                      |
| AT  | 12  |         | M       | ME      | ME      | Total amount without point decimal |
| AC  | 12  | C       | CE      | CE      | CE      | Total admin fee                    |
| C1  | 24  | C       | CE      | CE      | CE      | Customer reference 1               |
| C2  | 24  | C       | CE      | CE      | CE      | Customer reference 2               |
| FR  | 24  |         | M       | ME      | ME      | Forwarding reference number        |
| FS  | 24  |         | M       | ME      | ME      | Forwarding STAN                    |

### Field 57 
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description     |
|-----|-----|---------|---------|---------|---------|-----------------|
| S0  | 50  |         | M       |         |         | Inquiry screen  |
| S1  | 50  |         | M       |         |         | Inquiry screen  |
| S2  | 50  |         | M       |         |         | Inquiry screen  |
| S3  | 50  |         | M       |         |         | Inquiry screen  |
| S4  | 50  |         | M       |         |         | Inquiry screen  |
| S5  | 50  |         | M       |         |         | Inquiry screen  |
| S6  | 50  |         | M       |         |         | Inquiry screen  |
| S7  | 50  |         | M       |         |         | Inquiry screen  |
| S8  | 50  |         | M       |         |         | Inquiry screen  |
| S9  | 50  |         | M       |         |         | Inquiry screen  |
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

### Field 100

Field 32 is filled by `Receiving Institution Code`. Receiving institution code will be informed to client.

### Field 125

Field 125 us transaction indicator. Transaction indicator format will be informed to client.

### Field 127

Field 32 is filled by `Destination Code`. Destination code will be informed to client.

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
"f32": "360003",
"f37": "000000000248",
"f41": "KOI ",
"f48": "PI06013026CN12081228812348AC042500",
"f49": "360",
"f100": "1234567",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f125": "",
"f127": "071234567"
}
```

ISO message: "0200B23A400108818000000000001000010A370000000000000000112707241600000414241311281128601206360003000000000248KOI             034PI06013026CN12081228812348AC042500360071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

### Inquiry Response

```json
{
"f125": "",
"f41": "KOI ",
"f32": "360003",
"f100": "1234567",
"f12": "142413",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f11": "000004",
"f13": "1128",
"f57": "S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN : 081228812348S333NAMA : KAMSHORY ROYS431TAGIHAN : Rp 138.983S534PERIODE : November 2020S631JATUH TEMPO : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
"f18": "6012",
"f3": "370000",
"f39": "00",
"f4": "000000138983",
"f7": "1127072416",
"f127": "071234567"
}
```

ISO message: "0210B23A40010A818080000000001000010A37000000000013898311270724160000041424131128112860120636000300000000024800KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360296S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN    : 081228812348S333NAMA               : KAMSHORY ROYS431TAGIHAN            : Rp 138.983S534PERIODE            : November 2020S631JATUH TEMPO        : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

### Payment Request

```json
{
"f125": "",
"f41": "KOI ",
"f32": "360003",
"f100": "1234567",
"f12": "142413",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f11": "000004",
"f13": "1128",
"f57": "",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
"f18": "6012",
"f3": "370000",
"f4": "000000138983",
"f7": "1127072416",
"f127": "071234567"
}
```

ISO message: "0200B23A400108818080000000001000010A370000000000138983112707241600000414241311281128601206360003000000000248KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360000071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

### Payment Response

```json
{
"f125": "",
"f41": "KOI ",
"f32": "360003",
"f100": "1234567",
"f12": "142413",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f11": "000004",
"f13": "1128",
"f57": "S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN : 081228812348S333NAMA : KAMSHORY ROYS431TAGIHAN : Rp 138.983S534PERIODE : November 2020S631JATUH TEMPO : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
"f18": "6012",
"f3": "370000",
"f39": "00",
"f4": "000000138983",
"f7": "1127072416",
"f127": "071234567"
}
```

ISO message: "0210B23A40010A818080000000001000010A37000000000013898311270724160000041424131128112860120636000300000000024800KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360296S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN    : 081228812348S333NAMA               : KAMSHORY ROYS431TAGIHAN            : Rp 138.983S534PERIODE            : November 2020S631JATUH TEMPO        : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

### Advice Request

```json
{
"f125": "",
"f41": "KOI ",
"f32": "360003",
"f100": "1234567",
"f12": "142413",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f11": "000004",
"f13": "1128",
"f57": "",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
"f18": "6012",
"f3": "370000",
"f4": "000000138983",
"f7": "1127072416",
"f127": "071234567"
}
```

ISO message: "0220B23A400108818080000000001000010A370000000000138983112707241600000414241311281128601206360003000000000248KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360000071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

### Advice Response

```json
{
"f125": "",
"f41": "KOI ",
"f32": "360003",
"f100": "1234567",
"f12": "142413",
"f120": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD8",
"f11": "000004",
"f13": "1128",
"f57": "S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN : 081228812348S333NAMA : KAMSHORY ROYS431TAGIHAN : Rp 138.983S534PERIODE : November 2020S631JATUH TEMPO : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG",
"f49": "360",
"f15": "1128",
"f37": "000000000248",
"f48": "AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY",
"f18": "6012",
"f3": "370000",
"f39": "00",
"f4": "000000138983",
"f7": "1127072416",
"f127": "071234567"
}
```

ISO message: "0230B23A40010A818080000000001000010A37000000000013898311270724160000041424131128112860120636000300000000024800KOI             077AC042500PI06013026CN12081228812348FR1466479217091550FS0567279NM12KAMSHORY ROY360296S018TAGIHAN KARTU HALOS140========================================S233NOMOR PELANGGAN    : 081228812348S333NAMA               : KAMSHORY ROYS431TAGIHAN            : Rp 138.983S534PERIODE            : November 2020S631JATUH TEMPO        : 2020-12-01S700S840SIMPAN STRUK INI YAH, JANGAN SAMPE ILANG071234567123eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJBTFRPIiwiZXhwIjoxNjA2NjIyNjg1fQ.-QVFeBEbSB4woOrTw3qY8PIi5G67iUZdTrWyKU7tGD800009071234567"

# Product Message Specification

In this session, we will only discuss field  48 and field  57 of ISO 8583. Other field s are adjusted according to the specifications we discussed in the Transaction Message Format section.

## Prepaid Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description                        |
|-----|-----|---------|---------|---------|---------|------------------------------------|
| PI  | 6   | M       | ME      | ME      | ME      | Product ID                         |
| CN  | 24  | M       | ME      | ME      | ME      | Contract number or customer number |
| NM  | 50  |         | M       | ME      | ME      | Customer name                      |
| AT  | 12  |         | M       | ME      | ME      | Total amount without point decimal |
| AC  | 12  | M       | ME      | ME      | ME      | Total admin fee                    |
| C1  | 24  | C       | CE      | CE      | CE      | Customer reference 1               |
| C2  | 24  | C       | CE      | CE      | CE      | Customer reference 2               |
| FR  | 24  |         | M       | ME      | ME      | Forwarding reference number        |
| FS  | 24  |         | M       | ME      | ME      | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description     |
|-----|-----|---------|---------|---------|---------|-----------------|
| S0  | 50  |         | M       |         |         | Inquiry screen  |
| S1  | 50  |         | M       |         |         | Inquiry screen  |
| S2  | 50  |         | M       |         |         | Inquiry screen  |
| S3  | 50  |         | M       |         |         | Inquiry screen  |
| S4  | 50  |         | M       |         |         | Inquiry screen  |
| S5  | 50  |         | M       |         |         | Inquiry screen  |
| S6  | 50  |         | M       |         |         | Inquiry screen  |
| S7  | 50  |         | M       |         |         | Inquiry screen  |
| S8  | 50  |         | M       |         |         | Inquiry screen  |
| S9  | 50  |         | M       |         |         | Inquiry screen  |
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

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## Postpaid Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description                        |
|-----|-----|---------|---------|---------|---------|------------------------------------|
| PI  | 6   | M       | ME      | ME      | ME      | Product ID                         |
| CN  | 24  | M       | ME      | ME      | ME      | Contract number or customer number |
| NM  | 50  |         | M       | ME      | ME      | Customer name                      |
| AT  | 12  |         | M       | ME      | ME      | Total amount without point decimal |
| AC  | 12  | M       | ME      | ME      | ME      | Total admin fee                    |
| FR  | 24  |         | M       | ME      | ME      | Forwarding reference number        |
| FS  | 24  |         | M       | ME      | ME      | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description     |
|-----|-----|---------|---------|---------|---------|-----------------|
| S0  | 50  |         | M       |         |         | Inquiry screen  |
| S1  | 50  |         | M       |         |         | Inquiry screen  |
| S2  | 50  |         | M       |         |         | Inquiry screen  |
| S3  | 50  |         | M       |         |         | Inquiry screen  |
| S4  | 50  |         | M       |         |         | Inquiry screen  |
| S5  | 50  |         | M       |         |         | Inquiry screen  |
| S6  | 50  |         | M       |         |         | Inquiry screen  |
| S7  | 50  |         | M       |         |         | Inquiry screen  |
| S8  | 50  |         | M       |         |         | Inquiry screen  |
| S9  | 50  |         | M       |         |         | Inquiry screen  |
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

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

## Nontaglis Electricity

**Field 48 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description                        |
|-----|-----|---------|---------|---------|---------|------------------------------------|
| PI  | 6   | M       | ME      | ME      | ME      | Product ID                         |
| CN  | 24  | M       | ME      | ME      | ME      | Contract number or customer number |
| NM  | 50  |         | M       | ME      | ME      | Customer name                      |
| AT  | 12  |         | M       | ME      | ME      | Total amount without point decimal |
| AC  | 12  | M       | ME      | ME      | ME      | Total admin fee                    |
| FR  | 24  |         | M       | ME      | ME      | Forwarding reference number        |
| FS  | 24  |         | M       | ME      | ME      | Forwarding STAN                    |

**Field 57 Inquiry and Payment**
| Tag | Max | 200 INQ | 210 INQ | 200 PMT | 210 PMT | Description     |
|-----|-----|---------|---------|---------|---------|-----------------|
| S0  | 50  |         | M       |         |         | Inquiry screen  |
| S1  | 50  |         | M       |         |         | Inquiry screen  |
| S2  | 50  |         | M       |         |         | Inquiry screen  |
| S3  | 50  |         | M       |         |         | Inquiry screen  |
| S4  | 50  |         | M       |         |         | Inquiry screen  |
| S5  | 50  |         | M       |         |         | Inquiry screen  |
| S6  | 50  |         | M       |         |         | Inquiry screen  |
| S7  | 50  |         | M       |         |         | Inquiry screen  |
| S8  | 50  |         | M       |         |         | Inquiry screen  |
| S9  | 50  |         | M       |         |         | Inquiry screen  |
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

Note: Advice requests are the same as payment requests except MTI is 0220 instead of 0200

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
