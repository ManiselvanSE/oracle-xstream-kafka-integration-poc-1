# Configuration Guide

This document describes the configuration values you need to customize for your environment.

## Environment Variables

Create a `.env` file with these values (never commit this file):

```bash
# Oracle Database Connection
ORACLE_SYS_PASSWORD=<your-sys-password>
ORACLE_SCHEMA_USER=<your-schema-username>
ORACLE_SCHEMA_PASSWORD=<your-schema-password>
ORACLE_HOST_IP=<your-oracle-host-ip>
PDB_SERVICE_NAME=<your-pdb-service-name>
DATABASE_NAME=<your-database-name>

# XStream Configuration  
XSTREAM_CONNECT_USER=<your-xstream-connect-user>
XSTREAM_PASSWORD=<your-xstream-password>
XSTREAM_OUTBOUND_NAME=<your-xstream-outbound-server-name>

# Kafka VM
KAFKA_VM_IP=<your-kafka-vm-ip>
KAFKA_VM_HOSTNAME=<your-kafka-vm-hostname>
SSH_USER=<your-ssh-username>
SSH_KEY_PATH=<path-to-your-ssh-key>

# RAC Configuration (if using Oracle RAC)
RAC_SCAN_HOSTNAME=<your-rac-scan-hostname>
RAC_NODE1=<rac-node1-hostname>
RAC_NODE2=<rac-node2-hostname>
RAC_NODE1_IP=<rac-node1-ip>
RAC_NODE2_IP=<rac-node2-ip>
```

## Placeholder Mapping

Throughout this repository's documentation, you'll see these placeholders:

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<sys-password>` | Oracle SYS user password | `YourSecurePass123` |
| `<schema-user>` | Application schema username | `appuser` |
| `<schema-password>` | Application schema password | `AppPass456` |
| `<oracle-host-ip>` | Oracle database server IP | `192.168.1.100` |
| `<kafka-vm-ip>` | Kafka VM IP address | `192.168.1.200` |
| `<kafka-vm-hostname>` | Kafka VM hostname | `kafka-server` |
| `<ssh-user>` | SSH username for remote access | `opc` or `ubuntu` |
| `<path-to-ssh-key>` | Path to SSH private key | `~/.ssh/id_rsa` |
| `<database-name>` | Oracle database name | `PRODDB` |
| `<pdb-service-name>` | Pluggable database service name | `PRODPDB` |
| `<pdb-name>` | Pluggable database name | `PDB1` |
| `<schema-name>` | Application schema name | `APPSCHEMA` |
| `<xstream-connect-user>` | XStream connection user | `C##XSTREAM_ADMIN` |
| `<xstream-password>` | XStream user password | `XStreamPass789` |
| `<xstream-outbound-name>` | XStream outbound server name | `XOUT_SERVER` |
| `<rac-scan-hostname>` | RAC SCAN hostname | `rac-scan.example.com` |
| `<rac-node1>` | RAC node 1 hostname | `rac1` |
| `<rac-node2>` | RAC node 2 hostname | `rac2` |
| `<rac-node1-ip>` | RAC node 1 IP address | `192.168.1.10` |
| `<rac-node2-ip>` | RAC node 2 IP address | `192.168.1.11` |

## Security Best Practices

1. **Never commit credentials** to version control
2. **Use strong passwords** (minimum 12 characters, mixed case, numbers, special characters)
3. **Rotate passwords** regularly (every 90 days recommended)
4. **Use SSH keys** instead of passwords where possible
5. **Restrict SSH access** by IP address using firewall rules
6. **Enable Oracle Audit Trail** to track database access
7. **Use Oracle Wallet** for storing database credentials in production
8. **Encrypt XStream traffic** using SSL/TLS in production

## Quick Setup

1. Copy this template to create your `.env` file:
   ```bash
   cp CONFIGURATION.md .env
   ```

2. Edit `.env` and replace all placeholder values with your actual configuration

3. Source the environment file before running scripts:
   ```bash
   source .env
   ```

4. Update scripts to use environment variables instead of hardcoded values

## Example: Oracle Connection String

Replace this:
```sql
sqlplus sys/'<sys-password>'@<pdb-service-name> as sysdba
```

With your actual values:
```sql
sqlplus sys/'MySecurePassword123'@PRODPDB as sysdba
```

Or use environment variables:
```bash
sqlplus sys/${ORACLE_SYS_PASSWORD}@${PDB_SERVICE_NAME} as sysdba
```

---

**Last Updated:** April 2026
