## Runing

### Interpreter mode
souffle -F./input -D./output example.dl

### Compiler mode
souffle -F. -D. -oexample example.dl
./example [-F./input -D./output]

### Debug report
souffle -F. -D. -rexample.html example.dl

### Profiling
souffle -F. -D. -pexample.log example.dl
souffle-profile example.log
souffle-profile -h

## Input / Output
The tab-seqarated input .facts files can be seen as the extensinal database
(EDB) of the program. 
