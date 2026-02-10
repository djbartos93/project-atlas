# Interface Configuration

This is working documentation and will be updated and organized as the build progresses.

| Device A       | Interface A | IP Address A     | A Configured? | Device B        | Interface B | B IP Address    | B Configured? | Ping Tested |
| -------------- | ----------- | ---------------- | ------------- | --------------- | ----------- | --------------- | ------------- | ----------- |
| L3-BR-1        | ge-0/0/5    | 10.1.0.12        | TRUE          | L3-BR-2         | ge-0/0/5    | 10.1.0.13       | TRUE          | TRUE        |
| L3-CR-1        | ge-0/0/9    | 10.1.0.2         | TRUE          | L3-CR-2         | ge-0/0/9    | 10.1.0.3        | TRUE          | TRUE        |
| L3-CR-1        | ge-0/0/0    | 10.1.0.4         | TRUE          | L3-BR-1         | ge-0/0/0    | 10.1.0.5        | TRUE          | TRUE        |
| L3-CR-2        | ge-0/0/0    | 10.1.0.10        | TRUE          | L3-BR-2         | ge-0/0/0    | 10.1.0.11       | TRUE          | TRUE        |
| L3-CR-1        | ge-0/0/1    | 10.1.0.6         | TRUE          | L3-BR-2         | ge-0/0/1    | 10.1.0.7        | TRUE          | TRUE        |
| L3-CR-2        | ge-0/0/1    | 10.1.0.8         | FALSE         | L3-BR-1         | ge-0/0/1    | 10.1.0.9        | TRUE          | TRUE        |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
| L3-BR-1        | ge-0/0/2    |                  | FALSE         | ALT-MSN-BR-1    | Gi1         |                 | FALSE         | FALSE       |
| L3-BR-2        | ge-0/0/2    |                  | FALSE         | ALT-MSN-BR-2    | Gi1         |                 | FALSE         | FALSE       |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
| CG-CR-1        | gi0/0/0/0   | 10.2.0.2/31      | TRUE          | CG-CR-2         | gi0/0/0/0   | 10.2.0.3/31     | TRUE          | TRUE        |
| CG-CR-1        | gi0/0/0/2   | 10.2.0.4/31      | TRUE          | CG-BR-1         | gi0/0/0/2   | 10.2.0.5/31     | TRUE          | TRUE        |
| CG-CR-1        | gi0/0/0/1   | 10.2.0.6/31      | TRUE          | CG-BR-2         | gi0/0/0/1   | 10.2.0.7/31     | TRUE          | TRUE        |
| CG-CR-2        | gi0/0/0/2   | 10.2.0.10        | TRUE          | CG-BR-2         | gi0/0/0/2   | 10.2.0.11       | TRUE          | TRUE        |
| CG-CR-2        | gi0/0/0/1   | 10.2.0.8         | TRUE          | CG-BR-1         | gi0/0/0/1   | 10.2.0.9        | TRUE          | TRUE        |
| CG-BR-1        | gi0/0/0/0   | 10.2.0.12        | TRUE          | CG-BR-2         | gi0/0/0/0   | 10.2.0.13       | TRUE          | TRUE        |
| CG-BR-1        | gi0/0/0/5   | 198.51.100.52    | TRUE          | MADIX-1         | eth3        | n/a             | FALSE         | FALSE       |
| CG-BR-1        | future      | future           | FALSE         | MADIX-2         | future      | future          | FALSE         | FALSE       |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
| CG-BR-2        | gi0/0/0/5   | 198.51.100.62    | TRUE          | MADIX-1         | eth5        | n/a             | FALSE         | FALSE       |
| CG-BR-2        | future      | future           | FALSE         | MADIX-2         | future      | future          | FALSE         | FALSE       |
| MADIX-1        | eth1        | future           | FALSE         | MADIX-2         | eth1        | future          | FALSE         | FALSE       |
| MADIX-1        | eth2        | n/a              | FALSE         | ATLAS-MSN-CR-1  | gi5         | 198.51.100.53   | TRUE          | FALSE       |
| MADIX-1        | eth4        | n/a              | FALSE         | ATLAS-MSN-CR-2  | gi5         | 198.51.100.63   | TRUE          | FALSE       |
| MADIX-2        | eth2        | future           | FALSE         | ATLAS-MSN-CR-1  | gi6         | future          | FALSE         | FALSE       |
| MADIX-2        | eth4        | future           | FALSE         | ATLAS-MSN-CR-2  | gi6         | future          | FALSE         | FALSE       |
| L3-BR-1        | ge-0/0/4    | 198.51.100.51    | TRUE          | MADIX-1         | eth9        | n/a             | FALSE         | FALSE       |
| L3-BR-1        | future      | future           | FALSE         | MADIX-2         | future      | future          | FALSE         | FALSE       |
| L3-BR-2        | ge-0/0/4    | 198.51.100.61    | TRUE          | MADIX-1         | eth6        | n/a             | FALSE         | FALSE       |
| L3-BR-2        | future      | future           | FALSE         | MADIX-2         | future      | future          | FALSE         | FALSE       |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
| L3-BR-1        | ge-0/0/6    | 203.0.113.17/30  | TRUE          | CG-BR-1         | gi0/0/0/4   | 203.0.113.18/30 | TRUE          | FALSE       |
| L3-BR-2        | ge-0/0/6    | 203.0.113.21/30  | TRUE          | CG-BR-2         | gi0/0/0/6   | 203.0.113.22/30 | TRUE          | FALSE       |
|                |             |                  | FALSE         |                 |             |                 | FALSE         | FALSE       |
| ATLAS-MSN-BR-1 | gi4         |                  | FALSE         | ATLAS-MSN-BR-2  | gi4         |                 | FALSE         | FALSE       |
| ATLAS-MSN-BR-1 | gi3         | 10.0.1.5/30      | FALSE         | ATLAS-MSN-CR-1  | gi1         | 10.0.1.4/30     | FALSE         | FALSE       |
| ATLAS-MSN-BR-1 | gi2         | 10.0.1.13/30     | FALSE         | ATLAS-MSN-CR-2  | gi2         | 10.0.1.12/30    | FALSE         | FALSE       |
| ATLAS-MSN-BR-1 | gi7         | 198.51.100.53/24 | FALSE         | MADIX-1         | eth2        |                 | FALSE         | FALSE       |
| ATLAS-MSN-BR-2 | gi2         | 10.0.1.7/30      | FALSE         | ATLAS-MSN-CR-1  | gi2         | 10.0.1.8/30     | FALSE         | FALSE       |
| ATLAS-MSN-BR-2 | gi3         | 10.0.1.17/30     | FALSE         | ATLAS-MSN-CR-2  | gi1         | 10.0.1.16/30    | FALSE         | FALSE       |
| ATLAS-MSN-BR-2 | gi7         | 198.51.100.63/24 | FALSE         | MADIX-1         | eth4        |                 | FALSE         | FALSE       |
| ATLAS-MSN-CR-1 | gi4         | 10.0.1.2/30      | FALSE         | ATLAS-MSN-CR-2  | gi4         | 10.0.1.3/30     | FALSE         | FALSE       |
| ATLAS-MSN-BR-1 |             |                  | FALSE         | Customer Router | eth0        |                 | FALSE         | FALSE       |
| ATLAS-MSN-BR-2 |             |                  | FALSE         | Customer Router | eth1        |                 | FALSE         | FALSE       |



