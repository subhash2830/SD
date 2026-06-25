| Auth Type  | Area level                               | Interface level                       |                                                  |
| ---------- | ---------------------------------------- | ------------------------------------- | ------------------------------------------------ |
| Null       | No Cli                                   | ip ospf authentication Null           | No Cli                                           |
| Clear Text | area 0 authenticate                      | ip ospf authentication                | ip ospf authentication-key < value >             |
| MD5        | area o md5 authentication message-digest | ip ospf authentication message-digest | ip ospf message-digest-key key num md5 < value > |
| SHA        |                                          | key chain 1                           |                                                  |
|            |                                          | key 1                                 |                                                  |
|            |                                          | key-string password                   |                                                  |
|            |                                          |                                       |                                                  |
	