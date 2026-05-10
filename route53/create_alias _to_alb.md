This is a classic infrastructure revamp task. Automating this across multiple domains requires careful orchestration, especially when dealing with destructive actions like deleting hosted zones and records.

Since Route 53 expects very specific formats—and because you need a seamless rollback path—the script below extracts the exact `ResourceRecordSet` dictionaries and wraps them in a standard Boto3 `ChangeBatch` format (`{"Action": "UPSERT", "ResourceRecordSet": {...}}`).

Here is the complete, multithreaded Python script using `boto3`.

### Prerequisites

Make sure your environment is authenticated with AWS (via environment variables, AWS CLI profile, or IAM role) and has the necessary permissions for `route53` and `elasticloadbalancing`.

```bash
pip install boto3

```

### The Automation Script

```python
import boto3
import json
import os
import concurrent.futures
from datetime import datetime

# Initialize AWS clients
r53_client = boto3.client('route53')
elbv2_client = boto3.client('elbv2')

# Configuration
MAIN_DOMAIN = "certifyme.co."
ALB_ARN = "arn:aws:elasticloadbalancing:REGION:ACCOUNT:loadbalancer/app/NAME/ID" # Replace with your ALB ARN
PROPERTY_DOMAINS = [
    "southcap.certifyme.co.",
    # Add other domains here
]
BACKUP_DIR = "dns_backups"

# Ensure domains end with a trailing dot for Route53 compatibility
MAIN_DOMAIN = MAIN_DOMAIN if MAIN_DOMAIN.endswith('.') else f"{MAIN_DOMAIN}."

def setup():
    if not os.path.exists(BACKUP_DIR):
        os.makedirs(BACKUP_DIR)

def get_hosted_zone_id(domain_name):
    """Fetches the Hosted Zone ID for a given domain name."""
    paginator = r53_client.get_paginator('list_hosted_zones')
    for page in paginator.paginate():
        for zone in page['HostedZones']:
            if zone['Name'] == domain_name:
                return zone['Id'].split('/')[-1]
    return None

def get_alb_details(alb_arn):
    """Retrieves the DNS Name and Canonical Hosted Zone ID for the ALB."""
    print(f"Fetching ALB details for {alb_arn}...")
    response = elbv2_client.describe_load_balancers(LoadBalancerArns=[alb_arn])
    alb = response['LoadBalancers'][0]
    return alb['DNSName'], alb['CanonicalHostedZoneId']

def backup_and_format_for_rollback(zone_id, domain, backup_filename, is_main_zone=False, target_record_name=None, target_record_type=None):
    """
    Fetches records and formats them as a Boto3 ChangeBatch JSON for easy rollback.
    If is_main_zone is True, it only backs up the specific target_record.
    """
    paginator = r53_client.get_paginator('list_resource_record_sets')
    records_to_backup = []
    
    for page in paginator.paginate(HostedZoneId=zone_id):
        for record in page['ResourceRecordSets']:
            if is_main_zone:
                if record['Name'] == target_record_name and record['Type'] == target_record_type:
                    records_to_backup.append(record)
            else:
                records_to_backup.append(record)

    # Format for easy Boto3 ChangeBatch rollback
    rollback_batch = {
        "Comment": f"Rollback backup for {domain} taken at {datetime.now().isoformat()}",
        "Changes": [
            {"Action": "UPSERT", "ResourceRecordSet": rec} for rec in records_to_backup
        ]
    }

    with open(backup_filename, 'w') as f:
        json.dump(rollback_batch, f, indent=4)
    
    print(f"Backup saved to {backup_filename}")
    return records_to_backup

def delete_hosted_zone_records(zone_id, domain, records):
    """Deletes all records in a hosted zone except the default SOA and NS records."""
    changes = []
    for record in records:
        # Route 53 does not allow deletion of the apex SOA and NS records
        if record['Type'] in ['SOA', 'NS'] and record['Name'] == domain:
            continue
        changes.append({
            'Action': 'DELETE',
            'ResourceRecordSet': record
        })
    
    if changes:
        # Process in batches of 1000 (Route53 limit)
        for i in range(0, len(changes), 1000):
            batch = changes[i:i+1000]
            r53_client.change_resource_record_sets(
                HostedZoneId=zone_id,
                ChangeBatch={'Changes': batch}
            )
        print(f"Deleted non-default records from {domain}")

def process_property_domain(property_domain, main_zone_id, alb_dns, alb_zone_id):
    """The main worker function for the multithreaded executor."""
    property_domain = property_domain if property_domain.endswith('.') else f"{property_domain}."
    print(f"\n--- Starting revamp for {property_domain} ---")
    
    try:
        # 1. Get Property Zone ID
        prop_zone_id = get_hosted_zone_id(property_domain)
        if not prop_zone_id:
            print(f"SKIPPING: Could not find Hosted Zone for {property_domain}")
            return

        # 2. Backup Property Zone Records
        prop_backup_file = os.path.join(BACKUP_DIR, f"backup_zone_{property_domain.strip('.')}.json")
        prop_records = backup_and_format_for_rollback(prop_zone_id, property_domain, prop_backup_file)

        # 3. Delete non-default records, then delete the Property Hosted Zone
        delete_hosted_zone_records(prop_zone_id, property_domain, prop_records)
        r53_client.delete_hosted_zone(Id=prop_zone_id)
        print(f"Deleted Hosted Zone: {property_domain}")

        # 4. Backup NS Record from Main Zone
        ns_backup_file = os.path.join(BACKUP_DIR, f"backup_ns_{property_domain.strip('.')}.json")
        ns_records = backup_and_format_for_rollback(
            main_zone_id, MAIN_DOMAIN, ns_backup_file, 
            is_main_zone=True, target_record_name=property_domain, target_record_type='NS'
        )

        # 5. Delete NS Record from Main Zone
        if ns_records:
            r53_client.change_resource_record_sets(
                HostedZoneId=main_zone_id,
                ChangeBatch={
                    'Changes': [{'Action': 'DELETE', 'ResourceRecordSet': ns_records[0]}]
                }
            )
            print(f"Deleted NS record for {property_domain} from {MAIN_DOMAIN}")

        # 6. Create A Record (Alias) to ALB in Main Zone
        alias_change = {
            'Action': 'UPSERT',
            'ResourceRecordSet': {
                'Name': property_domain,
                'Type': 'A',
                'AliasTarget': {
                    'HostedZoneId': alb_zone_id,
                    'DNSName': alb_dns,
                    'EvaluateTargetHealth': False
                }
            }
        }
        
        r53_client.change_resource_record_sets(
            HostedZoneId=main_zone_id,
            ChangeBatch={'Changes': [alias_change]}
        )
        print(f"SUCCESS: Created Alias A record for {property_domain} pointing to ALB.")

    except Exception as e:
        print(f"ERROR processing {property_domain}: {str(e)}")


def main():
    setup()
    
    # Pre-fetch Main Zone ID and ALB details to avoid redundant API calls in threads
    main_zone_id = get_hosted_zone_id(MAIN_DOMAIN)
    if not main_zone_id:
        print(f"FATAL: Could not find Main Hosted Zone {MAIN_DOMAIN}")
        return

    try:
        alb_dns, alb_zone_id = get_alb_details(ALB_ARN)
        print(f"ALB Resolved -> DNS: {alb_dns}, ZoneID: {alb_zone_id}")
    except Exception as e:
        print(f"FATAL: Failed to resolve ALB ARN: {e}")
        return

    # Execute domains concurrently
    # Adjust max_workers based on your AWS API rate limits (Route53 is 5 requests/sec hard limit usually)
    with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
        futures = [
            executor.submit(process_property_domain, domain, main_zone_id, alb_dns, alb_zone_id)
            for domain in PROPERTY_DOMAINS
        ]
        
        # Wait for all threads to complete
        concurrent.futures.wait(futures)
        
    print("\nRevamp script execution completed.")

if __name__ == "__main__":
    main()

```

### How to use the Rollback JSON files

The script outputs JSON files (e.g., `backup_zone_southcap.certifyme.co.json`) perfectly formatted as a Boto3 `ChangeBatch`.

**If you need to rollback the property zone:**

1. Manually recreate the `PROPERTY_SPECIFIC_DOMAIN` hosted zone in the AWS Console or via CLI (Deleting a zone destroys the ID, so a new one must be generated).
2. Grab the new Hosted Zone ID.
3. Open the JSON backup, remove the default `SOA` and `NS` objects from the `"Changes"` array (since AWS auto-generates new ones for the new zone).
4. Run a simple script to push the JSON back:
```python
with open('backup_zone_southcap.certifyme.co.json') as f:
    change_batch = json.load(f)
r53_client.change_resource_record_sets(HostedZoneId='NEW_ZONE_ID', ChangeBatch=change_batch)

```



**If you need to rollback the NS record in the Main Domain:**
Simply pass the `backup_ns_southcap.certifyme.co.json` directly into `change_resource_record_sets` targeting the `MAIN_DOMAIN` Hosted Zone ID. The `UPSERT` action in the JSON will recreate it.

```

```
