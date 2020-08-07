# Volman Design and Implementation Notes

## 2019 March 2A - JMJ: Volman disk summary representation options and decisions
UPDATE: Edited on August 8, 2020 to reflect change from JSON to YAML for files and 
other changes.

The "disk summary" is mainly the names and MD5 hash of all the files on a drive
volume. It is a building block for disk content validation and file duplicate
detection.

_Considerations_
- Incrementally create, update and validate
- MD5 hash of the entire content
- CRC32 checksum of a portion of the content - for quick partial file integrity checks
- Easily ingest into database for analysis
- Options to add metadata like "starred" files and location of shoots, parsed from README files
  embedded in directories
- Option to cache readmes and other summaries [UPDATE: Not doing - would be a different project]
- Option to record history [No, but could use Github]
- Resists truncation / undetected partial scan

_Design Alternatives_
- Summary is a single file | multiple files | multiple directories and files
- CSV | JSON | YAML | custom format
- Interpret special metadata files like starred lists and file metadata
- Flat or hierarchical representation of path names
- Progress in files or tracked externally - or bit of both

_Decisions_
- Summary is only YAML. JSON was considered earlier so it's easy to  suck into MongoDB and/or program.
  CSV is too brittle. YAML main benefit is that files can be incrementally appended to. JSON precludes
  this because of the need to close braces, etc.
- Each summary generation session generates a single file, logically appending to
  the previous file. Signature files are ordered lexicographically, not by
  creation date. Only the last file (if present) is examined to determine the point to continue
  the scan.
- Each signature file is a YAML document, which lists initial metadata about the scan followed
  by an array of objects, each object covering one directory. Each directory object
  has the full path and an array of file information objects.
- Ongoing scan state is maintained elsewhere in some mechanism relevant to the software orchestrating
  scans. That software needs to deal with ongoing scans over multiple drives over long periods of time.
  The simplest mechanism is to simply look at the last of the previous scan files and find the point
  to continue the scan.
- Paths are sorted so incremental scanning proceeds in a consistent order.
- Each time a scan is started, get the last path name and start where that
  left off.
- Since the state of the drive may be modified during a scan that takes a long time, the verification
  software needs to deal with this in a pragmatic manner that does not inundate the user with
  lots of detail or take a lot of time given that these are terabyte drives.
  File and entire directories could have been added, deleted or moved. The design needs more fleshing-out.


[OLD OBSOLETE JSON VERSION]
Sample summary file (comments shown with '//' just for clarification):

```

{
    "machine": "XPS-15",
    "volume_label": "BLUE1",
    "start_utc": "2019-03-02T18:52:52Z" // time of start of scan - UTC ISO 8601, in "Zulu" time
    "end_utc": "2019-03-02T19:01:47Z" 	// time of end of scan
    "dircount": 1212323,		// count of directories in summary
    "filecount": 1212323,		// count of files in summary
    "bytecount": 1212323L,		// total count of bytes of all scanned files
    "dirinfos":[			// list of directory information blocks
        {
	    "path":"/org/archives/archive1", // Path from root of volume (not including drive 'letter')
	    				     // Note: forward-slash represents directory separator
	    "fileinfos": [	// List of file information blocks
		{"md5": "143a1ffd7aa4e7436af2e51bf87f1756", "crc10k":"239090fd", "name":"IMG01.JPG", "size":12312312L},
		// Notes:
		//  -"crc10k" is the CRC32 checksum of the first min(size, 10K) bytes of the file,
		//  	specified as a string that contains hex digits (without leading '0x').
		//	A string is used because JSON represents numbers as floating point numbers.
		//  -"name" is the file name without any path prefix
		//  -"size" is file size in bytes
        //  
	     <more file info blocks...>

	    ]
	},
	<more directory info blocks ...>
}

```

YAML 
Sample summary file (comments shown with '//' just for clarification):
```
    machine: "XPS-15", // where scan was done
    volume_label: "BLUE1", // volume of disk partition scanned.
    start_utc: "2019-03-02T18:52:52Z" // time of start of scan - UTC ISO 8601, in "Zulu" time
    dirs: // list of directory information blocks
    -  path:"/org/archives/archive1", // Path from root of volume (not including drive 'letter')
	    				     // Note: forward-slash represents directory separator
       files: // List of file information blocks
		 - {md5: "143a1ffd7aa4e7436af2e51bf87f1756", crc10k:"239090fd", name:"IMG01.JPG", size:12312312, cr_utc: "...", mod_utc: "...", error:true}
		// Notes:
		//  -"crc10k" is the CRC32 checksum of the first min(size, 10K) bytes of the file,
		//  	specified as a string that contains hex digits (without leading '0x').
		//	A string is used because JSON represents numbers as floating point numbers.
		//  - name: is the file name without any path prefix
		//  - size: is file size in bytes
        //  - cr_utc: creation date in UTC
        //  - mod_utc: modification date in UTC
        //  - error: [optional] Records if an error occurred attempting to read the file.
        //          (most common cause would be the file is locked or a permissions issue).
        //    In this case various other fields, such as "md5" would simply be missing.
	     <more file info blocks...>
    end_utc: "2019-03-02T19:01:47Z" 	// time of end of scan
    dircount: 1212323,		// count of directories in summary
    filecount: 1212323,		// count of files in summary
    bytecount: 1212323,		// total count of bytes of all scanned files
    

```

In a self-validating drive - that typically contains archived content that will  "never" change,
the summary files will be located in the root directory
`<volume>_tbu_summaries`, where `<volume>` is the name of the drive volume.
Example: A drive with volume label `BEAN` will have its summary information
files stored under `BEAN_tbu_summaries`. The `_tbu_summaries` suffix will
always be in lower case.  The label part will typically match the case of the volume
label. If the label is empty, the directory becomes `_tbu_summaries`. The `tbu` part of the suffix
is an acronym for "terabyte back up" and is inserted to reduce the chance of a conflict with
an existing root-level directory.

If for some reason the drive is partitioned, each partition will get its own root-level 
summary directory. 

Under the summary directory, the files are named `<volume>_summary<NNNN>.json`, where `<NNNN>`
is a 4-digit counter padded with zeroes. In the `BEAN` example, the files under 
summary directory `BEAN_tbu_summaries` will be: `BEAN_summary0001.json`, `BEAN_summary0002.json`, etc.

The volume label is prefixed to each summary file so that is less chance of summary files across
drives being confused with each other, and these files will become 'invalid' if the summary
directory is copied to another volume with a different label. The general guideline for Volman
volumes is for every backed-up drive to have a unique volume label.


## 2020 August 7 - JMJ: How to update a scan?
It will take several hours to do a full scan. Simplest solution: we do not support incremental updates. A scan, once started, must be finished.
It can be re-started mid-stream of course, but it is a sequential scan. When complete, perhaps a 2nd scan can be done,
that is much faster - verifying file paths and sizes. After this 2nd step, the scan can be considered "complete and verified"
as of some date.

Initially (before orchestration software is built ) this process is manual. So this suggests a
readme-file could contain this fact - when the last scan was verified.

Some file size and count measurements:
BEAN1\BEAN1: 477GB, 6K files, 1K dirs. F/D ratio: 6. AvgS: 79 MB/file
Program(x86): 4.5GB, 12K files, 2.6K dirs. ratio: 4.6 AvgS: 0.38 MB/file
Users: 112GB, 290Kfiles, 43K dirs. Ratio:  6.7. AvgS: 0.39 MB/file - about 23MB for 100bytes per file.
