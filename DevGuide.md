# Prerequisites #

> AWS4C depends on the following prerequisites:

  * CURL (http://curl.haxx.se/) --  for network communication
  * OpenSSL (http://www.openssl.org/) --  for security and cryptography

These prerequisites must be installed before you attempt to build the examples or integrate the library into your project.  Fortunately these libraries are already supplied by most operating systems either as a part of the default install or as a standard add-on package.

# Platform #
The AWS4C had been tested with Mac OS Leopard, Linux and Cygwin. The GCC 3.3.4 was used on all of these platforms.


# Authentication Keys #

AWS4C uses the `~/.awsAuth` file for retrieving AWS credentials. (This file is formatted similarly to a UNIX `passwd` file.)   Each line of the file should contain the following items,  a user ID,  the AWS Key ID, and the AWS Key. These items should be separated by a colon `:`.  In the first item of each line, the user ID parameter is a record identifier.  The record identifier is used by the [aws\_read\_config](APIRef#aws_read_config.md) function to the AWS key  id and the value to be used  by the library.  The sample code supplied with this library uses the user ID `sample`  Make sure to configure this ID in your `.awsAuth` file before running the  example programs.

Note the `.awsAuth` file should be owned by the user who is executing the  program and should have permissions such that it is only readable by the owner.

Here is an example of the file:

```
id:1XASDSDSDSAMPLE:IK234jJk3454543SAMPLE
user2:1X35D84SD9AMPLE:IK234jJk3454543SAMPLE
sample:1XA39S3xwSAMPLE:IK234jJk3454543SAMPLE
```


You don't have to rely on this facility to use the AWS4C library; if you have your own secure mechanism for storing the AWS credentials, you can use [aws\_set\_key](APIRef#aws_set_key.md) and [aws\_set\_keyid](APIRef#aws_set_keyid.md) functions to set the AWS credentials.  Keep in mind that the practices of hard-coding these values into the code recommended by some are terribly insecure.

# Integrating into your Project #

Integrating the AWS4C into your project is very simple.  First you should copy the aws4c.c and aws4c.h files to a folder where your build script and the compiler can find them.

Next, you should edit the build script to include aws4c.o to the list of your object modules, and make sure that this module is created during the project build.

Finally, you should make sure that you link to the AWS4C pre-requisite libraries OpenSSL  (libcrypto) and Curl  (libcurl).

# Memory Ownership #

AWS4C automatically handles its own memory allocations.  If AWS4C requires that a data object last across several calls it copies  the object into its own local buffer.  It is therefore completely safe to use local stack variables in any call.

The only exception to this rule is I/O buffers.  Since it is up to your program to decide the lifetime of the I/O buffer, you should allocate and free them as needed.   Keep in mind that I/O buffers are not reusable;  After  an I/O buffer was used by an AWS4C operation, it is no longer valid.

# IO Buffers #

The AWS4C implements I/O buffers to handle input and output of large objects.  For example  I/O buffers are used to put large files into S3 storage, and to subsequently retrieve these files to be processed by your application.   The amount of data that can be appended to the buffer is limited by the available memory and limitations of the AWS service.

In addition to data buffering capabilities I/O buffers store the AWS operation results such as HTTP result codes.  The result codes are stored in the respective ‘code’ and ‘result’ fields.  The ‘code’ field contains an integer HTTP result code. A code value of 200 indicates a successful operation, while other values indicate some kind of error.  Refer to  [Error Response Codes](HTTP.md) for more information.  The ‘result’ field contains a response string from the server.

Besides the result codes the I/O buffer type contains the request's metadata.  This metadata includes the ‘len’, ‘etag’ and ‘lastMod’ fields.  The ‘len’ field contains the total length of the data in the buffer.  The ‘eTag` and ‘lastMod` fields contain the  information that will help you identify if the object you just requested is the same as the one you requested before.


Here is a typical program flow for sending the data to the S3

  * Allocate a new I/O buffer by calling [aws\_iobuf\_new](APIRef#aws_iobuf_new.md)
  * Prepare the data to be submitted by repeatedly calling [aws\_iobuf\_append](APIRef#aws_iobuf_append.md).
  * Submit the data to AWS S3 by  using [s3\_put](APIRef#s3_put.md)
  * Examine results of the operation
  * Free the I/O buffer by calling [aws\_iobuf\_free](APIRef#aws_iobuf_free.md)


The workflow for reading the data from S3  is very similar:

  * Allocate a new I/O buffer by calling [aws\_iobuf\_new](APIRef#aws_iobuf_new.md)
  * Request the data  the data to AWS S3 by  using [s3\_get](APIRef#s3_get.md)
  * Read the data by from the buffer by using  [aws\_iobuf\_getline](APIRef#aws_iobuf_getline.md)
  * Examine results of the operation
  * Free the I/O buffer by calling [aws\_iobuf\_free](APIRef#aws_iobuf_free.md)

Note that I/O buffers are not reusable.  Once you start reading the data from the buffer the additional data can not be appended.

## Checking Results of AWS Operations ##

When AWS operations returns 0 it means that there were no problems communicating with AWS, however it doesn’t mean that the operation succeeded. To verify the result of the operation check the ‘code’ field of the buffer.  If the operation was successful the code will be equal to ‘200’,  other values indicate an error condition.  To get the details of an error condition check the ‘result’ field and the contents of I/O buffer.

Here is the example of checking an error condition.

```
	rv = sqs_set_queuevisibilitytimeout ( bf, queueURL , 16 );
	if ( rv || bf->code != 200 )
         { 
           while(-1)
           {	
              char Ln[1024];
              int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
              if ( Ln[0] == 0 ) break;
              puts ( Ln );
           }
	 }
```



# Examples and Walk-throughs #

For many programmers the easiest way to understand the workings of the AWS4C library is through following examples that use the various functions of the library.   After each example below we will have a line-by-line explanation of what function each of the code fragment performs.


## Putting a File Into S3 ##


The following example is a simple program that writes a file into an S3 bucket.

```
00001 #include <stdio.h>
00002 #include <getopt.h>
00003 #include <stdlib.h>
00004 #include <string.h>
00005 #include "aws4c.h"
00006 
00007 
00008 
00009 
00010 int main ( int argc, char * argv[] )
00011 {
00012   puts ( "Curl submitter" );
00013 
00014   aws_init ();
00015   aws_set_debug  ( 0 );
00016   int rc = aws_read_config  ( "id" );
00017   if ( rc )
00018     {
00019       puts ( "Could not find a credential in the config file" );
00020       puts ( "Make sure your ~/.awsAuth file is correct" );
00021       exit ( 1 );
00022     }
00023 
00024   s3_set_bucket ("aws4c.samples");
00025 
00026 
00027 
00028   IOBuf * bf = aws_iobuf_new ();
00029 
00030 
00031   int i;
00032   for ( i = 0 ; i < 9000 ; i ++ )
00033     {
00034       char S[128];
00035       snprintf ( S,sizeof(S), "Ln %d \n" , i );
00036       aws_iobuf_append ( bf,S, strlen(S));
00037     }
00038 
00039 
00040   int rv = s3_put ( bf, "aws4c.samplefile" );
00041 
00042   printf ( "RV %d\n", rv );
00043 
00044   printf ( "CODE    [%d] \n", bf->code );
00045   printf ( "RESULT  [%s] \n", bf->result );
00046   printf ( "LEN     [%d] \n", bf->len
);
00047   printf ( "LASTMOD [%s] \n", bf->lastMod );
00048   printf ( "ETAG    [%s] \n", bf->eTag );
00049 
00050   while(-1)
00051     {
00052   char Ln[1024];
00053   int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00054   if ( Ln[0] == 0 ) break;
00055   printf ( "S[%3d] %s", sz, Ln );
00056     }
00057 
00058 
00059 }

```

Here is an explanation for the above code. Note that we will only discuss the lines that are relevant to AWS4C.

**Line 5**

```
00005 #include "aws4c.h"
```

Includes the AWS4C header file.

**Lines 14-15**

```
00014   aws_init ();
00015   aws_set_debug  ( 1 );
```

These lines initialize the AWS4 library and turn on verbose debugging output.

**Lines 16-22**

```
00016   int rc = aws_read_config  ( "id" );
00017   if ( rc )
00018     {
00019       puts ( "Could not find a credential in the config file" );
00020       puts ( "Make sure your ~/.awsAuth file is correct" );
00021       exit ( 1 );
00022     }
```

These lines read the credentials from the [~/.awsAuth](DevGuide#Auth.md) file and then check the result.  In the event of a credential reading failure. The error message is printed and the program is stopped.

**Line 24**

```
00024   s3_set_bucket ("aws4c.samples");
```

Selects the current bucket for the operations.  Note that in the current version of AWS4C does not provide capabilities for creating buckets.  After this function is called all subsequent AWS4C S3 operations will be performed in the context of this bucket.

**Line 28**

```

00028   IOBuf * bf = aws_iobuf_new ();

```

Creates a new [I/O buffer](DevGuide#IOBuffers.md).  This buffer will hold the data that we are going to send to the Amazon Simple Storage Service.

**Lines 31-37**

```
00031   int i;
00032   for ( i = 0 ; i < 9000 ; i ++ )
00033     {
00034       char S[128];
00035       snprintf ( S,sizeof(S), "Ln %d \n" , i );
00036       aws_iobuf_append ( bf,S, strlen(S));
00037     }
```

Prepares the data contents of the file to be sent to S3. This fragment places 9000 lines into the [I/O Buffer](DevGuide#IO_buffer.md) that was created on line 28.

**Line 40**

```
00040   int rv = s3_put ( bf, "aws4c.samplefile" );
```


Submits the contents of [I/O Buffer](DevGuide#IO_Buffer.md)  to the S3 service using the [s3\_put](APIRef#s3_put.md) function.  The contents will be placed into the object named aws4c.samplefile, that is located in the bucket selected on line 24.

**Lines 42-48**

```
00042   printf ( "RV %d\n", rv );
00043 
00044   printf ( "CODE    [%d] \n", bf->code );
00045   printf ( "RESULT  [%s] \n", bf->result );
00046   printf ( "LEN     [%d] \n", bf->len
);
00047   printf ( "LASTMOD [%s] \n", bf->lastMod );
00048   printf ( "ETAG    [%s] \n", bf->eTag );
```

Prints out the result and associated metadata of the `S3_Put` operation.  If the operation was successful the return code ‘rv’ should be equal to 0 and the I/O buffer HTTP code should be equal to 200.

**Lines 50-56**

```
00050   while(-1)
00051     {
00052   char Ln[1024];
00053   int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00054   if ( Ln[0] == 0 ) break;
00055   printf ( "S[%3d] %s", sz, Ln );
00056     }
```


Prints out the detailed S3 service response.  The response contains the request's metadata. If any errors occurred during the operation the response will contain a detailed explanation of the error.


## Retrieving a File From S3 ##

The following example reads the file from the S3 bucket.

```
00001 #include <stdio.h>
00002 #include <getopt.h>
00003 #include <stdlib.h>
00004 #include "aws4c.h"
00005 
00006 
00007 
00008 
00009 int main ( int argc, char * argv[] )
00010 {
00011   
00012 
00013   
00014 
00015   aws_init ();
00016   aws_set_debug ( 1 );
00017   int rc = aws_read_config  ( "sample" );
00018   if ( rc )
00019     {
00020       puts ( "Could not find a credential in the config file" );
00021       puts ( "Make sire your ~/.awsAuth file is correct" );
00022       exit ( 1 );
00023     }
00024 
00025   s3_set_host ( "s3.amazonaws.com");
00026   s3_set_bucket ("aws4c.samples");
00027 
00028 
00029   IOBuf * bf = aws_iobuf_new ();
00030   int rv = s3_get ( bf, "aws4c.samplefile" );
00031 
00032   printf ( "RV %d\n", rv );
00033 
00034   printf ( "CODE    [%d] \n", bf->code );
00035   printf ( "RESULT  [%s] \n", bf->result );
00036   printf ( "LEN     [%d] \n", bf->len );
00037   printf ( "LASTMOD [%s] \n", bf->lastMod );
00038   printf ( "ETAG    [%s] \n", bf->eTag );
00039 
00040   while(-1)
00041     {
00042   char Ln[1024];
00043   int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00044   if ( Ln[0] == 0 ) break;
00045   printf ( "S[%3d] %s", sz, Ln );
00046     }
00047 
00048   aws_iobuf_free ( bf );
00049 
00050 }

```


The explanation of the code is given below.

**Line 4**


```
00004 #include "aws4c.h"
```

Includes the AWS4C header file.

**Lines 15-16**

```
00015   aws_init ();
00016   aws_set_debug ( 1 );
```

Initializes the AWS4 library and turns on verbose debugging output.

**Lines 17-23**

```
00017   int rc = aws_read_config  ( "sample" );
00018   if ( rc )
00019     {
00020       puts ( "Could not found your credential in the the config file" );
00021       puts ( "Make your ~/.awsAuth file is correct" );
00022       exit ( 1 );
00023     }
```

Reads the credentials from the [~/.awsAuth](DevGuide#Auth.md) file and then checks the result.  In the event of credential reading failure, an error message is printed and the program is stopped.

**Lines 25-26**

```
00025   s3_set_host ( "s3.amazonaws.com");
00026   s3_set_bucket ("aws4c.samples");
```

Selects the S3 host and the current bucket for the operations.  Note that in the current version, AWS4C does not provide capabilities for creating buckets.  After this function is called, all subsequent AWS4C S3 operations will be performed in the context of this bucket and host.   You don’t need to call [s3\_set\_host](APIRef#s3_set_host.md) if your application uses the default S3 host ‘s3.amazonaws.com`.

**Line 29**

```
00029   IOBuf * bf = aws_iobuf_new ();
```

Creates a new [I/O buffer](DevGuide#IOBuffers.md).  This buffer will be a receptacle for the data that we will retrieve from the Amazon Simple Storage Service.

**Line 30**

```
00030   int rv = s3_get ( bf, "aws4c.samplefile" );
```

Requests the contents of the file ‘aws4c.samplefile` from the current S3 bucket. The contents of the file will be placed into the buffer allocated on the previous line.

**Lines 32-38**

```
00032   printf ( "RV %d\n", rv );
00033 
00034   printf ( "CODE    [%d] \n", bf->code );
00035   printf ( "RESULT  [%s] \n", bf->result );
00036   printf ( "LEN     [%d] \n", bf->len );
00037   printf ( "LASTMOD [%s] \n", bf->lastMod );
00038   printf ( "ETAG    [%s] \n", bf->eTag );
```

Prints out the result and associated metadata of the `S3_Put` operation.  If the operation was successful, the return code ‘rv` should be equal to 0 and the I/O buffer HTTP code of should be equal to 200.

**Lines 40-46**

```
00040   while(-1)
00041     {
00042   char Ln[1024];
00043   int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00044   if ( Ln[0] == 0 ) break;
00045   printf ( "S[%3d] %s", sz, Ln );
00046     }
```

Prints out the contents of the file received from the S3.   If the request for the file resulted in an error, this fragment will print out the detailed explanation of the error condition.

**Line 48**

```
00048   aws_iobuf_free ( bf );
```

Frees the allocated I/O Buffer.

## SQS example ##

The following example program illustrates the use of the SQS API provided by AWS4C.
The program creates the queue, and sets its visibility timeouts.  It then proceeds to place a number of messages into the queue and tries to retrieve them back. When each message is successfully retrieved the program deletes it from the queue.  The program will continue to retrieve messages until it detects that there are no none left in the queue. It then prints a message that the queue is empty and then shuts itself down.

The detailed line-by-line explanation of this program is given below.

```
00001 
00002 
00003 /// This is an example that shows how to use SQS functions of AWS4C. 
00004 /// The example performs the following actions
00005 ///
00006 ///    Initializes the library
00007 ///    Creates the queue
00008 ///    Obtains the full URL of the queue
00009 ///    Gets the queue parameters
00010 ///    Sets visibility timeout of the queue
00011 ///    Puts a number of messages into the queue
00012 ///    Reads a number of messages from the queue 
00013 ///    Delets the processed messages
00009 ///    Gets the queue parameters
00010 ///    Sets the visibility timeout of the queue
00011 ///    Puts a number of messages into the queue
00012 ///    Reads a number of messages from the queue 
00013 ///    Deletes the processed messages
00014 
00015 #include <stdio.h>
00016 #include <getopt.h>
00017 #include <stdlib.h>
00018 #include <string.h>
00019 #include "aws4c.h"
00020 
00021 
00022 /// This helper function that prints out a result of the operation
00023 /// and data returned by the server
00024 void DumpResult ( int rv, IOBuf * bf )
00025 {
00026   printf ( "RV %d\n", rv );
00027   printf ( "CODE [%d] \n", bf->code );
00028   printf ( "RESULT  [%s] \n", bf->result );
00029   printf ( "LEN     [%d] \n", bf->len );
00030   if ( bf->lastMod) printf ( "LASTMOD [%s] \n", bf->lastMod );
00031   if ( bf->eTag)    printf ( "ETAG    [%s] \n", bf->eTag );
00032 
00033   while(-1)
00034     {
00035       char Ln[1024];
00036       int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00037       if ( Ln[0] == 0 ) break;
00038       printf ( "S[%3d] %s", sz, Ln );
00039     }
00040 
00041 }
00042 
00043 
00044 int main ( int argc, char * argv[] )
00045 {
00046   char * queueURL;
00047 
00048   /// Initialize the library.
00049   aws_init ();
00050   aws_set_debug ( 0 );
00051   int rc = aws_read_config  ( "sample" );
00052   if ( rc )
00053     {
00054       puts ( "Could not find a credential in the the config file" );
00055       puts ( "Make sure your ~/.awsAuth file is correct" );
00056       exit ( 1 );
00057     }
00058  
00059 
00060 
00061 
00062 
00063   /// Create the queue
00064   IOBuf * bf = aws_iobuf_new ();
00065   int rv = sqs_create_queue ( bf, "AWS-SQS-Sample" );
00066   if ( rv || bf->code != 200 )
00067     { 
00068       /// Note that failure to create the queue suggests either a genuine
00069       /// error or simply that the queue already exists.
00070       /// For most applications, it is okay to continue even if this
00071       /// operation fails.
00072       puts ( "Failed to create queue\n" );
00073       DumpResult ( rv, bf ); 
00074     }
00075   aws_iobuf_free(bf);
00076 
00077 
00078 
00079   /// Get the URL of the queue.
00080   /// Most applications require the full URL of the queue that might be 
00081   /// different from the queue name passed to the SQSCreate queue
00082   bf = aws_iobuf_new ();
00083   rv = sqs_list_queues ( bf, "AWS-SQS-Sample" );
00084   if ( rv || bf->code != 200 )
00085     { 
00086       puts ( "Failed to retrieve list of queues.\n" );
00087       DumpResult ( rv, bf ); 
00088       exit(0); 
00089     }
00090 
00091   while(-1)
00092     {
00093       char Ln[1024];
00094       int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00095       if ( Ln[0] == 0 ) break;
00096       Ln[strlen(Ln)-1] = 0;
00097       printf ( "Queue: [%s]\n",  Ln );
00098       queueURL = strdup(Ln);
00099     }
00100 
00101   if ( queueURL == NULL ) 
00102      { printf ( "Could not find our queue" ); exit(0); }
00103 
00104   bf = aws_iobuf_new ();
00105   int timeOut, qLength;
00106   rv = sqs_get_queueattributes ( bf, queueURL , &timeOut, &qLength );
00107   if ( !rv && bf->code == 200 )
00108     {
00109       printf ( "Queue Timeout = %d  Approximate Length =  %d \n",  
00110                timeOut, qLength );
00111     }
00112   else
00113     {
00114       DumpResult(rv,bf); exit(0);
00115     }
00116     
00117   bf = aws_iobuf_new ();
00118   rv = sqs_set_queuevisibilitytimeout ( bf, queueURL , 16 );
00119   if ( rv || bf->code != 200 )
00120     {
00121       DumpResult(rv,bf); exit(0);
00122     }
00123 
00124   /// Send a few messages
00125   int i;
00126   for ( i = 0 ; i < 18 ; i ++ )
00127     {
00128       puts ( "Send message" );
00129       bf = aws_iobuf_new ();
00130       char Message[256];
00131       snprintf(Message,sizeof(Message),"Msg #%d  \n\n L=%d\n", i,i);
00132       rv = sqs_send_message ( bf, queueURL, Message );
00133       if ( rv || bf->code != 200 ) { DumpResult(rv,bf); exit(0); }
00134     }
00135   /// Retrieve messages.
00136   for ( i = 0 ; i < 500 ; i ++ )
00137   {
00138       puts ( "Get Message" );
00139       bf = aws_iobuf_new ();
00140       char receipt[1024];
00141       memset ( receipt, 0 , sizeof(receipt));
00142       rv = sqs_get_message ( bf, queueURL, receipt );
00143       DumpResult(rv,bf); 
00144       puts ("\n----");
00145       printf ( "ID: %s\n", receipt );
00146       aws_iobuf_free(bf);
00147 
00148       if ( receipt[0] != 0 )
00149         {
00150           bf = aws_iobuf_new ();
00151           rv = sqs_delete_message ( bf, queueURL, receipt );
00152           if ( rv || bf->code != 200 ) DumpResult(rv,bf); 
00153         }    
00154       else { puts ( "Empty queue" ); break; }
00155   }
00156   puts ("\n---");
00157 }

```

**Line 19**

```

00019 #include "aws4c.h"

```

Includes the AWS4C header file.

**Line 46**

```

00046   char * queueURL;

```

This variable will hold the pointer to the queue URL.  Most of the SQS operations require a specific queue URL, not just a generic queue name.

**Lines 49-50**

```

00049   aws_init ();
00050   aws_set_debug ( 0 );

```

Initializes the AWS4 library and turns on verbose debugging output.

**Lines 51-57**

```

00051   int rc = aws_read_config  ( "sample" );
00052   if ( rc )
00053     {
00054       puts ( "Could not find a credential in the the config file" );
00055       puts ( "Make sure your ~/.awsAuth file is correct" );
00056       exit ( 1 );
00057     }

```

Reads the credentials from the [~/.awsAuth](DevGuide#Auth.md) file and then checks the result.  In the event of credential reading failure, the error message is printed and the program is stopped.

**Lines 64-75**

```

00064   IOBuf * bf = aws_iobuf_new ();
00065   int rv = sqs_create_queue ( bf, "AWS-SQS-Sample" );
00066   if ( rv || bf->code != 200 )
00067     { 
00068       /// Note that failure to create the queue suggests either a genuine
00069       /// error or simply that the queue already exists.
00070       /// For most applications, it is okay to continue even if this
00071       /// operation fails.
00072       puts ( "Failed to create queue\n" );
00073       DumpResult ( rv, bf ); 
00074     }
00075   aws_iobuf_free(bf);

```

Creates a new [I/O Buffer](DevGuide#IO_Buffers.md)  and has the SQS server create a queue named `AWS-SQS-Sample`.   In the event of an error, this code prints out a diagnostic message, but doesn’t terminate the program.  Failure to create a queue might not prevent us from proceeding.   If the queue already exists from previously running this example, the [sqs\_create\_queue](APIRef#sqs_create_queue.md) will return an error condition, but the program can continue.   After the result of the operation is checked, the fragment releases the IO Buffer by calling [aws\_iobuf\_free](APIRef#aws_iobuf_free.md) function.

**Lines 82-83**

```

00082   bf = aws_iobuf_new ();
00083   rv = sqs_list_queues ( bf, "AWS-SQS-Sample" );

```

Allocates a new [I/O Buffer](DevGuide#IOBuffer.md)  and request a list of the queues starting with the `AWS-SQS-Sample`.  The purpose of this operation is to obtain a specific queue URL. As noted above, many SQS operation will require this URL.

**Lines 84-89**

```

00084   if ( rv || bf->code != 200 )
00085     { 
00086       puts ( "Failed to retrieve list of  queue\n" );
00087       DumpResult ( rv, bf ); 
00088       exit(0); 
00089     }

```


Checks the result of previous SQS operation.  If it detects a failure it prints out a detailed explanation and terminates the program.

**Lines 91-99**

```

00091   while(-1)
00092     {
00093       char Ln[1024];
00094       int sz = aws_iobuf_getline ( bf, Ln, sizeof(Ln));
00095       if ( Ln[0] == 0 ) break;
00096       Ln[strlen(Ln)-1] = 0;
00097       printf ( "Queue: [%s]\n",  Ln );
00098       queueURL = strdup(Ln);
00099     }

```

Reads the list of the queues returned by the [sqs\_list\_queues](APIRef#sqs_list_queues.md) operation.   Each of the queues is printed out, and the last retrieved queues is saved in the memory block referenced by the `queueURL`  variable.  This last retrieved queue will be used for all subsequent SQS operations.

**Lines 101-102**

```

00101   if ( queueURL == NULL ) 
00102      { printf ( "Could not find queue" ); exit(0); }

```

Checks if we actually retrieved any queue URLs.   In case we didn’t, the program can not continue.

**Lines 104-106**

```

00104   bf = aws_iobuf_new ();
00105   int timeOut, qLength;
00106   rv = sqs_get_queueattributes ( bf, queueURL , &timeOut, &qLength );

```

Allocates a new IO Buffer and then requests the number of messages in the queue and the queue’s default timeout value.

**Lines 107-115**

```

00107   if ( !rv && bf->code == 200 )
00108     {
00109       printf ( "Queue Timeout = %d  Approximate Length =  %d \n",  
00110                timeOut, qLength );
00111     }
00112   else
00113     {
00114       DumpResult(rv,bf); exit(0);
00115     }

```

Checks if the [sqs\_get\_attribtues](APIRef#sqs_get_attributes.md) operation was successfully completed.  In the event of failure, the fragment prints out a diagnostic messages and exits.  If the operation succeeded, the fragment prints out the queue’s timeout and the approximate length of the queue.

Note that given the distributed nature of the queue the returned number of messages is only an approximate number.

**Lines 117-118**

```

00117   bf = aws_iobuf_new ();
00118   rv = sqs_set_queuevisibilitytimeout ( bf, queueURL , 16 );

```

Sets the queue default visibility timeout to 16 seconds.

**Lines 126-134**

```

00126   for ( i = 0 ; i < 18 ; i ++ )
00127     {
00128       puts ( "Send message" );
00129       bf = aws_iobuf_new ();
00130       char Message[256];
00131       snprintf(Message,sizeof(Message),"Msg #%d  \n\n L=%d\n", i,i);
00132       rv = sqs_send_message ( bf, queueURL, Message );
00133       if ( rv || bf->code != 200 ) { DumpResult(rv,bf); exit(0); }
00134     }

```

Sends  a number of messages to the queue.  For each message the fragment allocates a new buffer,  prepares a message text,  and then places the message into the queue.  After placing the message the fragment checks the result code. In the event of failure, it prints out a diagnostic message and terminates the program.

**Lines 136-155**

```

00136   for ( i = 0 ; i < 500 ; i ++ )
00137   {
00138       puts ( "Get message" );
00139       bf = aws_iobuf_new ();
00140       char receipt[1024];
00141       memset ( receipt, 0 , sizeof(receipt));
00142       rv = sqs_get_message ( bf, queueURL, receipt );
00143       DumpResult(rv,bf); 
00144       puts ("\n----");
00145       printf ( "ID: %s\n", receipt );
00146       aws_iobuf_free(bf);
00147   
00148       if ( receipt[0] != 0 )
00149         {
00150           bf = aws_iobuf_new ();
00151           rv = sqs_delete_message ( bf, queueURL, receipt );
00152           if ( rv || bf->code != 200 ) DumpResult(rv,bf); 
00153         }    
00154       else { puts ( "Empty queue" ); break; }
00155    }

```

Attempts to retrieve a number of messages from the queue.  For each attempt, the fragment allocates a new IO Buffer,  prepares the receipt buffer, and then requests a message from SQS queue.   In the event of failure, the fragment prints out a diagnostic message and terminates the program.  Upon success the fragment deletes the message from the queue.  An empty receipt indicates that the queue doesn’t have anymore messages. In this case, the fragment terminates the loop.