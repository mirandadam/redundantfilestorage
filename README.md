# redundantfilestorage
Brainstorming for the requirements for storing files in multiple hard drives with redundancy and fault tolerance


# Inspiration

brtfs raid 5/6 modes

# Requirements
* Able to store files in several physical or logical devices
* gracefully handle device failures up to M=N-1 devices, where N is the number of devices used and M is a redundancy factor 
* gracefully handle device disconnect during writes then reconnected later
* automatically self heal and reduce volume capacity when a device fails, provided that there is enough capacity left to hold the data at the required redundancy.
* gracefully decrease redundancy level and self rebuild when a drive fails. For example, in a volume where N=3 (3 devices) and N=2 (2 drives can fail and data can be preserved), if a device fail the volume will be rebuilt with N=1. Or if M=4, N=2 and a drive fails but the data in the array cannot be held with one drive capacity, the volume degrades to M=3, N=1.
* provide a means to push alerts of degradation
