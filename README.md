# Malware Detection System
Note:Dataset is available in release section
Traditional malware detection engines rely on the use of signatures(unique values that have been manually 
selected by a malware researcher to identify the presence of malicious code).Problem in this approach is that the number of researchers is orders of magnitude smaller than the number of unique malware families they need to manually reverse engineer,
identify and write signatures for leading to easy bypass of detection.
In this repository the goal is to teach a computer, more specifically an artificial neural network,
to detect Windows malware without relying on any explicit signatures database that needs to be created,
but by simply ingesting the dataset of malicious files we want to be able to detect and learning from it
to distinguish between malicious code or not, both inside the dataset itself but, most importantly, 
while processing new, unseen samples.

