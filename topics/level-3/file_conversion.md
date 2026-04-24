### Topic: File Conversion (EDI, XML, CSV to JSON) — Level 3

> **Related:** [Level 3 — IO (NIO, file read/write)](./io.md) · [Level 2 — SOAP Calls](../level-2/soap_calls.md)

**Why it matters (Karat angle)**
Banking systems exchange data in EDI, XML, CSV, and JSON. A senior Java dev at Citi must parse legacy formats and convert them for modern APIs. Interviewers ask this to verify you've worked with real enterprise data formats — not just JSON.

**Core concept**

| Format | Structure | Use at Citi | Java library |
|--------|----------|-------------|-------------|
| **EDI** | Positional/delimited segments | Payment files (SWIFT, ACH) | Smooks, BerryWorks |
| **XML** | Tree-structured, verbose | SOAP, legacy config, FpML | JAXB, Jackson XML, DOM/SAX |
| **CSV** | Comma/tab-delimited flat | Reports, bulk imports | OpenCSV, Apache Commons CSV |
| **JSON** | Key-value, nested | REST APIs, modern messaging | Jackson, Gson |

**CSV → JSON:**

```java
// File: topics/level-3/CsvToJsonDemo.java

// Using Jackson + OpenCSV
public class CsvToJsonConverter {

    public static String convert(Path csvPath) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        List<Map<String, String>> records = new ArrayList<>();

        try (CSVReader reader = new CSVReader(Files.newBufferedReader(csvPath))) {
            String[] headers = reader.readNext();         // first row = headers
            String[] row;
            while ((row = reader.readNext()) != null) {
                Map<String, String> record = new LinkedHashMap<>();
                for (int i = 0; i < headers.length; i++) {
                    record.put(headers[i].trim(), i < row.length ? row[i].trim() : "");
                }
                records.add(record);
            }
        }

        return mapper.writerWithDefaultPrettyPrinter().writeValueAsString(records);
    }

    // Alternative: Jackson CsvMapper (direct CSV → POJO → JSON)
    public static List<Transaction> csvToPojos(Path csvPath) throws Exception {
        CsvMapper csvMapper = new CsvMapper();
        CsvSchema schema = CsvSchema.emptySchema().withHeader();
        MappingIterator<Transaction> iter = csvMapper.readerFor(Transaction.class)
            .with(schema)
            .readValues(csvPath.toFile());
        return iter.readAll();
    }
}
```

**XML → JSON:**

```java
// File: topics/level-3/XmlToJsonDemo.java

// Using Jackson's XML + JSON modules
public class XmlToJsonConverter {

    // Approach 1: XML String → JsonNode → JSON String
    public static String convert(String xml) throws Exception {
        XmlMapper xmlMapper = new XmlMapper();
        JsonNode tree = xmlMapper.readTree(xml);           // parse XML to tree

        ObjectMapper jsonMapper = new ObjectMapper();
        return jsonMapper.writerWithDefaultPrettyPrinter().writeValueAsString(tree);
    }

    // Approach 2: XML → POJO → JSON (type-safe)
    public static String convertTyped(Path xmlPath) throws Exception {
        // JAXB to deserialize XML
        JAXBContext ctx = JAXBContext.newInstance(PaymentMessage.class);
        Unmarshaller unmarshaller = ctx.createUnmarshaller();
        PaymentMessage payment = (PaymentMessage) unmarshaller.unmarshal(xmlPath.toFile());

        // Jackson to serialize to JSON
        ObjectMapper mapper = new ObjectMapper();
        mapper.registerModule(new JavaTimeModule());
        return mapper.writeValueAsString(payment);
    }
}

@XmlRootElement(name = "Payment")
@XmlAccessorType(XmlAccessType.FIELD)
class PaymentMessage {
    @XmlElement private String accountId;
    @XmlElement private BigDecimal amount;
    @XmlElement private String currency;
    @XmlElement private LocalDate valueDate;
}
```

**EDI parsing (banking-specific):**

```java
// File: topics/level-3/EdiParseDemo.java

// EDI format example (simplified SWIFT MT103):
// :20:TXN20260423001
// :32A:260423USD1000000,
// :50K:/1234567890
// ALICE SMITH
// :59:/9876543210
// BOB JONES

public class EdiParser {

    public static Map<String, String> parseMT103(String ediContent) {
        Map<String, String> fields = new LinkedHashMap<>();
        String[] lines = ediContent.split("\\n");
        String currentTag = null;

        for (String line : lines) {
            if (line.startsWith(":")) {
                // Tag line: :TAG:VALUE
                int tagEnd = line.indexOf(':', 1);
                currentTag = line.substring(1, tagEnd);
                String value = line.substring(tagEnd + 1);
                fields.put(currentTag, value);
            } else if (currentTag != null) {
                // Continuation line
                fields.merge(currentTag, "\n" + line, String::concat);
            }
        }
        return fields;
    }

    // Convert to JSON
    public static String toJson(String ediContent) throws Exception {
        Map<String, String> fields = parseMT103(ediContent);
        return new ObjectMapper().writerWithDefaultPrettyPrinter().writeValueAsString(fields);
    }
}
```

**Streaming large files (Jackson streaming API):**

```java
// File: topics/level-3/StreamingJsonDemo.java

// For very large files — don't load into memory
try (JsonParser parser = new JsonFactory().createParser(Path.of("huge.json").toFile())) {
    while (parser.nextToken() != null) {
        if (parser.getCurrentToken() == JsonToken.FIELD_NAME
                && "accountId".equals(parser.getCurrentName())) {
            parser.nextToken();
            String accountId = parser.getValueAsString();
            // process one record at a time
        }
    }
}

// Streaming CSV write
try (CSVWriter writer = new CSVWriter(Files.newBufferedWriter(Path.of("output.csv")))) {
    writer.writeNext(new String[]{"AccountId", "Name", "Balance"});
    for (Account acc : accounts) {
        writer.writeNext(new String[]{
            acc.getId().toString(),
            acc.getName(),
            acc.getBalance().toString()
        });
    }
}
```

**What to say in the interview (4-beat answer)**
1. **Definition:** File conversion maps between data formats — CSV (flat, delimited), XML (tree, verbose), EDI (positional segments), JSON (key-value). Jackson handles JSON/XML/CSV; JAXB handles XML; custom parsers for EDI.
2. **Why/when:** Banking integrations receive EDI (SWIFT messages), XML (SOAP, FpML), and CSV (batch reports). Modern APIs expect JSON. Conversion happens in integration layers and ETL pipelines.
3. **Example:** `XmlMapper.readTree(xml)` parses XML to a tree; `ObjectMapper.writeValueAsString(tree)` outputs JSON. For type safety, parse to POJOs first (JAXB for XML, CsvMapper for CSV) then serialise to JSON.
4. **Gotcha/tradeoff:** Loading large files into memory (DOM parsing, `readAllLines`) causes OOM. Use streaming APIs (Jackson `JsonParser`, SAX for XML, buffered CSV reader) for files over ~100MB.

**Common pitfalls**
- DOM-parsing a 1GB XML file — OOM. Use SAX or StAX (streaming).
- CSV with embedded commas or newlines — use a library (OpenCSV) instead of `String.split(",")`.
- XML namespace issues — Jackson's `XmlMapper` may drop namespace prefixes; configure explicitly.
- EDI field positions vary by message type — hardcoded offsets break on different formats.

**Self-check question**
You receive a 500MB CSV file with 10 million rows. You need to convert it to JSON and push each record to a Kafka topic. Do you load the entire file into memory? Describe the streaming approach.
