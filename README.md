# volman
Terabyte backup to on-premise storage with option for longer term backup to the cloud.

## Scenarios
- Rapidly ingest and backup thousands of images and videos that make up a
  single session of a 3D photo shoot from multiple angles, such as by the
  "CMRig" project (<https://github.com/rinworks/rig>). Be able to retrieve this
  data by using database queries and full-text search.
- Backup a messy backlog of hard drives and CF cards collectively containing
  terabytes of images and videos with many duplicates.

## Desired Features
- Ingest terabytes of data at speeds approaching hard-drive or SSD-drive speeds.
- Find duplicate files and deal with them - i.e., avoid duplication. The specific way this is addressed is TBD.
- Disaster mitigation - background storage to the cloud for higher priority data as well as 'sneakernet' to another physical location.
- Deal with the fact that hard drives go bad at any time, including when disconnected and in storage.
- Do not require availability of a high bandwidth Internet connection. Local backup should be possible without any internet connection.
- The user does not have to remember anything (such as having to retrieve a remotely-located hard disk periodically to check that it is still viable). The system 
  should provide reminders which persist until the system verifies that the situation has been addressed (such as verifying the integrity of a particular hard disk).
- Support gradual migration to new storage technologies - such as higher capacity hard disks.
- Support full-text search into backed-up text files and database queries into image and video metadata (within reason).
- Support generation of smart summaries of large collections of related media - such as a single photo shoot or video shoot. Some suggestions for summaries:
	- Collages made from images and key frames in videos.
	- Text generated through object recognition in images and videos.
	- Smaller/condensed version of a small subset of images and videos and other forms of bulk data
