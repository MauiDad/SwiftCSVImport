# SwiftCSVImport
Fast Import of CSV 
I will echo the croud on this one, why doesn’t Apple include a CSV parser as part of Foundation?  With all the permutations out there for the CSV format, it would be nice to use a standardized/optimized version.  That being said, I seemed to find a few decent ones on GitHub, but opted to write my own instead.

In my efforts, I found some unexpected (to me anyway) issues relative to  performance.  1) Using the Scanner class is about an order of magnitude slower than a direct parse. 2)  according to a benchmark by Techspot, String building using the + operator is faster than string interpolation  “\(string1)\(string2)”.  3) Using a String as a buffer instead of appending to a string array is faster, 4) according toTony Allevato on Medium.com using strings as UTF16 or UnicodeScalars is faster yet (but I have not attempted to implement).

All in all, I was able to increase my best processing speed from about 4000 records /second using Scanners to 40,000 records/second  using a direct parse.   For a well-structured csv, you can use  1 line of Swift at about 75,000 records/second.  (Well structured files have no column delimiters embedded in quotes and no quotes.) While this is fine for me, I have also read of people who wrote a PHP parser that runs at about 145,000 records/second (on their machine not mine, so not directly comparable.)  I t is probable that PHP could be used with Swift, but I haven’t researched it yet
# Usage
Primary objective of this routine is to grab data from a csv or the pasteboard created by Numbers or other  apps and import it into my project and then write it to Numbers and/or SQLite with a minimum of processing.  If you cut and paste from Numbers, you get  nice clean tab-delimited text with formatted numbers (that is,  a well-structured format), but if you export Numbers to CSV, you get quoted and unquoted text and numbers, with the key criteria on  quoting being the formats in the formatted contents of the cell.  Yeah! Use tab-delimited, am I the only one to ever think of that?   

The other gremlin appears to be that the Numbers csv export format want s to play nice with Windows and uses a CRLF as a line terminator.  i.e. two terminators in a row - why use one, when two will do?  

So the routine below can handle all of these conditions . 

You can call it with the well-behaved parameter set to true and it executes a 1 line swift conversion using input row and column delimiters.  Enter both line delimiters (CRLF = \r\n) if used.  No string manipulation  is performed. This runs at about 75,000  records/second  on a name, address file with 10 columns and 10,000 records.

With playsnice  set to false, the csv is parsed line by line, removing enclosing quotes.  Embedded  column delimiters in strings are retained, but removed on numerics.  This runs at 40,000 records/second on the test file.

Data returned is a columnName array [String] and a data array [[Any]].


## Installation
Copy the code into a Swift file.
*
```

func LoadCSV (fileURL: URL, rowdelimiters: String = cfg_rowdelim, coldelimiters: String = cfg_coldelim, quotes: String = cfg_quotes, playsnice: Bool = false) throws -> (columnNames: [String], data: [[Any]]) {
    let globaltimer = NSDate()
    let quoteSet = CharacterSet(charactersIn: quotes)
    let rowdelimiterSet = CharacterSet(charactersIn: rowdelimiters)
    let coldelimiterSet = CharacterSet(charactersIn: coldelimiters)
    var founddelim: String = ""

    
    func getcoldelim (linetoparse: String) -> String {
      var chars = coldelimiters
        while !chars.isEmpty {
            if linetoparse.components(separatedBy: String(chars.first!)).count > 1 {
                return String(chars.first!)
            }
            chars.removeFirst()
        }
       return String(coldelimiters.first!)
    }
    
    
    func parseline (linetoparse: String, expectedcolumns: Int) throws -> [String] {
        
        let workarray: [String] = linetoparse.components(separatedBy: coldelimiterSet)
        var returnarray: [String] = []
        let quote: Character = "\""
        
        if workarray.count == expectedcolumns {
            return workarray.map {$0.trimmingCharacters(in: quoteSet)}
        }
        
        var combineString: String = ""
        for work in workarray {
            if work.last == quote {
                combineString += work.trimmingCharacters(in: quoteSet)
                returnarray.append(combineString)
                combineString = ""
            } else  if work.first == quote {
                let trimstring: String = work.trimmingCharacters(in: quoteSet)
                combineString += trimstring
                if Double(trimstring) == nil {
                    combineString += founddelim
                }
            } else { returnarray.append(work) }
        }
        return returnarray
    }
    
    let file: String = try String(contentsOf: fileURL)
    if playsnice {
        var fullarray: [[Any]] = file.components(separatedBy: rowdelimiters).map { $0.components(separatedBy: coldelimiters) }
        
        print("total time: \(NSDate().timeIntervalSince(globaltimer as Date))")
        print(fullarray)
        return (Array(fullarray[0] as! [String]), Array(fullarray[1...]))
    }
    
    let lines: [String] = file.components(separatedBy: rowdelimiterSet)
    let names = try parseline(linetoparse: lines[0], expectedcolumns: 0)
    founddelim = getcoldelim(linetoparse: lines[0])
   
    var data: [[Any]] = []
    for i in 1..<lines.count {
        if !lines[i].isEmpty {
            let returnline = try parseline(linetoparse: lines[i], expectedcolumns: names.count)
            data.append(returnline)
        }
    }
    
    print("total time: \(NSDate().timeIntervalSince(globaltimer as Date))")
    return (names, data)    
}

```
Calling:

```
let (names, data) = try LoadCSV(fileURL: URL(fileURLWithPath: path), rowdelimiters: "\n\r", coldelimiters: ",\t", quotes: "\"", playsnice: false)
