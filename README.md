# Malware Detection API leveraging Artificial Neural Network
<p align="center">
  <img src="https://user-images.githubusercontent.com/47772616/95663618-583a9e00-0b5e-11eb-8be8-79f6bd4dc811.png" />
</p>

Conceptualised from :  

>### Saad, Sherif & Briguglio, William & Elmiligi, Haytham. (2019). The Curious Case of Machine Learning In Malware Detection. 10.5220/0007470705280535. 

# Introduction:

Traditional malware detection engines rely on the use of signatures(unique values that have been manually 
selected by a malware researcher to identify the presence of malicious code).  
Problem in this approach is that the number of researchers is orders of magnitude smaller than the number of unique malware families they need to manually reverse engineer,
identify and write signatures for leading to easy bypass of detection.  

The goal is to teach a computer, more specifically an artificial neural network,
to detect Windows malware without relying on any explicit signatures database that needs to be created,
but by simply ingesting the dataset of malicious files we want to be able to detect and learn from it
to distinguish between malicious code or not, both inside the dataset itself but, most importantly, 
while processing new, unseen samples.  

Since training and testing the model on the very same dataset wouldn’t make much sense (as it could perform extremely well on the training set, but not being able to generalize at all on new samples), this dataset is divided into 3 sub-sets:
- A training set, with 70% of the samples, used for training.
- A validation set, with 15% of the samples, used to benchmark the model at each training epoch.
- A test set, with 15% of the samples, used to benchmark the model after training.  
(Since the amount of correctly labelled samples in dataset constitue the model's accuracy more samples are used in training)  

## The Windows PE format:  
Elucidated here:https://docs.microsoft.com/en-us/windows/win32/debug/pe-format  

The key facts that must be kept in mind are:

- A PE has several headers describing its properties and various addressing details, such as the base address the PE is going to be loaded in memory and where the entry point is.
- A PE has several sections, each one containing data (constants, global variables, etc), code (in which case the section is marked as executable) or sometimes both.
- A PE contains a declaration of what API are imported and from what system libraries.

# Methodology, Coding and Implementation  
What we’re going to do now is looking at how we can encode the PE Header values that are very heterogeneous in nature (they’re numbers of all types of intervals and strings of variable length) into a vector of scalar numbers, each normalized in the interval [0.0,1.0], and of constant length. This is the type of input that the machine learning model is able to understand.  

## Features Engineering  
The process of determining which features of the PE to consider is possibly the most important part of designing any machine learning system and it’s called features engineering, while the act of reading these values and encoding them is called features extraction.  

Features extraction algorithm is inside the **encode.py** file described as follows:  


The library LIEF parses a set of boolean properties from PE each of which is encoded to a 1.0 if true, or to a 0.0 if false using first 11 scalars of the vector.  

Then 64 elements follow, representing the first 64 bytes of the PE entry point function, each normalized to [0.0,1.0] by dividing each of them by 255 - this will help the model detecting those executables that have very distinctive entrypoints that only vary slightly among different samples of the same family.  

Then a histogram of the repetitions of each byte of the ASCII table (therefore size 256) in the binary file follows - this data point will encode basic statistical information about the raw contents of the file.  

The next thing encoded in the features vector is the import table, as the API being used by the PE is quite a relevant information.For this 150 most comonly used libraries are taken from the dataset and for each API being used by the PE, one the column of the relative library is incremented,creating another histogram of 150 values then normalized by the total amount of API being imported.  

Next encoded is the ratio of the PE size on disk vs the size it’ll have in memory (its virtual size).  

Next,some information about the PE sections, such the amount of them containing code vs the ones containing data, the sections marked as executable, the average Shannon entropy) of each one and the average ratio of their size vs their virtual size - these datapoints will tell the model if and how the PE is packed/compressed/obfuscated and hence are encoded.  

Last, we glue all the pieces into one single vector of size 486.


The only thing left to do, is telling our model how to encode the input samples by the **prepare_input function** in the **prepare.py** file descibed below:

It function does the encoding of a file given its path, given its contents (sent as a file upload to the ergo API), or just the evaluation on a raw vector of scalar features.

Now the dataset can be encoded using   

```ergo encode /dataset path --output /path/to/dataset.csv```    

**Note: Feature extracted Dataset is in the release section.**    

 ## Training the Model  
The model we’re using is a computational structure called Artificial neural network that we’re training using the Adam optimization algorithm :
>An ANN is a “box” containing hundreds of numerical parameters (the “weights” of the “neurons”, organized in layers) that are multiplied with the inputs (our vectors) and combined to produce an output prediction. The training process consists in feeding the system with the dataset, checking the predictions against the known labels, changing those parameters by a small amount, observing if and how those changes affected the model accuracy and repeating this process for a given number of times (epochs) until the overall performance has reached what we defined as the required minimum.

The main assumption is that there is a numerical correlation among the datapoints in our dataset that we don’t know about but that if known would allow us to divide that dataset into the output classes. What we do is asking this blackbox to ingest the dataset and approximate such function by iteratively tweaking its internal parameters.

The **model.py** file contains definition of the ANN which is a fully connected network with two hidden layers of 70 neurons each, ReLU as the activation function and a dropout of 30% during training.  
The training process can be started using   

```ergo train /project path --dataset /path/to/dataset.csv```  

Once done, model's performance statistics can be viewed with:

```ergo view /project path```

This will show the training history, where one can verify that the model accuracy indeed increased over time and the ROC curve, which tells us how effectively the model can distinguish between malicious or not.  

Moreover, a confusion matrix for each of the training, validation and test sets will also be shown. The diagonal values from the top left (dark red) represent the number of correct predictions, while the other values (pink) are the wrong ones

# Results, Conclusion and Future Scope
97% accuracy on such a big dataset is a very interesting result considering how simple our features extraction algorithm is. Many of the misdetections are caused by packers such as UPX (or even just self extracting zip/msi archives) that affect some of the datapoints we’re encoding - adding an unpacking strategy (such as emulating the unpacking stub until the real PE is in memory) and more features such as bigger entrypoint vector, dynamic analysis to trace the API being called is the key to get it to 99% 

# Usage as an API  
Load the model and use it as an API:

```ergo serve /project path --classes "clean, malicious" ```  

And request its classification from a client:

```curl -F "x=@/path/to/file.exe" "http://localhost:8080/" ```  

*Useful links*

- Library to Instrument Executable Formats : https://lief.quarkslab.com/
- Ergo-ai: https://pypi.org/project/ergo-ai/
