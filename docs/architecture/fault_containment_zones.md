# Fault Containment Zones

Each AWS-style service must be segmented into independent fault domains:

1. **AZ-Level Containment:** Automation operates within its AZ, isolated for 60 minutes.
2. **Regional Partitioning:** Control planes decoupled by region, preventing global recursion.
3. **Cross-Service Containment:** Lambda, EC2, and DynamoDB share no real-time dependency loops.
