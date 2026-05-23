
## Basic String Operations

|Tool|Command|Description|
|---|---|---|
|`tr`|`echo "hello" \| tr 'a-z' 'A-Z'`|Replace/translate characters|
|`rev`|`echo "hello" \| rev`|Reverse a string|
|`cut`|`echo "a:b:c" \| cut -d: -f2`|Cut by delimiter|
|`fold`|`fold -w 10 file.txt`|Wrap lines at width|
|`tac`|`tac file.txt`|Print file in reverse order|
|`paste`|`paste file1 file2`|Merge files by column|

## Search & Filter

|Tool|Command|Description|
|---|---|---|
|`grep`|`grep "pattern" file.txt`|Search for pattern|
|`grep -r`|`grep -r "pattern" ./`|Recursive search in folder|
|`grep -v`|`grep -v "pattern" file.txt`|Exclude pattern|
|`sed`|`sed 's/foo/bar/g' file.txt`|Find and replace|
|`awk`|`awk '{print $1}' file.txt`|Process by column|

## File Viewing

|Tool|Command|Description|
|---|---|---|
|`cat`|`cat file.txt`|Print entire file|
|`head`|`head -n 10 file.txt`|Print first 10 lines|
|`tail`|`tail -n 10 file.txt`|Print last 10 lines|
|`less`|`less file.txt`|View file page by page|
|`xxd`|`xxd file.txt`|View hex dump|

## Count & Statistics

|Tool|Command|Description|
|---|---|---|
|`wc`|`wc -l file.txt`|Count lines|
|`wc -w`|`wc -w file.txt`|Count words|
|`sort`|`sort file.txt`|Sort lines|
|`sort -u`|`sort -u file.txt`|Sort and remove duplicates|
|`uniq`|`uniq file.txt`|Remove adjacent duplicate lines|
|`uniq -c`|`uniq -c file.txt`|Count occurrences|

## Encode / Decode

|Tool|Command|Description|
|---|---|---|
|`base64`|`echo "hello" \| base64`|Encode base64|
|`base64 -d`|`echo "aGVsbG8=" \| base64 -d`|Decode base64|
|`xxd`|`xxd -r -p hex.txt`|Hex to binary|
|`hexdump`|`hexdump -C file`|View hex + ASCII|
|`od`|`od -c file.txt`|Octal dump|
|ROT13|`echo "hello" \| tr 'A-Za-z' 'N-ZA-Mn-za-m'`|ROT13|

## File Operations

|Tool|Command|Description|
|---|---|---|
|`diff`|`diff file1 file2`|Compare two files|
|`strings`|`strings binary_file`|Extract printable strings|
|`file`|`file unknown_file`|Identify file type|
|`split`|`split -l 100 file.txt`|Split file by lines|

## CTF Tips

- `strings` + `grep` → quickly find flag in binary
- `xxd` + `rev` → reverse hex
- `base64 -d` multiple times → multi-layer encoding
- Always run `file` first → identify file type before analyzing