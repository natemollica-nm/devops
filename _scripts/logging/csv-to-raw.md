---
layout: default
title: "Utility: csv-to-raw"
parent: Logging
grand_parent: Scripts
nav_order: 2
---

# Script Utility: Log file `csv` to `raw` converter.
{: .no_toc }

Unix/Linux script to convert a `csv` formatted log file to `raw` format.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---


Converting a CSV log file to raw log entries can mean different things based on the structure of 
your CSV and the desired format of the raw logs. However, a common task might involve extracting 
data from a CSV file and formatting it into a specific log format. Here's a basic example to 
demonstrate how you could do this using a Bash script.

Let's say you have a CSV log file named log.csv with the following structure:

{% highlight txt %}
Date,Host,Service,@thread,@@module,@@level,Message
"2024-04-09T13:57:28.518Z","""ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app""","""consul-dataplane""","15","""envoy.dns""","""debug""","c-ares library cleaned up."
"2024-04-09T13:57:28.518Z","""ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app""","""consul-dataplane""","15","""envoy.init""","""debug""","init manager Server destroyed"
"2024-04-09T13:57:28.518Z","""ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app""","""consul-dataplane""","15","""envoy.main""","""debug""","destroying dispatcher main_thread"
{% endhighlight %}

You want to convert this into raw log entries that look like this:

{% highlight txt %}
2024-04-09T13:57:28.518Z "ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app":"consul-dataplane" ["DEBUG"] "envoy.dns" : c-ares library cleaned up."
2024-04-09T13:57:28.518Z "ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app":"consul-dataplane" ["DEBUG"] "envoy.init" : init manager Server destroyed"
2024-04-09T13:57:28.518Z "ip-20-130-20-188.us-east-2.compute.internal-us-140-prod-app":"consul-dataplane" ["DEBUG"] "envoy.main" : destroying dispatcher main_thread"
{% endhighlight %}

## `csv-to-raw.sh`

You can use the following script to accomplish this via Bash:

```bash
#!/bin/bash

# Path to your CSV file
csv_file="$1"

if [ -z "$csv_file" ]; then
  echo "Usage: $(basename "$0") <csv_file>"
fi

# Check if the file exists
if [[ ! -f "$csv_file" ]]; then
    echo "File $csv_file does not exist."
    exit 1
fi

# Read the CSV file line by line, skipping the header
awk -F, 'NR > 1 { # Skip the header line
    # Clean up fields: remove leading/trailing quotes and extra quotes inside fields
    for(i = 1; i <= NF; i++) {
        gsub(/^"|"$/, "", $i);
        gsub(/""/, "\"", $i);
    }

    # Format level to uppercase
    level = toupper($6);

    # Concatenate message parts if the message was split due to internal commas
    message = $7;
    for(i = 8; i <= NF; i++) {
        message = message ", " $i;
    }

    # Print the reformatted log entry
    # Assuming fields are: Date, Host, Service, Thread, Module, Level, Message
    print $1 " " $2":"$3 " [" level "] " $5 " : " message
}' "$csv_file"
```

### Script Breakdown
* It first checks if the specified CSV file exists.
* It iterates through all fields (`$i`) of each record, removes leading and trailing quotes, and replaces doubled quotes with single quotes.
* It extracts and reformats the log level to uppercase to match the desired output (`[DEBUG]`, `[TRACE]`, etc.).
* It concatenates all parts of the message that might have been split due to commas inside quoted fields.
* It prints each log entry in the desired format.
* The level (`$6`) is converted to uppercase, following your example. 
* The message is constructed by concatenating the base message field (`$7`) with any additional 
  fields that might contain parts of the message split due to internal commas. This step ensures 
  that messages containing commas are handled correctly. 
* The output format is constructed using the date (`$1`), host (`$2`), service name (`$3`), level, 
  thread name (`$5`), and the reconstructed message.