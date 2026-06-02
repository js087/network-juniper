# 2026 - June
## mx204; 20.4R3 >> 23.4R2 journey + some cool routing and memory optmizations
- This started as an idea like... 2 years ago and due to the unique nature of our configurations there were some of the changes below couldn't use for various reasons until '23.4R2'
    - ECMP and multiple routing tables were not supported with some of these tweaks
    - JunOS memory increase was not available until a later version
- JTAC notes + random sources from the web aiding in this journey:
    - `JTAC seems to think this sharding + threading with multipath/ecmp support may have come around 22.4R1 but documentation doesn’t confirm 100%; there was mention of NSR with 22.1 and multiple REs.`
    - Curious read from Juniper about - [RIB sharding](https://community.juniper.net/blogs/ravindran-thangarajah/2022/10/24/bgp-rib-sharding)
       - There are some great graphs and measurements gauging the pros and cons of threading and sharding and challenges
    - Juniper forum post that lead to discovery of increasing JunOS memory availabilities (starting in `Junos 21.2R2`) - [Juniper Post](https://community.juniper.net/discussion/mx204-and-re-s-1600x8-available-memory)
       - `Repeated crash and core dump of the RPD process due to the 32-bit RPD memory resource limitation for a logical system`
          - [KB96776](https://supportportal.juniper.net/s/article/Repeated-crash-and-core-dump-of-the-RPD-process-due-to-the-32-bit-RPD-memory-resource-limitation-for-a-logical-system?language=en_US) - definitely ran into memory pressure issues when dealing with 4-5 full internet tables, but the `force-64-bit` + added memory seems to have aided in stabilizing things.


## configs and commands enabling routing and memory optmizations



#### tell junos to use more RAM; default 16GB, increase to 24GB - (32GB physical RAM) - ***requires 'root' access and a reboot***

`request vmhost exec "set_vjunos_memory high"`


#### show command to look at memory available and used:
`show task memory logical-system all`

```
!before change and reboot
Available:            17107702

!after change and reboot
Available:            25088136
```

## tell `rpd` to: 
- use 64-bit if possible, fall back to 32-bit (hidden to users)
- multithread and shard crawling routes (useful for multiple for bgp tables)
- changing threads and shards will impact routing as `rpd` restarts
- `force-64-bit` can be added without impact in my experience

```
set system processes routing force-64-bit
set system processes routing bgp rib-sharding number-of-shards 2
set system processes routing bgp update-threading number-of-threads 2
```

#### show command to look for rpd threads
`show system processes extensive | match rpd`

##### default mx204 config; 1u/no shards; no memory increase; 4 x full routing tables
```
show task memory logical-system all 
logical-system: default
Memory                 Size (kB)  Percentage  When
  Currently In Use:      6094127         35%  now
  Maximum Ever Used:     6096237         35%  yy/mm/dd hh:mm:ss
  Available:            17107702        100%  now

show system processes extensive | match rpd 
32672 root           20    0  7736M  6615M kqread  4 362.1H   0.59% rpd{rpd}
32672 root           20    0  7736M  6615M kqread  2  62.7H   0.00% rpd{bgpio-0-th}
32672 root           20    0  7736M  6615M kqread  2  26.3H   0.00% rpd{krtio-th}
32672 root           20    0  7736M  6615M kqread  0 814:06   0.00% rpd{TraceThread}
32726 root           20    0   884M 22412K kqread  4   9:26   0.00% rpdtmd
```

##### 2u/2s; memory increase; 3 x full routing tables
```
show task memory logical-system all 
logical-system: default
Memory                 Size (kB)  Percentage  When
  Currently In Use:      9444716         37%  now
  Maximum Ever Used:     9789068         39%  yy/mm/dd hh:mm:ss
  Available:            25088136        100%  now

!`rpd` high here because actively crawling routes
show system processes extensive | match rpd 
17117 root        101    0    11G    10G CPU0     0   5:05  93.07% rpd{rpd}
17117 root         87    0    11G    10G CPU1     1   0:40  49.85% rpd{krtio-th}
17117 root         30    0    11G    10G kqread   3   1:42  23.58% rpd{junos-bgpshard0}
17117 root         29    0    11G    10G kqread   0   1:42  23.10% rpd{junos-bgpshard1}
18989 root         20    0   799M  7256K kqread   2   0:05   0.00% rpdtmd
17117 root         20    0    11G    10G kqread   4   0:01   0.00% rpd{bgp-updio-1}
17117 root         20    0    11G    10G kqread   0   0:01   0.00% rpd{bgp-updio-0}
17117 root         20    0    11G    10G kqread   2   0:00   0.00% rpd{TraceThread}
```

##### 2u/2s; memory increase; 4 x full routing tables + Internet Exchange (~38 peers between ipv4 and ipv6)
```
show task memory logical-system all 
logical-system: default
Memory                 Size (kB)  Percentage  When
  Currently In Use:     10857884         43%  now
  Maximum Ever Used:    11207488         44%  yy/mm/dd hh:mm:ss
  Available:            25088136        100%  now

show system processes extensive | match rpd 
25843 root         20    0    14G    13G kqread   1 258:45   0.68% rpd{rpd}
25843 root         20    0    14G    13G kqread   4 256:01   0.49% rpd{junos-bgpshard1}
25843 root         20    0    14G    13G kqread   3 252:42   0.39% rpd{junos-bgpshard0}
25843 root         20    0    14G    13G kqread   0  33:15   0.00% rpd{bgp-updio-0}
25843 root         20    0    14G    13G kqread   2  31:04   0.00% rpd{bgp-updio-1}
25843 root         20    0    14G    13G kqread   3  30:44   0.00% rpd{krtio-th}
25843 root         20    0    14G    13G kqread   2   9:53   0.00% rpd{TraceThread}
23635 root         20    0   799M    26M kqread   3   0:07   0.00% rpdtmd
```
