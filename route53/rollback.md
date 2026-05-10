This is a critical operation. When you delete a Hosted Zone and recreate it, **AWS assigns a brand new set of Name Servers to the new zone**.

If you blindly push the old NS records from your JSON backup into the Main Domain, DNS resolution will break because the parent domain will point to dead name servers.

This rollback script handles that architectural edge case perfectly. It will:

1. Delete the ALB Alias A-Record from the Main Domain.
2. Recreate the Property Hosted Zone.
3. Read your JSON backup, filter out the old default `SOA`/`NS` records, and restore all your custom records (TXT, MX, subdomains, etc.).
4. Grab the **newly assigned** Name Servers from the newly created zone and automatically update the Main Domain's NS record to properly restore delegation.

### The Rollback Automation Script

```python
import boto3
import json
import os
import uuid

# Initialize AWS client
r53_client = boto3.client('route53')

# --- CONFIGURATION ---
MAIN_DOMAIN = "certifyme.co."
PROPERTY_SPECIFIC_DOMAIN = "southcap.certifyme.co."
BACKUP_DIR = "dns_backups"
# ---------------------

def format_domain(domain):
    return domain if domain.endswith('.') else f"{domain}."

def get_hosted_zone_id(domain_name):
    """Fetches the Hosted Zone ID for a given domain name."""
    paginator = r53_client.get_paginator('list_hosted_zones')
    for page in paginator.paginate():
        for zone in page['HostedZones']:
            if zone['Name'] == domain_name:
                return zone['Id'].split('/')[-1]
    return None

def rollback():
    main_domain = format_domain(MAIN_DOMAIN)
    property_domain = format_domain(PROPERTY_SPECIFIC_DOMAIN)
    
    zone_backup_file = os.path.join(BACKUP_DIR, f"backup_zone_{property_domain.strip('.')}.json")
    
    if not os.path.exists(zone_backup_file):
        print(f"ERROR: Backup file {zone_backup_file} not found!")
        return

    main_zone_id = get_hosted_zone_id(main_domain)
    if not main_zone_id:
        print(f"ERROR: Could not find Main Hosted Zone {main_domain}")
        return

    print(f"--- Starting Rollback for {property_domain} ---")

    # STEP 1: Find and delete the ALB Alias record in the Main Zone
    print("1. Cleaning up ALB Alias record from Main Domain...")
    main_records = r53_client.list_resource_record_sets(HostedZoneId=main_zone_id)['ResourceRecordSets']
    alias_record = next((r for r in main_records if r['Name'] == property_domain and r['Type'] == 'A'), None)
    
    if alias_record:
        r53_client.change_resource_record_sets(
            HostedZoneId=main_zone_id,
            ChangeBatch={'Changes': [{'Action': 'DELETE', 'ResourceRecordSet': alias_record}]}
        )
        print("   -> ALB Alias record deleted.")
    else:
        print("   -> No ALB Alias record found. Moving on.")

    # STEP 2: Recreate the Property Hosted Zone
    print(f"2. Recreating Hosted Zone for {property_domain}...")
    caller_ref = str(uuid.uuid4()) # Unique string required by AWS to prevent duplicate creation
    create_response = r53_client.create_hosted_zone(
        Name=property_domain,
        CallerReference=caller_ref,
        HostedZoneConfig={'Comment': 'Restored via rollback script'}
    )
    
    new_zone_id = create_response['HostedZone']['Id'].split('/')[-1]
    new_name_servers = create_response['DelegationSet']['NameServers']
    print(f"   -> Created new zone ID: {new_zone_id}")

    # STEP 3: Restore custom records to the new Property Zone
    print("3. Restoring records from JSON backup...")
    with open(zone_backup_file, 'r') as f:
        backup_data = json.load(f)

    valid_changes = []
    for change in backup_data['Changes']:
        record = change['ResourceRecordSet']
        # DO NOT restore the apex SOA and NS records from the backup
        if record['Name'] == property_domain and record['Type'] in ['SOA', 'NS']:
            continue
        valid_changes.append(change)

    if valid_changes:
        # Route53 limits batches to 1000 records
        for i in range(0, len(valid_changes), 1000):
            batch = valid_changes[i:i+1000]
            r53_client.change_resource_record_sets(
                HostedZoneId=new_zone_id,
                ChangeBatch={'Changes': batch}
            )
        print(f"   -> Successfully restored {len(valid_changes)} records to the property zone.")
    else:
        print("   -> No custom records to restore in the backup.")

    # STEP 4: Update the Main Domain with the NEW Name Servers
    print(f"4. Updating NS delegation in Main Domain ({main_domain})...")
    ns_resource_records = [{'Value': ns} for ns in new_name_servers]
    
    ns_change = {
        'Action': 'UPSERT',
        'ResourceRecordSet': {
            'Name': property_domain,
            'Type': 'NS',
            'TTL': 300,
            'ResourceRecords': ns_resource_records
        }
    }
    
    r53_client.change_resource_record_sets(
        HostedZoneId=main_zone_id,
        ChangeBatch={'Changes': [ns_change]}
    )
    print("   -> NS record restored with fresh Name Servers.")
    
    print("\n✅ Rollback completed successfully.")
    print("⚠️ Note: DNS propagation for the new Name Servers may take a few minutes.")

if __name__ == "__main__":
    rollback()

```

### How to use this script

1. Ensure your `dns_backups` folder (created by the first script) is in the same directory.
2. Edit the `PROPERTY_SPECIFIC_DOMAIN` variable at the top of the file to target the specific domain you want to restore.
3. Run the script. It will automatically handle the cleanup, recreation, JSON parsing, and delegation mapping.
