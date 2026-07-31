---
title: "Cleanup"
date: 2026-07-31
weight: 14
chapter: false
pre: " <b> 5.14. </b> "
---

After the workshop is complete, the AWS resources created for CloudHop RAG should be removed when they are no longer needed. Cleaning up the deployment avoids leaving unused compute, storage, networking, and application resources active in the AWS account.

Resources should be removed in an order that avoids leaving dependent components behind.

{{% notice warning %}}
Deletion is permanent. If you may need to demonstrate the system again, use the **pause** option in section 3 instead - it removes almost all of the running cost without destroying anything.
{{% /notice %}}

---

## 1. Order of removal

Work top-down: the things that call other things first, the things that store data last.

| # | Resource | Console | Note |
| --- | --- | --- | --- |
| 1 | **Amplify app** | Amplify → App settings → General → **Delete app** | Removes hosting, build history and the `amplifyapp.com` URL |
| 2 | **API Gateway API** | API Gateway → select API → **Delete** | Removes all routes, integrations and the Invoke URL |
| 3 | **EC2 instance** | EC2 → Instances → Instance state → **Terminate** | The attached EBS root volume is deleted with it by default - verify |
| 4 | **Elastic IP** | EC2 → Elastic IP addresses → **Release** | See the warning below |
| 5 | **S3 Vectors index** | S3 → Vector buckets → your bucket → delete the index | Must go before the bucket |
| 6 | **S3 Vectors bucket** | S3 → Vector buckets → **Delete** | Only deletes once empty |
| 7 | **S3 objects, then bucket** | S3 → your bucket → empty, then delete | See the versioning note below |
| 8 | **IAM role** | IAM → Roles → `rag-ec2-runtime-role` → **Delete** | Only after the instance is gone |
| 9 | **Secrets Manager secret** | Secrets Manager → your secret → **Delete** | Has a recovery window - see below |
| 10 | **SSM parameters** | Systems Manager → Parameter Store | Only if you created any. This deployment configures the service through `.env.prod`, so there are none |

{{% notice warning %}}
**Releasing the Elastic IP is the step people forget, and it is the one that keeps charging.** Terminating the instance only *disassociates* the address; the allocation survives and public IPv4 addresses are billed hourly whether attached or not. Go to **EC2 → Elastic IP addresses** and confirm the list is empty.
{{% /notice %}}

**Versioned bucket.** The artifact bucket has versioning enabled (chapter 5.5), so "Empty" must delete **all object versions and delete markers** - otherwise the bucket refuses to delete and reports that it is not empty. In the Console, use **Empty**, then confirm; with the CLI, `aws s3 rb --force` does not remove old versions, so empty it explicitly first.

**Secret deletion.** Secrets Manager schedules deletion with a recovery window of 7–30 days rather than deleting immediately, and it bills for the secret during that window. To remove it now:

```bash
aws secretsmanager delete-secret \
  --region ap-southeast-1 \
  --secret-id /prod/aws-rag/groq-api-key \
  --force-delete-without-recovery
```

Also **revoke the Groq API key** in the Groq console. Deleting the AWS secret removes the copy, not the key itself.

---

## 2. CLI equivalents

```bash
# 3-4. EC2 and the Elastic IP
aws ec2 terminate-instances --instance-ids <instance-id> --region ap-southeast-1
aws ec2 release-address --allocation-id <allocation-id> --region ap-southeast-1

# 5-6. S3 Vectors
aws s3vectors delete-index --vector-bucket-name <vector-bucket> --index-name <index-id> --region ap-southeast-1
aws s3vectors delete-vector-bucket --vector-bucket-name <vector-bucket> --region ap-southeast-1

# 7. Regular S3 (empties all versions, then removes the bucket)
aws s3 rm s3://<artifact-bucket> --recursive --region ap-southeast-1
aws s3 rb s3://<artifact-bucket> --region ap-southeast-1

# 9. The secret (immediate deletion, no recovery window)
aws secretsmanager delete-secret --region ap-southeast-1 \
  --secret-id <your-secret-name> --force-delete-without-recovery
```

---

## 3. Pausing instead of deleting

If the project may still be demonstrated, stop rather than destroy:

| Action | Effect |
| --- | --- |
| **Stop** the EC2 instance | Removes the largest recurring charge |
| Keep S3, S3 Vectors and Secrets Manager | Negligible cost at this data size |
| Keep Amplify and API Gateway | Billed per use; nothing to pay while idle |

What still bills while stopped: the **EBS root volume** and the **Elastic IP**. If a pause will last weeks rather than days, releasing the Elastic IP is worth it - you will need to update the API Gateway integrations with the new address when you return (chapter 5.8 section 2).

Restarting is the daily checklist in chapter 5.7 section 12: start the instance, connect with Session Manager, check `/health`, warm up.

---

## 4. Verify nothing was left behind

Check each of these in the Console, in Region `ap-southeast-1`:

| Check | Expected |
| --- | --- |
| EC2 → Instances | No running or stopped project instance |
| EC2 → **Elastic IP addresses** | Empty |
| EC2 → **Volumes** | No orphaned volume from the terminated instance |
| S3 → Buckets | Project bucket gone |
| S3 → **Vector buckets** | Vector bucket gone |
| API Gateway → APIs | Project API gone |
| Amplify → Apps | App gone |
| IAM → Roles | `rag-ec2-runtime-role` gone |
| Secrets Manager | Secret gone or scheduled for deletion |
| Parameter Store | Empty, if you ever created parameters |

{{% notice tip %}}
Two of these rows exist because they are the usual culprits. **Elastic IP addresses** and **Volumes** are billed independently of the instance they used to belong to, and neither appears in the EC2 instance list - so a cleanup that only checks "is the instance gone" leaves them running. Check both explicitly.
{{% /notice %}}

Finally, look at **Billing → Cost Explorer** a day or two later. Charges are reported with a lag, so a clean Console does not immediately show a clean bill; a near-zero daily figure after 48 hours is the real confirmation.

---

## 5. What to keep

Deleting the AWS resources does not delete the work. These survive and are what make the deployment reproducible:

- the **source repository** - backend, frontend and the offline build notebook;
- the **offline artifact folder** if you kept a copy off AWS, which lets you rebuild the index without re-embedding the corpus;
- **this workshop**, which is the record of how every resource was created.

Rebuilding from scratch means running chapters 5.4 through 5.9 again - roughly an afternoon, and no step depends on anything that was deleted.
