MSSQL Database Provisioning Role
=================================

This Ansible role automates the creation and configuration of Microsoft SQL Server databases. It ensures compliance with organizational standards by validating parameters such as collation, compatibility level, and ownership. The role also includes safety mechanisms to prevent accidental modifications to existing databases.

Requirements
------------

- Ansible 2.9 or higher
- Python 3.8 or higher
- `community.general` Ansible collection
- Network access to the target SQL Server instance
- SQL Server credentials with sufficient permissions to create and configure databases

Role Variables
--------------

Connection Variables
--------------

These variables define the connection to the SQL Server:

- `runtime_mssql_host`: Hostname or IP address of the SQL Server.

  ```yaml
  runtime_mssql_host: "{{ lookup('env', 'MSSQL_HOST') }}"
  ```

- `runtime_mssql_port`: Port number for the SQL Server (default: `1433`).

  ```yaml
  runtime_mssql_port: "{{ lookup('env', 'MSSQL_PORT') | default(1433) }}"
  ```

- `runtime_mssql_user`: Username for SQL Server authentication.

  ```yaml
  runtime_mssql_user: "{{ lookup('env', 'MSSQL_USER') }}"
  ```

- `runtime_mssql_password`: Password for SQL Server authentication.

  ```yaml
  runtime_mssql_password: "{{ lookup('env', 'MSSQL_PASSWORD') }}"
  ```

Database Configuration
--------------

Define the databases to be created and their parameters:

- `mssql_databases`: A list of dictionaries, each representing a database. Example structure:

  ```yaml
  mssql_databases:
    # Example 1: Percentage-based growth (NOT RECOMMENDED for production by Microsoft)
    - name: db-test1
      owner: dbuser1     # Must match allowed_owners in vars.yml
      collation: Latin1_General_CI_AS  # Must match allowed_collations in vars.yml
      compatibility: 130  # SQL Server 2016
      log_autogrowth:
        type: percent     # NOT RECOMMENDED for production environments
        value: 5          # 5% growth each time
        max_size_type: limited
        max_size_mb: 2048
        
    # Example 2: Fixed-size growth (RECOMMENDED best practice for production)
    - name: db-test2
      owner: dbuser2
      collation: Ukrainian_CI_AS
      compatibility: 140  # SQL Server 2017
      log_autogrowth:
        type: fixed_size  # Microsoft's recommended best practice
        value: 64         # 64 MB growth increment (recommended range: 64-512MB)
        max_size_type: unlimited
  ```

The `log_autogrowth` dictionary accepts the following settings:

- `type`: The growth method, which can be one of two types:
  - `percent`: grows the log by a percentage of its current size
  - `fixed_size`: grows the log by a fixed number of megabytes
- `value`: The growth increment (percentage or MB, depending on type)
- `max_size_type`: The maximum size limit type
  - `limited`: sets a specific maximum size in megabytes
  - `unlimited`: allows the log to grow without a size limit
- `max_size_mb`: The maximum size in megabytes (required when max_size_type is 'limited')

Validation Variables
--------------

These variables define allowed values for database parameters:

- `allowed_collations`: List of allowed collation settings.
- `allowed_owners`: List of allowed database owners.
- `allowed_compatibility_levels`: List of allowed compatibility levels.
- `allowed_log_autogrowth_types`: Valid strategies for log growth (`percent` or `fixed_size`).
- `allowed_max_size_types`: Valid maximum size types (`limited` or `unlimited`).
- `log_autogrowth_limits`: Minimum and maximum values for each growth type.

Log File Configuration Notes
--------------

When configuring log files with unlimited maximum size:

- SQL Server represents "UNLIMITED" as a large number (268435456 pages) internally
- This is correctly handled in the validation logic
- Use `max_size_type: unlimited` to configure unlimited log growth
- Best practices suggest:
  - For fixed_size growth (Microsoft best practice): Use values between 64MB and 512MB
  - For percent growth (not recommended for production): Use values between 10% and 50% if needed
  - Be cautious with very large growth values as they can impact performance during log expansion

Example of proper unlimited log configuration:

```yaml
log_autogrowth:
  type: fixed_size
  value: 64              # 64MB is a Microsoft recommended value
  max_size_type: unlimited
```

