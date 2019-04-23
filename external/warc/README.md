#  Resources for generating WARC files with StormCrawler

First, you need to add the WARC module to the dependencies of your project.

```
		<dependency>
			<groupId>com.digitalpebble.stormcrawler</groupId>
			<artifactId>storm-crawler-warc</artifactId>
			<version>${storm-crawler.version}</version>
		</dependency>
```

Include the following snippet in your crawl topology

```java 
        String warcFilePath = "/warc";

        FileNameFormat fileNameFormat = new WARCFileNameFormat()
                .withPath(warcFilePath);

        Map<String,String> fields = new HashMap<>();
        fields.put("software:", "StormCrawler 1.14 http://stormcrawler.net/");
        fields.put("format", "WARC File Format 1.0");
        fields.put("conformsTo:",
                "https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.0/");
        
        WARCHdfsBolt warcbolt = (WARCHdfsBolt) new WARCHdfsBolt()
                .withFileNameFormat(fileNameFormat);
        warcbolt.withHeader(fields);

        // enable WARC request records
        warcbolt.withRequestRecords();

        // can specify the filesystem - will use the local FS by default
        String fsURL = "hdfs://localhost:9000";
        warcbolt.withFsUrl(fsURL);
        
        // a custom max length can be specified - 1 GB will be used as a default
        FileSizeRotationPolicy rotpol = new FileSizeRotationPolicy(50.0f,
                Units.MB);
        warcbolt.withRotationPolicy(rotpol);

        builder.setBolt("warc", warcbolt).localOrShuffleGrouping("fetch");
```

If you use Flux, you could add it like so:

```
components:
  - id: "WARCFileNameFormat"
    className: "com.digitalpebble.stormcrawler.warc.WARCFileNameFormat"
    configMethods:
      - name: "withPath"
        args:
          - "/path/to/warc"
      - name: "withPrefix"
        args:
          - "my-warc-prefix"

  - id: "WARCFileRotationPolicy"
    className: "org.apache.storm.hdfs.bolt.rotation.FileSizeRotationPolicy"
    constructorArgs:
      - 50.0
      - MB

  - id: "WARCInfo"
    className: "java.util.LinkedHashMap"
    configMethods:
      - name: "put"
        args:
         - "software"
         - "StormCrawler 1.14 http://stormcrawler.net/"
      - name: "put"
        args:
         - "format"
         - "WARC File Format 1.0"
      - name: "put"
        args:
         - "conformsTo"
         - "https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.0/"

[...]

bolts:
 - id: "warc"
    className: "com.digitalpebble.stormcrawler.warc.WARCHdfsBolt"
    parallelism: 1
    configMethods:
      - name: "withFileNameFormat"
        args:
          - ref: "WARCFileNameFormat"
      - name: "withRotationPolicy"
        args:
          - ref: "WARCFileRotationPolicy"
      - name: "withRequestRecords"
      - name: "withHeader"
        args:
          - ref: "WARCInfo"
```

Each instance of the bolt will generate a WARC file and close it once it has reached the required size.

Please note that the WARCHdfsBolt is a dead-end and does not output tuples to subsequent bolts.

The tuples are acked based on the sync policy, which is based on either of:
* an explicit sync as set in the sync policy which we have by default at 10 tuples
* an automatic one which happens via tick tuples every 15 secs by default

With the local file system, you need to specify 

```
  warcbolt.withConfigKey("warc");
  Map<String, Object> hdfsConf = new HashMap<>();
  hdfsConf.put("fs.file.impl", "org.apache.hadoop.fs.RawLocalFileSystem");
  getConf().put("warc", hdfsConf);
```

This uses the RawLocalFileSystem, which unlike the checksum one used by default does a proper sync of the content to the file.

Using Flux the RawLocalFileSystem is enabled by adding the following statements to `config` and the `warc` bolt:

```
config:
  warc: {"fs.file.impl": "org.apache.hadoop.fs.RawLocalFileSystem"}

bolts:
 - id: "warc"
    ...
    configMethods:
      ...
      - name: "withConfigKey"
        args:
          - "warc"
```

Writing complete and valid WARC requires that HTTP headers, IP address and capture time are stored by the HTTP protocol implementation in the metadata. Only the okhttp protocol package stores all necessary information. To enable the okhttp protocol implementation and let them save the HTTP headers, you need to add to your configuration:

```
  http.store.headers: true

  http.protocol.implementation: com.digitalpebble.stormcrawler.protocol.okhttp.HttpProtocol
  https.protocol.implementation: com.digitalpebble.stormcrawler.protocol.okhttp.HttpProtocol
```
