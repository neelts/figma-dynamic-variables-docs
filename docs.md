# Figma Dynamic Variables Documentation (WIP)

## Relative Paths
1. Same Group: {{ $./variable }}
2. Sub Group: {{ $./sub-group/variable }}
3. Parent Group: {{ $../variable }}
4. Sibling Group: {{ $../sibling-group/variable }}

## Advanced Features
- $$.name // returns the name of the current variable 
- $$.id // internal figma id of the variable 
- $$.type // returns type of variable 
- $$.alias // returns alias 
- $$.collection // current collection name 
- $$.collectionId // internal figma collection id 
- $$.mode // mode name 
- $$.modeIndex // current mode index 
- $$.modeId // internal figma mode id 
- $$.modes // returns array of mode names. ex: ["Desktop", "Tablet", "Mobile"]. 
- $$.values // returns array of mode values 
