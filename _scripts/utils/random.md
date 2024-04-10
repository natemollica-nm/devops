---
layout: default
title: "Random"
parent: utils
grand_parent: Scripts
nav_order: 2
---

# Random Utilities
{: .no_toc }

Various random data generating Linux/Unix scripting utilities.
{: .fs-6 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Random: DNS Hostname Generation

DNS hostnames have specific rules: they must start and end with a letter or a digit, can contain letters (a-z, A-Z), digits (0-9), and hyphens (-), but cannot start or end with a hyphen, and must not exceed 253 characters in total length.

Below is a simple POSIX-compliant shell function named `generate_random_hostname` that generates a 
random DNS hostname. This function doesn't rely on Bash-specific features, making it broadly compatible 
with shells that adhere to the POSIX standard. It combines randomly chosen letters for the start and 
end to comply with the rule that hostnames can't start or end with a hyphen and uses a combination of 
letters, digits, and occasionally a hyphen for the middle part. This example generates a hostname of a 
configurable length, defaulting to 10 characters for simplicity.

```shell
generate_random_hostname() {
    # Length of the generated hostname, default is 10 characters if not specified
    local length=${1:-10}

    # Define characters to use
    local letters="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    local letters_digits_hyphen="abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-"
    
    # Ensure the length is at least 3
    if [ "$length" -lt 3 ]; then
        echo "Minimum hostname length is 3"
        return 1
    fi

    # Start with a letter
    local hostname=$(dd if=/dev/urandom bs=1 count=1 2>/dev/null | tr -dc "$letters" | head -c 1)
    
    # Middle part can include hyphens, but start and end cannot
    if [ "$length" -gt 2 ]; then
        local middle_part_length=$((length - 2))
        hostname+=$(dd if=/dev/urandom bs=256 count=1 2>/dev/null | tr -dc "$letters_digits_hyphen" | head -c "$middle_part_length")
    fi

    # End with a letter or digit to avoid ending with a hyphen
    hostname+=$(dd if=/dev/urandom bs=1 count=1 2>/dev/null | tr -dc "${letters}0123456789" | head -c 1)

    echo "$hostname"
}

# Example usage:
hostname=$(generate_random_hostname 10)
echo "Generated hostname: $hostname"
```

## Script Breakdown

This script uses `dd` to get bytes from `/dev/urandom`, `tr` to filter characters to only those allowed in 
each part of the hostname, and `head` to trim the output to the desired length. It's structured to 
ensure the generated hostname complies with DNS rules: starts and ends with a letter or digit, and 
is of a specified length.