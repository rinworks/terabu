# Terabu Design and Implementation Notes

## 2019 March 2A - JMJ: Terabu disk summary representation options and decisions

The "disk summary" is mainly the names and MD5 hash of all the files on the drive. It is a building block for disk content validation and file duplicate detection.

Considerations
- Incrementally create
- Incrementally update
- Incrementally validate
- MD5 of the entire content
- CRC32 checksum of a portion of the content - for quick integrity checks
- Easily ingest into analysis db
- Options to add metadata like starred, location
- Option to cache readmes and other summaries [NO - that can be a separate process]
- Option to record history [No, but could use Github]
- Resists truncation / undetected partial scan

Alternatives
- Single file, multiple files, multiple directories and files
- CSV and / or JSON and / or custom format
- Interpret special metadata files like starred lists and file metadata
- Flat or hierarchical representation of path names
- Progress in files or tracked externally - or bit of both

Decisions
- Only JSON - suck into MongoDB and/or program, less custom parsing
- Each signature generation session generates a single file, logically appending to
  the previous file. Signature files are ordered lexicographically, not by
  creation date, which can't be trusted.
- Each signature file is a complete JSON document, which lists some initial metadata about the scan,
  and then is an array of objects, each object covering one directory. Each directory object
  has the full path and an array of file info objects.
- Ongoing scan state is maintained elsewhere in some mechanism relevant to the software orchestrating
  scans. That software needs to deal with ongoing scans over multiple drives over long periods of time.
- Paths are sorted so there is consistent order
- Each time the scan is started, get the last path name and start where that
  left off .
- Since the state of the drive may be modified during a scan that takes a long time, the verification
  software needs to deal with this in a pragmatic manner that does not inundate the user with
  lots of detail or take a lot of time given that these are terabyte drives.
  File and entire directories could have been deleted or added. Needs more thought.


Sample summary file (comments shown with '//' just for clarification):
```
{
    "machine": "XPS-15",
    "volume_label": "BLUE1",
    "start_utc": 1212323, 		// time of start of scan - UTC seconds (integer)
    "end_utc": 1212323,			// time of end of scan
    "dircount:" 1212323,		// count of directories
    "filecount:" 1212323,		// count of files
    "bytecount:" 1212323L,		// count of bytes from all the files combined
    "dirinfos":[			// list of information blocks, one per directory
        {
	    "path":"/org/archives/archive1", // Path from root of volume (not including drive 'letter')
	    "fileinfos": [	// List of file information blocks, one per file 
		{"md5": "143a1ffd7aa4e7436af2e51bf87f1756", "crc10k":"239090fd", "name":"IMG01.JPG", "size":12312312L}
		// Notes:
		//  -"crc10k" is the CRC32 checksum of the first min(size, 10K) bytes of the file,
		//  	specified as a string that contains hex digits (without leading '0x')
		//	Note that JSON represents numbers as floating point numbers.
		//  -"name" is the file name without any path prefix
		//  -"size" is file size in bytes
	    ]
	},
	...
}
```

In a self-validating drive - that typically contains archived content that will  "never" change,
the summary files will be located in the directory `<volume>_SUMMARIES`, where `<volume>` is
the name of the drive volume. Example: A drive with volume label `BEAN` will have its summary
information files stored under `BEAN_SUMMARIES`. The `_SUMMARIES` suffix will always be all-caps.
The label part will typically match the case of the volume label. If the label is empty, 
the directory becomes `_SUMMARY`

If for some reason the drive is partitioned, each partition will get its own summary directory. 

Under the summary directory, the files are named `<volume>_SUMARY<NNNN>.json`, where `<NNNN>`
is a 4-digit counter padded with zeroes. In the `BEAN` example, the files under 
summary directory `BEAN_SUMMARIES` will be: `BEAN_SUMMARY0001.json`, `BEAN_SUMMARY0002.json`, etc.

The volume label is prefixed to each summary file so that is less chance of summary files across
drives being confused with each other.
