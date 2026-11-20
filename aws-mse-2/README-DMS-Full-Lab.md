# AWS DMS MySQL Full-Load Lab — Deployment Guide

This bundle deploys the infrastructure described in the supplied lab guide in **one CloudFormation stack**. It also creates the IAM service role `dms-vpc-role` required by DMS for VPC networking.

## What the template creates

- VPC `training-vpc`, one public subnet and two private subnets in two AZs
- Internet Gateway, public/private route tables, optional NAT Gateway and Elastic IP
- Client, DMS, source DB, target DB and SSM endpoint security groups
- Private `ssm` and `ssmmessages` interface endpoints (optional parameter; enabled by default)
- EC2 `mysql-client` with an instance profile and `AmazonSSMManagedInstanceCore`
- RDS DB subnet group, source MySQL and target MySQL databases
- Target MySQL 8.0 parameter group with `local_infile=1`
- DMS replication subnet group, replication instance, source/target endpoints and a **Full load only** task
- Table mapping for all tables in schema `retaildb`

## Important scope boundary

CloudFormation creates the AWS infrastructure and DMS task definition. It does **not** execute SQL inside MySQL, test DMS endpoints, start the task, or verify row counts. Those steps are deliberately manual below so you can inspect each stage and diagnose failures. In particular, you must create the source `dms_user` before starting the task.

The template uses RDS-managed master passwords in AWS Secrets Manager for both RDS instances. It also creates the IAM service role `dms-vpc-role` with the AWS-managed `AmazonDMSVPCManagementRole` policy, which the DMS replication subnet group requires. The source DMS user password is supplied as a NoEcho CloudFormation parameter and must match the password you use in the SQL `CREATE USER` statement.

## Before deploying

1. Select **Asia Pacific (Mumbai) — ap-south-1**.
2. Check that `10.0.0.0/16` does not overlap with an existing VPC or connected network.
3. Check that the selected AZs are available to your account. Defaults are `ap-south-1a` and `ap-south-1b`.
4. Check service quotas and regional availability for `db.t3.micro`, `dms.t3.micro`, NAT Gateway, and interface endpoints.
5. This lab creates billable resources. NAT Gateway, DMS, RDS, EC2, EBS, and interface endpoints can incur charges while idle.

## Deploy using the AWS Console

1. Open **CloudFormation → Stacks → Create stack → With new resources (standard)**.
2. Upload `dms-full-lab.yaml`.
3. Stack name: `dms-mysql-full-load-lab`.
4. Set parameters:
   - `CreateNatGateway`: `true` (needed for private EC2 package downloads)
   - `CreateSsmEndpoints`: `true` (private Session Manager connectivity)
   - `SourceDmsUsername`: `dms_user`
   - `SourceDmsPassword`: enter a strong password and keep it safe; use the same value in the SQL step
   - Keep the small lab instance defaults unless your account does not support them.
5. Acknowledge named IAM resource creation if prompted, then create the stack. The template creates the account-level role name `dms-vpc-role`; if that role already exists in your account, do not deploy a duplicate. Verify the existing role has the `dms.amazonaws.com` trust relationship and the `AmazonDMSVPCManagementRole` managed policy, then remove the `DmsVpcRole` resource and its `DependsOn` from this template before deploying.
6. Wait until stack status is `CREATE_COMPLETE`. If it fails, inspect the first failed resource in the **Events** tab before retrying.

## Deploy using AWS CLI

Run from a terminal configured for the correct AWS account:

```bash
aws cloudformation validate-template \
  --template-body file://dms-full-lab.yaml \
  --region ap-south-1

aws cloudformation create-stack \
  --stack-name dms-mysql-full-load-lab \
  --template-body file://dms-full-lab.yaml \
  --parameters \
    ParameterKey=CreateNatGateway,ParameterValue=true \
    ParameterKey=CreateSsmEndpoints,ParameterValue=true \
    ParameterKey=SourceDmsUsername,ParameterValue=dms_user \
    ParameterKey=SourceDmsPassword,ParameterValue='REPLACE_WITH_YOUR_PASSWORD' \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-south-1

aws cloudformation wait stack-create-complete \
  --stack-name dms-mysql-full-load-lab \
  --region ap-south-1

aws cloudformation describe-stacks \
  --stack-name dms-mysql-full-load-lab \
  --region ap-south-1 \
  --query 'Stacks[0].Outputs'
```

