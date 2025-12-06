# Sybase ASE Migration Toolkit

A comprehensive toolkit for Sybase ASE (SAP Adaptive Server Enterprise) administration, replication, and migration to SQL Server.

## Contents

### Documentation

- **[Sybase DBA Knowledge Guide](Sybase_DBA_Knowledge_Guide.md)** - Comprehensive guide covering:
  - ASE architecture and internals
  - Replication Server concepts
  - Administration tasks
  - Migration strategies to SQL Server
  - External migration tools (Ispirer, SQLines, CData, Fivetran, etc.)

- **[ASE vs SQL Server Comparison](ASE_vs_SQLServer_Comparison.md)** - Detailed comparison including:
  - Data type mappings
  - T-SQL syntax differences (critical items like `*=` outer join conversion)
  - System catalog mappings
  - Migration checklists

### Lab Environment

- **[sybase-lab/](sybase-lab/)** - Docker-based Sybase lab environment with:
  - 2 Sybase ASE 16.0 containers (primary and secondary)
  - Pre-configured networking for replication testing
  - Ready-to-use with `docker-compose up -d`

## Quick Start

### Start the Sybase Lab

```bash
cd sybase-lab
docker-compose up -d
```

### Connect to ASE

```bash
# Primary ASE
docker exec -it sybase-primary /opt/sybase/OCS-16_0/bin/isql -Usa -PmyPassword -SMYSYBASE

# Secondary ASE
docker exec -it sybase-secondary /opt/sybase/OCS-16_0/bin/isql -Usa -PmyPassword -SMYSYBASE
```

### Default Credentials

- **Username**: sa
- **Password**: myPassword
- **Server Name**: MYSYBASE

## Port Mappings

| Service | Host Port | Description |
|---------|-----------|-------------|
| ASE Primary | 5000 | Primary ASE Server |
| ASE Secondary | 5002 | Secondary ASE Server |

## Use Cases

This toolkit is designed for:

1. **Learning Sybase ASE** - Hands-on lab environment for DBA training
2. **Migration Planning** - Documentation and comparison guides for SQL Server migration
3. **Replication Testing** - Two-node setup for testing replication scenarios
4. **Development** - Local Sybase environment for application development

## License

The Sybase ASE containers use the `datagrip/sybase:16.0` community Docker image. SAP ASE Developer Edition is free for development and testing purposes. Production use requires a commercial license from SAP.

## References

- [SAP ASE Documentation](https://help.sap.com/docs/SAP_ASE)
- [SAP Replication Server Documentation](https://help.sap.com/docs/SAP_REPLICATION_SERVER)
- [Sybase Infocenter](https://infocenter.sybase.com)
