# Monitoring Juniper RPM tests with Zabbix
We recently leased space in some DC, and our ISP helped us in setting up an L2VPN between sites. We configured iBGP between them but encountered an issue with session flapping. We used the Juniper RPM tool (like Cisco SLA) to monitor it and gather data for ISP NOC requests. However, monitoring results by Zabbix for all our BGP peers presented a challenge: SNMP parsing of RPM objects in JunOS is not straightforward due to the complex MIB structure, which native Zabbix functionalities cannot easily handle. 

According to JUNIPER-RPM-MIB, there’s no direct OID to retrieve RPM identifiers via snmpwalk. All OIDs can be split for:
- MIB item, e.g., `1.3.6.1.4.1.2636.3.50.1.1.1.2`
- RPM identifier: `8.84.101.115.116.95.82.80.77.11.49.57.50.46.49.54.56.46.48.46.49`
- Test parameters, e.g., `.1` for RTT, `.2` for Round-trip jitter

Further research revealed that the RPM identifier is an encoded name `8.84.101.115.116.95.82.80.77.11.49.57.50.46.49.54.56.46.48.46.49`:
- The first number, `8`, indicates the number of characters in the first word.
- The following 8 numbers are the ASCII codes of the characters in the first word: `84.101.115.116.95.82.80.77`.
- The following number, `11`, represents the number of characters in the second word.
- The last 11 numbers are the ASCII codes of the characters in the second word: `49.57.50.46.49.54.56.46.48.46.49`.

By following this principle, decoding the entire UUID can yield the desired information: `Test_RPM` and `192.168.0.1`. For Zabbix discovery setup, you can easily use on-the-fly JS post-processing during an SNMP WALK on `1.3.6.1.4.1.2636.3.50.1.1.1.2` to decode RPM names and owners with this function: 

```js
function (value) {
  // Parse OID RPM code
  var regex = /\.1\.3\.6\.1\.4\.1\.2636\.3\.50\.1\.1\.1\.2\.(.*?)\.\d+ = /;
  var uniqueOidParts = {};
  var lines = value.split('\n');
  for (var i = 0; i < lines.length; i++) {
    var match = lines[i].match(regex);
    if (match) {
      uniqueOidParts[match[1]] = true;
    }
  }

  // Decode data from OID and prepare output data
  var result = [];
  for (var part in uniqueOidParts) {
    var parts = part.split('.');
    var decodedParts = [];
    var index = 0;

    while (index < parts.length) {
      var count = parseInt(parts[index], 10);
      var word = ''; 

      for (var j = 0; j < count; j++) {
        index++;
        word += String.fromCharCode(parseInt(parts[index], 10));
      }

      decodedParts.push(word);
      index++;
    }

    if (decodedParts.length >= 2) {
      result.push({
        '{#RPM_OWNER}': decodedParts[0],
        '{#RPM_NAME}': decodedParts[1],
        '{#RPM_UUID}': part
      });
    }
  }

  return JSON.stringify(result);
}
```

As result you gonna have list of dictionaries containing test details from the device:

```json
[
  {
    "{#RPM_OWNER}": "ownerName",
    "{#RPM_NAME}": "testName",
    "{#RPM_UUID}": "SNMPOidCode"
  },
  {
    "{#RPM_OWNER}": "Test_RPM",
    "{#RPM_NAME}": "192.168.0.1",
    "{#RPM_UUID}": "8.84.101.115.116.95.82.80.77.11.49.57.50.46.49.54.56.46.48.46.49"
  }
]
```

With this data, you can create a discovery rule and item prototypes. I am attaching my customized Juniper template for Zabbix, which includes the discovery setup, monitoring of BGP peers, and their management.
