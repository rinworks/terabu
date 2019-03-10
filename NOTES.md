# Terabu Design and Implementation Notes

## 2019 March 6B - JMJ: Project Structure
Following guidelines in the _Serious Python_ book:
```
terabu/
    docs/
        conf.py
        quickstart.rst
	index.rst
    terabu/
        __init__.py
        tests/
	data/
```
## 2019 March 6A - JMJ: Local backup story
_Considerations_
- Individual bare drives are not great to handle - to keep inserting/removing from docking stations,
  so consider a set of drives in an enclosure as a unit.
- Any drive can fail at any time. Also there are unsubstantiated reports that suggest that drives
  manufactured in the same batch may fail at the same time (I find this hard to believe, but
  the guidance is that if possible to use drives from different batches or manufacturers)
- HA is not important for the main scenario - which is managing vast quantities of images and 
  videos coming in in bursts. But we're not running a high available service.
- There should be a backup at a remote location - fire or (more likely )theft are distinct
  possibilities.
- Given that drives may be stolen, they should be encrypted. Remote drives should also be encrypted
  so that the entity that houses that drive is not responsible for data being stolen. If that remote
  location is a secure location (bank) then there is no need for encryption.
- Ransomware is another threat. Copies should mostly live their lives disconnected.
- Drive capacity keeps increasing, so drives should be allowed to be incrementally replaced.
- Primary drives can potentially be higher speed as the back up drives are only used for backup.


_Decisions_
- Three drive banks. Each bank is a physical enclosure, such as "Mediasonic USB
  3.1 4 Bay 3.5” SATA Hard Drive Enclosure", holding 4 drives. One bank is primary and the other
  two are secondary. This designation is done manually and does not change - if necessary a drive
  or enclosure is replaced to keep everything working.
- Each drive in a bank has a 'sister' drive of the exact same capacity in the other two banks.
  The three drives that form a 'sisterhood' are called _NAME_0, _NAME_1, and _NAME_2, with
  _NAME_0 being the primary drive. They are single-volume drives with volume label set to their
  name. The backed-up data in each drive is stored under the top-level
  folder _NAME_. _NAME_ has at most 4 letters, all caps. It is kept small to keep overall path
  lengths from getting too long.


Following is a possible assignment

Primary|Secondary1|Secondary2|Capacity
-------|----------|----------|--------
BAT0   |BAT1      |BAT2      |1TB
BLUE0  |BLUE1     |BLUE2     |2TB
DOG0   |DOG1      |DOG2      |3TB
CAT0   |CAT1      |CAT2      |6TB



## 2019 March 2A - JMJ: Terabu disk summary representation options and decisions

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
- CSV | JSON | custom format
- Interpret special metadata files like starred lists and file metadata
- Flat or hierarchical representation of path names
- Progress in files or tracked externally - or bit of both

_Decisions_
- Summary is only JSON - suck into MongoDB and/or program, less custom parsing. CSV is too brittle.
- Each summary generation session generates a single file, logically appending to
  the previous file. Signature files are ordered lexicographically, not by
  creation date.
- Each signature file is a complete JSON document, which lists initial metadata about the scan followed
  by an array of objects, each object covering one directory. Each directory object
  has the full path and an array of file information objects.
- Ongoing scan state is maintained elsewhere in some mechanism relevant to the software orchestrating
  scans. That software needs to deal with ongoing scans over multiple drives over long periods of time.
- Paths are sorted so incremental scanning proceeds in a consistent order.
- Each time a scan is started, get the last path name and start where that
  left off.
- Since the state of the drive may be modified during a scan that takes a long time, the verification
  software needs to deal with this in a pragmatic manner that does not inundate the user with
  lots of detail or take a lot of time given that these are terabyte drives.
  File and entire directories could have been deleted or added. The design needs more fleshing-out.


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
	     <more file info blocks...>

	    ]
	},
	<more directory info blocks ...>
}
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
directory is copied to another volume with a different label. The general guideline for terabu
volumes is for every backed-up drive to have a unique volume label.
