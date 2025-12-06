# Sybase ASE and Replication Server Lab Environment

This directory contains Docker configurations for setting up a Sybase lab environment with:
- 2 Sybase ASE 16.0 containers (using `datagrip/sybase:16.0` community image)
- Replication Server template (requires SAP installer - not included)

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Docker Network (sybase-net)                   │
│                          172.28.0.0/16                               │
│                                                                      │
│  ┌─────────────────┐                         ┌─────────────────┐    │
│  │  ASE Primary    │                         │  ASE Secondary  │    │
│  │  SYBASE         │◄───────────────────────►│  SYBASE         │    │
│  │  172.28.0.10    │                         │  172.28.0.11    │    │
│  │  Port: 5000     │                         │  Port: 5002     │    │
│  └─────────────────┘                         └─────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

### Docker and Docker Compose

Ensure Docker and Docker Compose are installed:
```bash
docker --version
docker-compose --version
```

No additional downloads required - the lab uses the community `datagrip/sybase:16.0` Docker image.

## Quick Start

### 1. Start the Environment

```bash
cd sybase-lab
docker-compose up -d
```

### 2. Check Status

```bash
docker-compose ps
docker-compose logs -f
```

### 3. Connect to ASE

Connect to Primary ASE:
```bash
docker exec -it sybase-primary /opt/sybase/OCS-16_0/bin/isql -Usa -PmyPassword -SMYSYBASE
```

Connect to Secondary ASE:
```bash
docker exec -it sybase-secondary /opt/sybase/OCS-16_0/bin/isql -Usa -PmyPassword -SMYSYBASE
```

## Default Credentials

The `datagrip/sybase:16.0` image uses these default credentials:
- **Username**: sa
- **Password**: myPassword
- **Server Name**: MYSYBASE

## Port Mappings

| Service | Container Port | Host Port | Description |
|---------|---------------|-----------|-------------|
| ASE Primary | 5000 | 5000 | ASE Server |
| ASE Primary | 5001 | 5001 | Backup Server |
| ASE Secondary | 5000 | 5002 | ASE Server |
| ASE Secondary | 5001 | 5003 | Backup Server |

## Data Persistence

Data is stored in Docker volumes:
- `ase-primary-data` - Primary ASE data files
- `ase-secondary-data` - Secondary ASE data files

To remove all data and start fresh:
```bash
docker-compose down -v
```

## Troubleshooting

### Container Won't Start

Check the logs:
```bash
docker-compose logs ase-primary
docker-compose logs ase-secondary
docker-compose logs repserver
```

### Connection Refused

1. Ensure the container is running: `docker-compose ps`
2. Check if the server is listening: `docker exec sybase-primary netstat -tlnp`
3. Verify the interfaces file: `docker exec sybase-primary cat /opt/sybase/interfaces`

### Memory Issues

ASE requires significant memory. Ensure Docker has at least 4GB RAM allocated.

Edit Docker Desktop settings or for Linux:
```bash
# Check available memory
free -h

# Increase Docker memory limit if needed
```

## Setting Up Replication

Once all containers are running, you can set up replication between the ASE instances.

### 1. Create a Test Database on Primary

```sql
-- Connect to SYBASE_PRIMARY
create database testdb on default = 100
go
use testdb
go
create table orders (
    order_id int identity primary key,
    customer_name varchar(100),
    order_date datetime default getdate()
)
go
```

### 2. Enable RepAgent on Primary

```sql
-- Enable RepAgent for the database
sp_config_rep_agent testdb, 'enable', 'REPSERVER', 'testdb_prim'
go
sp_start_rep_agent testdb
go
```

### 3. Create Replication Definition

```sql
-- Connect to REPSERVER
create replication definition orders_rep
with primary at SYBASE_PRIMARY.testdb
with all tables named 'orders'
(order_id, customer_name, order_date)
primary key (order_id)
go
```

### 4. Create Subscription

```sql
-- Create subscription on secondary
create subscription orders_sub
for orders_rep
with replicate at SYBASE_SECONDARY.testdb
without materialization
go
```

## Directory Structure

```
sybase-lab/
├── docker-compose.yml      # Main compose file
├── README.md               # This file
├── ase/                    # ASE container files
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── interfaces.template
│   ├── sybase-ase.rs.template
│   ├── sybase-response.txt
│   └── sysctl.conf
├── repserver/              # Replication Server files
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── interfaces.template
│   ├── rs-config.rs.template
│   ├── rs-response.txt
│   └── sysctl.conf
└── scripts/                # Shared scripts
    ├── healthcheck.sh
    └── rs_healthcheck.sh
```

## License

SAP ASE Developer Edition is free for development and testing purposes. Production use requires a commercial license from SAP.

## References

- [SAP ASE Documentation](https://help.sap.com/docs/SAP_ASE)
- [SAP Replication Server Documentation](https://help.sap.com/docs/SAP_REPLICATION_SERVER)
- [Sybase Infocenter](https://infocenter.sybase.com)