Dependencies
------------

This role depends on the `community.general` Ansible collection for the `mssql_script` module. Ensure the collection is installed:

```bash
ansible-galaxy collection install community.general
```

Example Playbook
----------------

Here is an example of how to use this role:

```yaml
- name: Provision MSSQL Databases
  hosts: localhost
  connection: local
  gather_facts: false

  vars_files:
    - group_vars/all/vars.yml
    - config/scenario_test_dbs.example.yml

  roles:
    - mssql_database
```

License
-------

MIT

Author Information
------------------

This role was created by Bohdan Potishuk.

Production Safety Features
-------------------------

This role includes several safety features designed to protect production SQL Server environments:

Cluster-Awareness
--------------

The role now automatically detects if it's running against a SQL Server cluster or Availability Group configuration:

- Checks if the target instance is part of a traditional failover cluster or AlwaysOn AG
- Verifies the instance is the PRIMARY replica before allowing database creation
- Prevents accidental database creation on secondary replicas

Implementation:

```yaml
- name: Check if running on primary replica
  community.general.mssql_script:
      # Connection details
      script: |
          DECLARE @is_primary INT = 1; -- Default to primary for standalone servers
          
          -- Check if AlwaysOn AG is available (SQL 2012+)
          IF OBJECT_ID('sys.dm_hadr_availability_replica_states') IS NOT NULL
          BEGIN
              -- Additional checks for primary replica
              # ...
          END
          
          # Return node information and primary status
```

Existing Database Protection
--------------

The role prevents accidental modification of existing databases:

- Checks if any requested database already exists before attempting to create it
- Fails with an informative error message if a database would be overwritten

Parameter Validation
--------------

Ensures all database parameters meet organizational standards:

- Validates database owner against allowed list
- Validates collation settings against allowed list
- Validates compatibility level against allowed list
- Ensures log autogrowth settings follow best practices

Performance Safety Checks
--------------

Warns about potentially problematic configurations:

- Detects fixed-size log growth values exceeding 1GB (can cause performance issues)
- Verifies log file settings match the requested configuration

Post-Deployment Verification
--------------

Confirms that database creation and configuration was successful:

- Verifies database owner, collation, and compatibility level
- Verifies log file autogrowth settings match the specified values

Safety Warnings
--------------

⚠️ Fixed-Size Log Growth
--------------

Using large fixed-size log growth values (>1GB) can cause performance issues during log expansion. When SQL Server needs to grow a log file, it must zero-initialize the new space, which can cause database operations to pause while this happens.

**Recommendation**:

- Use fixed-size growth for production databases (Microsoft best practice)
- Keep growth increments between 64MB and 512MB for most databases
- For very large databases, consider values up to 1024MB maximum
- Avoid percentage-based growth as it leads to exponential growth patterns

⚠️ Log File Sizing
--------------

Inappropriate log file sizing can cause performance issues:

- Too small: Frequent growth operations impact performance
- Too large: Wastes disk space
- Unlimited growth: Can accidentally fill disk space

**Recommendation**:

- Size log files appropriately for the workload
- Monitor log file usage and growth patterns
- Consider setting reasonable limits even for databases that need large logs

⚠️ SQL Server Permissions
--------------

This role requires the following SQL Server permissions:

- CREATE DATABASE
- ALTER ANY DATABASE
- VIEW SERVER STATE (for verification queries)

Running this role with sysadmin privileges in production is not recommended.

**Recommendation**:

- Create a dedicated SQL login with only the necessary permissions
- Document and audit all provisioning operations

Troubleshooting
--------------

Primary Replica Detection Fails
--------------

If the role fails with an error related to checking primary replica status:

1. Verify SQL Server version supports the DMVs used (SQL 2012+)
2. Check if the SQL login has appropriate permissions to view server state
3. For standalone instances, you'll see a message that the tool is "Unable to determine if this is a primary replica"

Log File Configuration Issues
--------------

If verification of log file settings fails:

1. Check for conflicting settings from other policies or server defaults
2. Verify there's sufficient disk space for the requested log file size
3. Check for any SQL Server-specific limitations in your environment
