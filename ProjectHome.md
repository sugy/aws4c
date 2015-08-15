Believe it or not there is a growing demand C/C++ programs to access Amazon Web Services.
This project grew out of that demand.  This project is based on the conversion of my old HPC project to run on the Amazon EC2.

To access the AWS services I needed a C. Unfortunately I couldn't find any, so I wrote my own.

The code quality is mediocre; I wrote the whole thing pretty much in a single day.  So, I wouldn't use it for anything more than a proof of concept projects unless it is heavily reworked to be more robust.  But nevertheless, it gets the job done.

The library includes bindings for SQS and S3.

It depends on libcurl for its network operations and openssl for the crypto.

Enjoy!!

P.S.  Just came across libAWS but it is heavily C++ dependent, and as a result so it would not work for all types of C-based projects.