---
layout: post
title:  "Directly Uploading to S3 from iPhone"
---

This post is my reply to [Architectural and design question about uploading photos from iPhone app and S3](http://stackoverflow.com/questions/4481311/architectural-and-design-question-about-uploading-photos-from-iphone-app-and-s3) in blog form. The issue addressed is how to upload a file from the iPhone directly to S3 without potentially exposing your S3 secret key to jailbroken users, which could then add or remove keys or entire buckets.

You can upload directly from your iPhone to S3 using the REST API, and having the server be responsible for generating the part of the `Authorization` header value that requires the secret key. This way, you don't risk exposing the access key to anyone with a jailbroken iPhone, while you don't put the burden of uploading the file on the server. The exact details of the request to make can be found under ["Example Object PUT" of "Signing and Authenticating REST Requests](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html). I **strongly recommend** reading that document before proceeding any further.

The following code, written in Python, generates the part of the `Authorization` header value that is derived from your S3 secret access key. You should substitute your own secret access key and [bucket name in virtual host form](http://docs.aws.amazon.com/AmazonS3/latest/dev/VirtualHosting.html) for `_S3_SECRET` and `_S3_BUCKET_NAME` below, respectively:

{% highlight python %}
import base64
from datetime import datetime
import hmac
import sha

# Replace these values.
_S3_SECRET = "my_s3_secret"
_S3_BUCKET_NAME = "my-bucket-name"

def get_upload_header_values(content_type, filename): 
	now = datetime.utcnow()
	date_string = now.strftime("%a, %d %b %Y %H:%M:%S +0000")
	full_pathname = '/%s/%s' % (_S3_BUCKET_NAME, filename)
	string_to_sign = "PUT\n\n%s\n%s\n%s" % (
			content_type, date_string, full_pathname)
	h = hmac.new(_S3_SECRET, string_to_sign, sha)
	auth_string = base64.encodestring(h.digest()).strip()
	return (date_string, auth_string)
{% endhighlight %}

Calling this with the filename `foo.txt` and content-type `text/plain` yields:

{% highlight python %}
>>> get_upload_header_values('text/plain', 'foo.txt')
('Wed, 06 Feb 2013 00:57:45 +0000', 'EUSj3g70aEsEqSyPT/GojZmY8eI=')
{% endhighlight %}

Note that if you run this code, the time returned will be different, and so the encoded HMAC digest will be different.

Now the iPhone client just has to issue a PUT request to S3 using the returned date and HMAC digest. Assuming that

* the server returns `date_string` and `auth_string` above in some JSON object named `serverJson`
* your S3 Access Key (not your secret, which is only on the server) is named `kS3AccessKey`
* your S3 bucket name (set to `my-bucket-name` above) is named `kS3BucketName`
* the file contents are marshalled in an `NSData` object named `data`
* the filename that was sent to the server is a string named `filename`
* the content-type that was sent to the server is a string named `contentType`

Then you can do the following to create the `NSURLRequest`:

{% highlight objc %}
NSString *serverDate = [serverJson objectForKey:@"date"]
NSString *serverHmacDigest = [serverJson objectForKey:@"hmacDigest"]

// Create the headers.
NSMutableDictionary *headers = [[NSMutableDictionary alloc] init];
[headers setObject:contentType forKey:@"Content-Type"];
NSString *host = [NSString stringWithFormat:@"%@.s3.amazonaws.com",
                  kS3BucketName]
[headers setObject:host forKey:@"Host"];
[headers setObject:serverDate forKey:@"Date"];
NSString *authorization = [NSString stringWithFormat:@"AWS %@:%@",
                           kS3AccessKey,
                           serverHmacDigest];
[headers setObject:authorization forKey:@"Authorization"];

// Create the request.
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] init];
[request setAllHTTPHeaderFields:headers];
[request setHTTPBody:data];
[request setHTTPMethod:@"PUT"];
NSString *postUrl = [NSString stringWithFormat:@"http://%@.s3.amazonaws.com/%@",
                     kS3BucketName,
                     filename];
[request setURL:[NSURL URLWithString:postUrl]];
{% endhighlight %}

Next you can issue the request. If you're using the excellent [AFNetworking library](http://afnetworking.com/), then you can wrap `request` in an `AFXMLRequestOperation` object using [`XMLDocumentRequestOperationWithRequest:success:failure:`](http://afnetworking.github.com/AFNetworking/Classes/AFXMLRequestOperation.html#//api/name/XMLDocumentRequestOperationWithRequest:success:failure:), and then invoking its `start` method. Don't forget to release `headers` and `request` when done.

Note that the client got the value of the `Date` header from the server. This is because, as Amazon describes under ["Time Stamp Requirement"](http://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html):

> "A valid time stamp (using either the HTTP Date header or an x-amz-date alternative) is mandatory for authenticated requests. Furthermore, the client time-stamp included with an authenticated request must be within 15 minutes of the Amazon S3 system time when the request is received. If not, the request will fail with the RequestTimeTooSkewed error status code."

So instead of relying on the client having the correct time in order for the request to succeed, rely on the server, which should be using NTP (and a daemon like `ntpd`).