Avoid putting real passwords in shell history or shared scripts. The Console parameter entry is preferable for a classroom lab. For production, use a stronger secret-injection workflow.

## Step 1 — Connect to the private EC2 client

1. Open **EC2 → Instances** and select `mysql-client`.
2. Choose **Connect → Session Manager → Connect**.
3. If the instance is offline, check the instance profile, SSM agent, VPC DNS attributes, endpoint status, endpoint SG inbound TCP 443, and outbound TCP 443 from the client SG.
4. Install the MySQL-compatible client:

```bash
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf install -y mariadb105
mysql --version
```

If the package name is unavailable, run `sudo dnf search mariadb`. If `dnf` times out, confirm NAT Gateway is `Available`, the public subnet has `0.0.0.0/0 → IGW`, and the private route table has `0.0.0.0/0 → NAT`.

## Step 2 — Retrieve RDS admin passwords

The stack outputs include `SourceMasterSecretArn` and `TargetMasterSecretArn`.

For each secret:
1. Open **AWS Secrets Manager** in `ap-south-1`.
2. Find the secret by its ARN.
3. Choose **Retrieve secret value**.
4. Copy the `password` value securely.

Both RDS master usernames are `admin`. Do not put these passwords in screenshots or source control.

## Step 3 — Create the source sample schema and rows

Use the source endpoint from CloudFormation output:

```bash
mysql -h SOURCE_RDS_ENDPOINT -P 3306 -u admin -p
```

At the MySQL prompt, run:

```sql
CREATE DATABASE IF NOT EXISTS retaildb;
USE retaildb;

CREATE TABLE IF NOT EXISTS customers (
  customer_id INT PRIMARY KEY,
  first_name VARCHAR(50),
  last_name VARCHAR(50),
  email VARCHAR(120)
);

INSERT INTO customers (customer_id, first_name, last_name, email)
VALUES
  (1, 'Asha', 'Sharma', 'asha@example.com'),
  (2, 'Rohan', 'Mehta', 'rohan@example.com'),
  (3, 'Priya', 'Verma', 'priya@example.com')
ON DUPLICATE KEY UPDATE
  first_name = VALUES(first_name),
  last_name = VALUES(last_name),
  email = VALUES(email);

SELECT * FROM customers;
SELECT COUNT(*) AS source_row_count FROM customers;
```

If you already have meaningful data in this table, inspect it first rather than replacing it.

## Step 4 — Create the dedicated source DMS user

Connect to the **source** database as `admin` and run the following. Replace the password with exactly the `SourceDmsPassword` value supplied to CloudFormation:

```sql
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'REPLACE_WITH_THE_SAME_PASSWORD';
GRANT SELECT ON retaildb.* TO 'dms_user'@'%';
SHOW GRANTS FOR 'dms_user'@'%';
```

If the user already exists, use `ALTER USER` to update its password instead of running `CREATE USER` again.

For this full-load-only task, `SELECT` is the key source permission. CDC replication privileges are not required.

## Step 5 — Verify target parameter and target DB access

Connect to the target endpoint:

```bash
mysql -h TARGET_RDS_ENDPOINT -P 3306 -u admin -p
```

At the MySQL prompt:

```sql
SHOW GLOBAL VARIABLES LIKE 'local_infile';
SHOW DATABASES;
USE retaildb;
SHOW TABLES;
```

Expected `local_infile` value: `ON` or `1`.

The target database `retaildb` is created by RDS. The DMS task uses `DO_NOTHING` target table preparation, so it does not drop existing target tables. Ensure the target `customers` table does not already contain conflicting data before starting the task.

