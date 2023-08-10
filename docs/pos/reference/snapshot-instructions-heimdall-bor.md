---
id: snapshot-instructions-heimdall-bor
title: Heimdall and Bor Snapshots
sidebar_label: Snapshots
description: Heimdall and Bor snapshot Instructions for faster syncing.
keywords:
  - docs
  - matic
  - polygon
  - binary
  - node
  - validator
  - sentry
  - heimdall
  - bor
  - snapshots
image: https://wiki.polygon.technology/img/polygon-wiki.png
---

import useBaseUrl from '@docusaurus/useBaseUrl';

When setting up a new sentry, validator, or full node server, it is recommended that you use snapshots for faster syncing without having to sync over the network. Using snapshots will save you several days for both Heimdall and Bor. **Note: We no longer support bor archive snapshots due to unsustainable data growth.** 

:::tip

For the latest snapshot, please visit [<ins>Polygon Chains Snapshots</ins>](https://snapshot.polygon.technology/).

:::

## Client Snapshots

To begin, ensure that your node environment meets the **prerequisites** outlined [here](https://wiki.polygon.technology/docs/operate/full-node-binaries/). Before starting any services, execute the shell script provided below. This script will download and extract the snapshot data, which allows for faster bootstrapping. In our example, we will be using an Ubuntu Linux m5d.4xlarge machine with an 8TB block device attached.
To transfer the correct chaindata to your disk, follow these steps:

- All one has to do is specify the network ("mainnet" or "mumbai") and client type ("heimdall" or "bor" or "erigon") of your desired snapshot and run the following command:

```bash
curl -L https://snapshot-download.polygon.technology/snapdown.sh | bash -s -- --network {{ network }} --client {{ client }} --extract-dir {{ extract_dir }} --validate-checksum {{ true / false }}
```

For example:

```bash
curl -L https://snapshot-download.polygon.technology/snapdown.sh | bash -s -- --network mainnet --client heimdall --extract-dir data --validate-checksum true
```

> This bash script automatically handles all download and extraction phases, as well as optimizing disk space by deleting already extracted files along the way.

- `--extract-dir` and `--validate-checksum` flags are optional.
- Consider using a Screen session to prevent accidental interruptions during the chaindata download and extraction process.
- The raw bash script code is collapsed below for transparency:

<details> 
<summary>View script here ↓</summary>

```bash
  #!/bin/bash

  function validate_network() {
    if [[ "$1" != "mainnet" && "$1" != "mumbai" ]]; then
      echo "Invalid network input. Please enter 'mainnet' or 'mumbai'."
      exit 1
    fi
  }

  function validate_client() {
    if [[ "$1" != "heimdall" && "$1" != "bor" && "$1" != "erigon" ]]; then
      echo "Invalid client input. Please enter 'heimdall' or 'bor' or 'erigon'."
      exit 1
    fi
  }

  function validate_checksum() {
    if [[ "$1" != "true" && "$1" != "false" ]]; then
      echo "Invalid checksum input. Please enter 'true' or 'false'."
      exit 1
    fi
  }

  # Parse command-line arguments
  while [[ $# -gt 0 ]]; do
    key="$1"

    case $key in
      -n | --network)
        validate_network "$2"
        network="$2"
        shift # past argument
        shift # past value
        ;;
      -c | --client)
        validate_client "$2"
        client="$2"
        shift # past argument
        shift # past value
        ;;
      -d | --extract-dir)
        extract_dir="$2"
        shift # past argument
        shift # past value
        ;;
      -v | --validate-checksum)
        validate_checksum "$2"
        checksum="$2"
        shift # past argument
        shift # past value
        ;;
      *) # unknown option
        echo "Unknown option: $1"
        exit 1
        ;;
    esac
  done

  # Set default values if not provided through command-line arguments
  network=${network:-mumbai}
  client=${client:-heimdall}
  extract_dir=${extract_dir:-"${client}_extract"}
  checksum=${checksum:-false}


  # temporary as we transition erigon mainnet snapshots to new incremental model, ETA Aug 2023
  if [[ "$client" == "erigon" && "$network" == "mainnet" ]]; then
    echo "Erigon bor-mainnet archive snapshots currently unavailable as we transition to incremental snapshot model. ETA Aug 2023."
    exit 1
  fi

  # install dependencies and cursor to extract directory
  sudo apt-get update -y
  sudo apt-get install -y zstd pv aria2
  mkdir -p "$extract_dir"
  cd "$extract_dir"

  # download compiled incremental snapshot files list
  aria2c -x6 -s6 "https://snapshot-download.polygon.technology/$client-$network-parts.txt"

  # remove hash lines if user declines checksum verification
  if [ "$checksum" == "false" ]; then
      sed -i '/checksum/d' $client-$network-parts.txt
  fi

  # download all incremental files, includes automatic checksum verification per increment
  aria2c -x6 -s6 -c --auto-file-renaming=false --max-tries=100 -i $client-$network-parts.txt

  # Don't extract if download failed
  if [ $? -ne 0 ]; then
      echo "Download failed. Restart the script to resume downloading."
      exit 1
  fi

  declare -A processed_dates

  # Join bulk parts into valid tar.zst and extract
  for file in $(find . -name "$client-$network-snapshot-bulk-*-part-*" -print | sort); do
      date_stamp=$(echo "$file" | grep -o 'snapshot-.*-part' | sed 's/snapshot-\(.*\)-part/\1/')
      
      # Check if we have already processed this date
      if [[ -z "${processed_dates[$date_stamp]}" ]]; then
          processed_dates[$date_stamp]=1
          output_tar="$client-$network-snapshot-${date_stamp}.tar.zst"
          echo "Join parts for ${date_stamp} then extract"
          cat $client-$network-snapshot-${date_stamp}-part* > "$output_tar"
          rm $client-$network-snapshot-${date_stamp}-part*
          pv $output_tar | tar -I zstd -xf - -C . && rm $output_tar
      fi
  done

  # Join incremental following day parts
  for file in $(find . -name "$client-$network-snapshot-*-part-*" -print | sort); do
      date_stamp=$(echo "$file" | grep -o 'snapshot-.*-part' | sed 's/snapshot-\(.*\)-part/\1/')
      
      # Check if we have already processed this date
      if [[ -z "${processed_dates[$date_stamp]}" ]]; then
          processed_dates[$date_stamp]=1
          output_tar="$client-$network-snapshot-${date_stamp}.tar.zst"
          echo "Join parts for ${date_stamp} then extract"
          cat $client-$network-snapshot-${date_stamp}-part* > "$output_tar"
          rm $client-$network-snapshot-${date_stamp}-part*
          pv $output_tar | tar -I zstd -xf - -C . --strip-components=3 && rm $output_tar      
      fi
  done
```

</details>

**Note:** If experiencing intermittent aria2c download errors, try reducing concurrency as exampled here:

```bash
aria2c -c -m 0 -x6 -s6 -i $client-$network-parts.txt --max-concurrent-downloads=1
```

Once the extraction is complete, ensure that you update the datadir configuration of your client to point to the path where the extracted data is located. This ensures that the systemd services can correctly register the snapshot data when the client starts. 
If you wish to preserve the default client configuration settings, you can use symbolic links (symlinks).

For example, let's say you have mounted your block device at `~/snapshots` and have downloaded and extracted the chaindata
for Heimdall into the directory `heimdall_extract`, and for Bor into the directory `bor_extract`. To ensure proper registration
of the extracted data when starting the Heimdall or Bor systemd services, you can use the following sample commands:

```bash
# remove any existing datadirs for heimdall and bor
rm -rf /var/lib/heimdall/data
rm -rf /var/lib/bor/chaindata

# rename and setup symlinks to match default client datadir configs
mv ~/snapshots/heimdall_extract ~/snapshots/data
mv ~/snapshots/bor_extract ~/snapshots/chaindata
# make directories /var/lib/bor/data/bor/chaindata and /var/lib/heimdall/data not exist
# /var/lib/heimdall/data -> ~/snapshots/data
sudo ln -s ~/snapshots/data /var/lib/heimdall
# make /var/lib/bor/data/bor/chaindata -> ~/snapshots/chaindata
sudo ln -s ~/snapshots/chaindata /var/lib/bor/data/bor

# bring up clients with all snapshot data properly registered
sudo service heimdalld start
# wait for heimdall to fully sync then start bor
sudo service bor start
```

## Recommended disk size guidance

**Polygon Mumbai Testnet**

| Metric | Calculation Breakdown | Value |
| ------ | --------------------- | ----------- |
| approx. compressed total | 250 GB (bor) + 35 GB (heimdall) | 285 GB |
| approx. data growth daily | 10 GB (bor) + .5 GB (heimdall) | 10.5 GB |
| approx. total extracted size | 350 GB (bor) + 50 GB (heimdall) | 400 GB |
| suggested disk size (2.5x buffer) | 400 GB * 2.5 (natural chain growth) | 1 TB | 

**Polygon Mainnet**

| Metric | Calculation Breakdown | Value |
| ------ | --------------------- | ----------- |
| approx. compressed total | 1500 GB (bor) + 225 GB (heimdall) | 1725 GB |
| approx. data growth daily | 100 GB (bor) + 5 GB (heimdall) | 105 GB |
| approx. total extracted size | 2.1 TB (bor) + 300 GB (heimdall) | 2.4 TB |
| suggested disk size (2.5x buffer) | 2.4 TB * 2.5 (natural chain growth) | 6 TB |

**Polygon Mumbai Erigon Archive**

| Metric | Calculation Breakdown | Value |
| ------ | --------------------- | ----------- |
| approx. compressed total | 210 GB (erigon) + 35 GB (heimdall) | 245 GB |
| approx. data growth daily | 4.5 GB (erigon) + .5 GB (heimdall) | 5 GB |
| approx. total extracted size | 875 GB (erigon) + 50 GB (heimdall) | 925 GB |
| suggested disk size (2.5x buffer) | 925 GB * 2.5 (natural chain growth) | 2.5 TB | 

**NOTE**
PoS Network is deprecating Archive Node snapshots we request users to move to the Erigon Client and make use of Erigon Snapshots.

**Polygon Mainnet Erigon Archive**

Currently under maintenance. ETA Aug 2023 for Erigon bor-mainnet incremental snapshots.

## Recommended disk type and IOPS guidance

- Disk IOPS will impact speed of downloading/extracting snapshots,
  getting in sync, and performing LevelDB compaction
- To minimize disk latency, direct attached storage is ideal.
- In AWS, when using gp3 disk types, we recommend provisioning IOPS of 16000 and
  throughput of 1000 - this minimizes cost and adds a lot of performance. io2 EBS volumes with matching IOPS and throughput values are similarly performant.
- For GCP, we recommend using performance (SSD) persistent disks (`pd-ssd`) or extreme persistent disks (`pd-extreme`) with similar IOPS and throughput values as seen above.
