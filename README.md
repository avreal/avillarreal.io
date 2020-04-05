# avillarreal.io

Creating a static website/blog to play around with some aws services. 

The site is hosted on S3 using the static site feature, served by cloudfront, uses route53 for DNS, and has an SSL certificate created in ACM.

The site itself was generated with hugo using the kiera theme, I have made a number of modifications to the templates to suit my needs and tastes.

TODO:

Write script that will update the necessary files in the S3 bucket and invalidate the objects in cloudfront when new files are added to the site.