The target endpoint uses the target RDS `admin` credentials retrieved from the target RDS-managed Secrets Manager secret. The RDS master account has the administrative privileges needed for this disposable lab. For production, create and use a dedicated least-privilege target account.

## Step 6 — Test DMS endpoints

1. Open **AWS DMS → Endpoints**.
2. Select `retail-source-endpoint` → **Actions → Test connection**.
3. Choose `retail-dms-replication` as the replication instance.
4. Repeat for `retail-target-endpoint`.
5. Both tests must show **successful** before starting the task.

If the source test fails, check the source DMS user/password and `SELECT` grant. If the target test fails, check the target secret, RDS status, and security group. Both RDS security groups allow TCP 3306 only from the client and DMS security groups.

## Step 7 — Review and start the Full load only task

The stack creates `retail-full-load-task` with:
- Migration type: `full-load`
- Selection rule: include schema `retaildb`, table `%`
- Target table preparation: `DO_NOTHING`
- No CDC / ongoing replication

Open **AWS DMS → Database migration tasks → retail-full-load-task**. Review its settings and table mappings, then choose **Actions → Start/Restart/Resume → Start** (wording may vary by console version). The task is not automatically started by CloudFormation.

## Step 8 — Monitor and verify

1. Open the task details and **Table statistics**.
2. Wait until the `customers` table reports a successful full load.
3. Inspect task logs and table-level errors if the task fails.
4. Connect to the target RDS instance and run:

```sql
USE retaildb;
SHOW TABLES;
SELECT COUNT(*) AS target_row_count FROM customers;
SELECT * FROM customers ORDER BY customer_id;
```

Compare `target_row_count` with the source count. Successful completion requires task/table success and matching source/target rows.

Do not expect later source inserts to appear on the target: this is intentionally **Full load only**, not CDC.

## Troubleshooting checklist

| Symptom | Check |
|---|---|
| Session Manager cannot connect | Instance profile, SSM agent, `ssm` and `ssmmessages` endpoints, private DNS, endpoint SG TCP 443, outbound client TCP 443 |
| `dnf` times out | NAT status, public IGW route, private NAT route, outbound rules |
| EC2 cannot connect to RDS | Correct endpoint, RDS status, DB SG inbound TCP 3306 from client SG |
| Source DMS endpoint test fails | Endpoint hostname, `dms_user` password, source `SELECT` grant, DMS SG inbound path on DB SG |
| Target DMS endpoint test fails | Target endpoint hostname, target RDS-managed secret, DMS SG inbound path on DB SG |
| `local_infile` is OFF | Target parameter group attached, parameter saved, pending-reboot status; reboot if AWS reports it is required |
| No tables migrate | `retaildb` schema and source rows exist; selection mapping includes `retaildb` / `%`; inspect table statistics |
| Existing target data conflict | `DO_NOTHING` does not clear target tables; use an empty target table/schema for this lab |

## Cleanup — avoid ongoing charges

After saving screenshots and results:

1. Delete the CloudFormation stack `dms-mysql-full-load-lab`.
2. CloudFormation will remove the DMS task, endpoints, replication instance, RDS instances, EC2 instance, NAT Gateway/EIP, interface endpoints, and networking resources in dependency order.
3. If stack deletion is blocked, inspect the CloudFormation Events tab and remove the reported dependency.
4. Check **Billing → Bills / Cost Explorer** for remaining resources.

**Warning:** The template sets RDS deletion policies to `Delete` for lab cleanup. Deleting the stack deletes both databases. Export or snapshot any data you need before cleanup. Snapshots retained separately can continue to incur charges.

## Validation status

The template is authored against the AWS CloudFormation resource model and the supplied lab's architecture. Run `aws cloudformation validate-template` and preferably `cfn-lint` before deployment. A successful template validation does not guarantee that your account's quotas, AZ availability, instance-class availability, IAM permissions, or service-side resource creation will all succeed.
