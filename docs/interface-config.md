# Interface Configuration

This is working documentation and will be updated and organized as the build progresses.

| Device A | Interface A | IP Address A  | A Configured? | Device B       | Interface B | B IP Address  | B Configured? | Ping Tested |
| -------- | ----------- | ------------- | ------------- | -------------- | ----------- | ------------- | ------------- | ----------- |
| L3-BR-1  | ge-0/0/5    | 10.1.0.12     | TRUE          | L3-BR-2        | ge-0/0/5    | 10.1.0.13     | TRUE          | TRUE        |
| L3-CR-1  | ge-0/0/9    | 10.1.0.2      | TRUE          | L3-CR-2        | ge-0/0/9    | 10.1.0.3      | TRUE          | TRUE        |
| L3-CR-1  | ge-0/0/0    | 10.1.0.4      | TRUE          | L3-BR-1        | ge-0/0/0    | 10.1.0.5      | TRUE          | TRUE        |
| L3-CR-2  | ge-0/0/0    | 10.1.0.10     | TRUE          | L3-BR-2        | ge-0/0/0    | 10.1.0.11     | TRUE          | TRUE        |
| L3-CR-1  | ge-0/0/1    | 10.1.0.6      | TRUE          | L3-BR-2        | ge-0/0/1    | 10.1.0.7      | TRUE          | TRUE        |
| L3-CR-2  | ge-0/0/1    | 10.1.0.8      | FALSE         | L3-BR-1        | ge-0/0/1    | 10.1.0.9      | TRUE          | TRUE        |
|          |             |               | FALSE         |                |             |               | FALSE         | FALSE       |
| L3-BR-1  | ge-0/0/2    |               | FALSE         | ALT-MSN-BR-1   | Gi1         |               | FALSE         | FALSE       |
| L3-BR-2  | ge-0/0/2    |               | FALSE         | ALT-MSN-BR-2   | Gi1         |               | FALSE         | FALSE       |
|          |             |               | FALSE         |                |             |               | FALSE         | FALSE       |
| CG-CR-1  | gi0/0/0/0   | 10.2.0.2/31   | TRUE          | CG-CR-2        | gi0/0/0/0   | 10.2.0.3/31   | TRUE          | TRUE        |
| CG-CR-1  | gi0/0/0/2   | 10.2.0.4/31   | TRUE          | CG-BR-1        | gi0/0/0/2   | 10.2.0.5/31   | TRUE          | TRUE        |
| CG-CR-1  | gi0/0/0/1   | 10.2.0.6/31   | TRUE          | CG-BR-2        | gi0/0/0/1   | 10.2.0.7/31   | TRUE          | TRUE        |
| CG-CR-2  | gi0/0/0/2   | 10.2.0.10     | TRUE          | CG-BR-2        | gi0/0/0/2   | 10.2.0.11     | TRUE          | TRUE        |
| CG-CR-2  | gi0/0/0/1   | 10.2.0.8      | TRUE          | CG-BR-1        | gi0/0/0/1   | 10.2.0.9      | TRUE          | TRUE        |
| CG-BR-1  | gi0/0/0/0   | 10.2.0.12     | TRUE          | CG-BR-2        | gi0/0/0/0   | 10.2.0.13     | TRUE          | TRUE        |
| CG-BR-1  | gi0/0/0/5   | 198.51.100.52 | FALSE         | MADIX-1        | eth3        | n/a           | FALSE         | FALSE       |
| CG-BR-1  | future      | future        | FALSE         | MADIX-2        | future      | future        | FALSE         | FALSE       |
| CG-BR-2  | gi0/0/0/5   | 198.51.100.62 | FALSE         | MADIX-1        | eth5        | n/a           | FALSE         | FALSE       |
| CG-BR-2  | future      | future        | FALSE         | MADIX-2        | future      | future        | FALSE         | FALSE       |
| MADIX-1  | eth1        | future        | FALSE         | MADIX-2        | eth1        | future        | FALSE         | FALSE       |
| MADIX-1  | eth2        | n/a           | FALSE         | ATLAS-MSN-CR-1 | gi5         | 198.51.100.53 | FALSE         | FALSE       |
| MADIX-1  | eth4        | n/a           | FALSE         | ATLAS-MSN-CR-2 | gi5         | 198.51.100.63 | FALSE         | FALSE       |
| MADIX-2  | eth2        | future        | FALSE         | ATLAS-MSN-CR-1 | gi6         | future        | FALSE         | FALSE       |
| MADIX-2  | eth4        | future        | FALSE         | ATLAS-MSN-CR-2 | gi6         | future        | FALSE         | FALSE       |
| L3-BR-1  | ge-0/0/4    | 198.51.100.51 | FALSE         | MADIX-1        | eth9        | n/a           | FALSE         | FALSE       |
| L3-BR-1  | future      | future        | FALSE         | MADIX-2        | future      | future        | FALSE         | FALSE       |
| L3-BR-2  | ge-0/0/4    | 198.51.100.61 | FALSE         | MADIX-1        | eth6        | n/a           | FALSE         | FALSE       |
| L3-BR-2  | L3-BR-2     | future        | FALSE         | MADIX-2        | future      | future        | FALSE         | FALSE       |