## Loopback interfaces

| Device        | Interface | IP               | ISO                       |
| ------------- | --------- | ---------------- | ------------------------- |
| L3-CR-1       | lo0       | 10.255.1.1/32    | 49.0001.0102.5500.1001.00 |
| L3-CR-2       | lo0       | 10.255.1.2/32    | 49.0001.0102.5500.1002.00 |
| L3-BR-1       | lo0       | 10.255.1.11/32   | 49.0001.0102.5500.1011.00 |
| L3-BR-2       | lo0       | 10.255.1.12/32   | 49.0001.0102.5500.1012.00 |
| ATLAS-MSN-BR1 |           | 10.0.0.11/32     | N/A - no ISIS configured  |
| ATLAS-MSN-BR2 |           | 10.0.0.12/32     | N/A - no ISIS configured  |
| ATLAS-MSN-CR1 |           | 10.0.0.1/32      | N/A - no ISIS configured  |
| ATLAS-MSN-CR2 |           | 10.0.0.2/32      | N/A - no ISIS configured  |
| CG-CR-1       | loopback0 | 10.255.2.1/32    | 49.0002.0102.5500.2001.00 |
| CG-CR-2       | loopback0 | 10.255.2.2/32    | 49.0002.0102.5500.2002.00 |
| CG-BR-1       | loopback0 | 10.255.2.11/32   | 49.0002.0102.5500.2011.00 |
| CG-BR-2       | loopback0 | 10.255.2.12/32   | 49.0002.0102.5500.2012.00 |
| PN-CR-1       |           |                  |                           |
| PN-CR-2       |           |                  |                           |
| MADIX-1       | loopback0 | 10.255.99.251/32 | N/A - no ISIS configured  |
|               |           |                  |                           |