# Introduction #

The public AWS4C functions are divided into four groups:

  * [Configuration Functions](APIRef#Configuration_Functions.md)
  * [S3 Interface Functions](APIRef#S3_Interface_Functions.md)
  * [SQS Interface Functions](APIRef#SQS_Interface_Functions.md)
  * [I/O Buffer Functions](APIRef#IO_Buffer_Functions.md)




## Configuration Functions ##

### aws\_init ###
_void aws\_init  (  )_

Initialize the library. This function must be the first AWS4C function called.

### aws\_read\_config ###

_int aws\_read\_config 	( char `*` const **id**  )_

Read AWS authentication records.

Parameters:
  * **id**  - user ID


This function loads AWS authentication credentials corresponding to the user identified by the user ID from ~/.awsAuth file.   All subsequent S3 and SQS operations will use these credentials.
This function must be called before the first SQS or S3 operation.   It can be called again when the credential switch is required.

See the [Developer's Guide](DevGuide#Authentication_Keys.md) for details about the `~/.awsAuth` file.

### aws\_set\_debug ###

_void aws\_set\_debug 	( int 	**d**  )_


Enable debugging output.

Parameters:
  * **d** -  non-zero parameter causes debugging output to be printed

This function controls whether debugging output is printed out. Calling it with a non-zero parameter results in the printing of the debug messages.

### aws\_set\_rrs ###

_void aws\_set\_rrs 	( int 	**rrs**  )_

Enable Reduced Redundancy Storage

Parameters:
  * **rrs** -  non-zero parameter causes all subsequent PUTs to use RRS


This function controls whether normal or reduced redundancy storage class is used for the put operations.  Calling this function with non-zero parameter will force all subsequent PUT operations to use reduced redundancy storage.  Reduced redundancy objects are not replicated as many times as the standard S3 objects,  hence they are slightly cheaper to store.




### aws\_set\_id ###

### aws\_set\_key ###

### aws\_set\_keyid ###



## IO Buffer Functions ##

### IOBuf ###

IOBuf is a data structure that holds the data to be sent to the AWS service or results of AWS operation. The structure has the following fields of interest:

  * _int code_  - operation result code. Standard HTTP result codes
  * _char `*` result_   -  textual representation of HTTP result code such as "404 Not found"
  * _char `*` lastMod_  -  date and time of the last modification applied to the object
  * _char `*` eTag_  - object signature eTag
  * _int contentLen_ - length of the object returned by the service
  * _int len_  - actual length of data retrieved from the service



### aws\_iobuf\_new ###

_IOBuf`*` aws\_iobuf\_new (  )_

Create a new I/O buffer.

Returns:
> a newly allocated I/O buffer

### aws\_iobuf\_append ###

_void aws\_iobuf\_append  (  IOBuf `*`  **B**, char `*` **d**, int **len** )_

Append data to I/O buffer.

Parameters:
  * **B**  - I/O buffer
  * **d**  - pointer to the data to be appended
  * **len** -  length of the data to be appended


### aws\_iobuf\_getline ###

_int aws\_iobuf\_getline 	( IOBuf `*`  **B**, char `*`  **Line**, int  **size**)_

Read the next line from the buffer.

Parameters:
  * **B**  - I/O buffer
  * **Line** - character array to store the next line from the buffer
  * **size** - size of the character array line

Returns:
> Number of characters read or 0

### aws\_iobuf\_free ###

_void aws\_iobuf\_free 	( IOBuf `*`  **bf** )_

Release IO Buffer.

Parameters:
  * **bf** - I/O buffer to be deleted


## S3 Interface Functions ##

## s3\_set\_host ##

_void s3\_set\_host 	(    char `*`const    **str** 	 )_

Sets S3 host.  The AWS4C sets the default host to ‘s3.amazonaws.com`;  you only need to call this function if you want to use a different Simple Storage Services host.

Parameters:

  * **str** -  hostname

## s3\_set\_bucket ##

_void s3\_set\_bucket 	( 	char `*`const  **str** 	 )_


Select current S3 bucket.  All subsequent S3 operations will be performed on this bucket.

Parameters:
  * **str**  - bucket ID

## s3\_set\_mime ##

_void s3\_set\_mime ( char `*` const str )_

Set the mime type for the S3 put operation.  All subsequent S3 PUT operations will use this mime type.

## s3\_set\_acl ##

Set access control policy for the S3 put operations. The access control policy should be one of the Amazon's canned access control policies such as 'public-read', 'public-read-write' or 'authenticated-read'

## s3\_get ##

_int s3\_get 	( IOBuf `*` **b**,  char `*` const  **file**)_

Downloads the  file from the current bucket and  places the data into the I/O buffer.

Parameters:
  * **b** - I/O buffer
  * **file** - filename

Returns:
> 0 on success or an error code on failure.  Also check the return code in I/O buffer. For more information see [Checking results of AWS operation](DevGuide#Checking_results_of_AWS_operation.md).

## s3\_get ##

_int s3\_delete 	( IOBuf `*` **b**,  char `*` const  **file**)_


Deletes the specified file from the current bucket.


Parameters:
  * **b** - I/O buffer
  * **file** - filename

Returns:
> 0 on success or an error code on failure.  Also check the return code in I/O buffer.  If operation was performed successfully the HTTP return code should be "204 No Content".    For more information see [Checking results of AWS operation](DevGuide#Checking_results_of_AWS_operation.md).

Note:   If the specified file doesn't not exist in the bucket the operation still will complete successfully with the return code 0 and HTTP return code 204.




## s3\_put ##

_int s3\_put   ( IOBuf `*` **b**, char `*` const   **file** )_

Uploads the file into currently selected bucket and takes the data from I/O buffer.

Parameters:
  * **b**  - I/O buffer
  * **file** - filename

Returns:
> 0 on success or an error code on failure.  Also check the return code in I/O buffer. For more information see  [Checking results of AWS operation](DevGuide#Checking_results_of_AWS_operation.md).



## SQS Interface Functions ##

## sqs\_create\_queue ##

_int sqs\_create\_queue 	( IOBuf `*` **b**, char `*` const **name** )_

Create an SQS queue.

Parameters:
  * **b** -	I/O buffer
  * **name**  -	queue name

Returns:
> 0 on success, or an error code on failure

## sqs\_list\_queues ##

**int sqs\_list\_queues 	( IOBuf b**,char `*` const**prefix_)_**

Retrieves URL of the queue.

Parameters:
  * **b** -	I/O buffer
  * **prefix**  -	queue prefix. better use the whole name

Returns:
> 0 on success, or an error code on failure

URL is placed into the I/O buffer. Use [aws\_iobuf\_get\_line](APIREF#aws_iobuf_get_line.md) to retrieve it


## sqs\_get\_queueattributes ##

int sqs\_get\_queueattributes 	(  IOBuf `*`  **b**,   char `*`  url,  int `*`  **timeOut**,  int `*`  **nMesg** )

Retrieve queue’s default visibility timeout and approximate number of messages.

Parameters:
  * **b** -	I/O buffer
  * **url** -	queue url. Use sqs\_list\_queues to retrieve
  * **timeOut** -  points to a variable that will get the value of queue visibility timeout
  * **nMesg**  -   points to a variable that will get the value of approximate number of messages in the queue

Returns:
> 0 on success, or an error code on failure

> Note that given the distributed nature of the SQS service it is not possible to get the exact number of messages in the queue.  This function tries its best to get an approximate number, which is good enough for capacity estimation.


## sqs\_set\_queuevisibilitytimeout ##

_int sqs\_set\_queuevisibilitytimeout 	(  IOBuf `*` **b**, char `*` **url**, int  sec )_


Set the queue's visibility timeout.

Parameters:
  * **b** -	I/O buffer
  * **url** -	queue url. Use sqs\_list\_queues to retrieve
  * **sec** -	queue visibility timeout in seconds

Returns:
> 0 on success, or an error code on failure




## sqs\_send\_message ##

_int sqs\_send\_message  (  IOBuf `*`  **b**, char `*` const  **url**, char `*`const **msg** )_

Send a message to the queue.

Parameters:
  * **b** -	I/O buffer
  * **url** -	queue url. Use sqs\_list\_queues to retrieve
  * **msg** -	a message to send

Returns:
> 0 on success, or an error code on failure


## sqs\_get\_message ##

**int sqs\_get\_message ( IOBuf `*` **b**, char **const**url**, char `*`**id_)_**

Retrieve a message from the queue.

Parameters:
  * **b** 	- I/O buffer
  * **url** - queue url. Use sqs\_list\_queues to retrieve
  * **id** - message receipt handle.

Returns:
> 0 on success, or an error code on failure

Message contents are placed into the I/O buffer Caller must allocate at least 1024 bytes of  memory for the receipt.

## sqs\_delete\_message ##

_int sqs\_delete\_message ( IOBuf `*` **bf**, char `*`const **url**, char `*`  **receipt** )_

Delete a processed message from the queue.

Parameters:
  * **bf** - I/O buffer
  * **url** - queue url. Use sqs\_list\_queues to retrieve
  * **receipt** - message receipt handle.

Returns:
> 0 on success, or an error code on failure