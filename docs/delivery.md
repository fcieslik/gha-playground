# Delivery architecture — private CloudFront origin

After-state: CloudFront is the only public entry point; the S3 bucket is a
private origin that only CloudFront may read.

```
                          deploy identity (IAM user from lesson 7)
                          PutObject / DeleteObject
                                     │
                                     ▼
 viewer ──HTTPS──> CloudFront ──signed HTTPS (OAC)──> private S3 bucket
        (redirect        (public          origin request     my-frontend-app-assets-1
         HTTP→HTTPS)      entry point)     signed by OAC       Block Public Access = ON

 direct request to  https://my-frontend-app-assets-1.s3.us-east-1.amazonaws.com/index.html
   └──> 403 AccessDenied   (bucket policy allows only CloudFront)
```

## Annotations

- **Public entry point:** the CloudFront distribution domain, served over
  HTTPS (viewer protocol redirects HTTP to HTTPS).
- **Private origin:** the S3 bucket `my-frontend-app-assets-1` used as a
  normal S3 REST origin (not the website endpoint). Block Public Access stays on.
- **Deployment identity:** the existing IAM user from lesson 7. It keeps only
  `PutObject`/`DeleteObject` to push the tested `site` artifact. Unchanged.
- **OAC identity:** Origin Access Control attached to the origin, set to
  "Sign requests". CloudFront signs every request it makes to S3.
- **Bucket-policy condition:** allow `s3:GetObject` to principal
  `cloudfront.amazonaws.com` only when
  `AWS:SourceArn == arn:aws:cloudfront::427047664719:distribution/<DISTRIBUTION_ID>`.
- **Direct S3 object request:** returns **403 AccessDenied** — the bucket is
  no longer publicly readable; CloudFront is the only allowed reader.
