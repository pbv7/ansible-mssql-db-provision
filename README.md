# Microsoft SQL Server Database Provisioning Tool

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Ansible Version](https://img.shields.io/badge/ansible-2.9%2B-blue.svg)](https://docs.ansible.com/ansible/latest/index.html)

Ansible-based automation solution for provisioning and configuring Microsoft SQL Server databases with enterprise-grade safety controls and validation mechanisms.

## ⚠️ Production Safety Guidelines

### IMPORTANT DISCLAIMER

**By using this tool, you acknowledge that:**

- You assume full responsibility for any changes made to your database systems
- You will ensure proper backups are in place before running this tool
- You understand that database provisioning can impact production systems
- You have tested these scripts in a non-production environment first

Microsoft SQL Server is often used for mission-critical applications. This tool makes structural changes to your database environment. **Always have a tested backup and recovery plan in place.**

### Critical Warnings and Precautions

When using this tool with production SQL Server environments, observe the following precautions:

1. **Cluster Awareness**
   - This tool includes checks to verify it's running on the primary replica in a cluster setup
   - Running database creation on secondary replicas can cause synchronization issues
   - If the tool fails with "This instance appears to be a secondary/replica node", do not override this check

2. **Transaction Log Safety**
   - Incorrect log growth settings can significantly impact database performance
   - Large fixed-size growth values (>1GB) may cause performance issues during log expansion
   - Use fixed-size growth (Microsoft best practice) with moderate increments (64-512MB) in production
   - Avoid percentage-based growth in production as it leads to unpredictable, exponential growth patterns

3. **Password Security**
   - Never hardcode credentials in scenario files
   - Use environment variables or Ansible Vault for all credentials:

     ```yaml
     runtime_mssql_password: "{{ lookup('env', 'MSSQL_PASSWORD') }}"
     ```

4. **Backup Requirements**
   - **ALWAYS** back up the master database before running this tool
   - Create full backups of any existing databases that might be affected
   - Verify your backup strategy works by testing a restore process
   - Keep backup history for compliance and recovery purposes

### 🔐 Security Best Practices

1. **Use Least Privilege Accounts**
   - The SQL account used should have only the necessary permissions
   - Create a dedicated account for database provisioning rather than using 'sa'
   - Document and regularly review permissions granted to automation accounts

2. **Secure Communication**
   - Use encrypted connections to SQL Server when possible
   - Consider implementing network-level security (firewalls, VPNs) for remote provisioning
   - Restrict network access to database servers to known automation hosts

3. **Audit Logging**
   - Enable SQL Server audit logging to track database creation operations
   - Keep Ansible logs for audit purposes
   - Document all database provisioning operations with change request references
   - Consider implementing a formal change management process for database provisioning

4. **Resource Impact**
   - Creating multiple databases simultaneously can impact SQL Server performance
   - Schedule database provisioning during low-traffic periods
   - Monitor SQL Server resource usage during operations
   - Consider rate-limiting database creation in large-scale deployments

### ⚙️ Recommended Workflow for Production

1. Develop and test your scenario in a non-production environment
2. Review log growth settings and other performance-impacting parameters
3. Perform a dry run with `ansible-playbook site.yml -i inventory.yml -e runtime_config_file="config/your_scenario.yml" --check`
4. Schedule a maintenance window for the actual deployment
5. Back up the master database before running the playbook
6. Run the playbook against production
7. Verify database settings post-deployment
8. Document the changes for operational records
9. Retain logs and evidence of changes for compliance and auditing purposes

## Table of Contents

- [Microsoft SQL Server Database Provisioning Tool](#microsoft-sql-server-database-provisioning-tool)
  - [⚠️ Production Safety Guidelines](#️-production-safety-guidelines)
    - [IMPORTANT DISCLAIMER](#important-disclaimer)
    - [Critical Warnings and Precautions](#critical-warnings-and-precautions)
    - [🔐 Security Best Practices](#-security-best-practices)
    - [⚙️ Recommended Workflow for Production](#️-recommended-workflow-for-production)
  - [Table of Contents](#table-of-contents)
  - [Project Overview](#project-overview)
  - [Technical Requirements](#technical-requirements)
  - [Core Capabilities](#core-capabilities)
  - [Usage Instructions](#usage-instructions)
  - [Scenario Configuration](#scenario-configuration)
  - [Configuration Example](#configuration-example)
    - [Scenario File Example](#scenario-file-example)
    - [`vars.yml` Example](#varsyml-example)
    - [Log Autogrowth Configuration](#log-autogrowth-configuration)
      - [Percentage-based growth with limited maximum size (not recommended by Microsoft for production)](#percentage-based-growth-with-limited-maximum-size-not-recommended-by-microsoft-for-production)
      - [Fixed-size growth with limited maximum size](#fixed-size-growth-with-limited-maximum-size)
      - [Fixed-size growth with unlimited maximum size](#fixed-size-growth-with-unlimited-maximum-size)
      - [Percentage-based growth with unlimited maximum size (not recommended by Microsoft for production)](#percentage-based-growth-with-unlimited-maximum-size-not-recommended-by-microsoft-for-production)
    - [SQL Server Unlimited Log Size Behavior](#sql-server-unlimited-log-size-behavior)
  - [Validation and Standards](#validation-and-standards)
  - [Security and Idempotency](#security-and-idempotency)
    - [Handling Sensitive Data](#handling-sensitive-data)
    - [Handling Sensitive Data in `vars.yml`](#handling-sensitive-data-in-varsyml)
    - [Idempotency Workaround](#idempotency-workaround)
  - [Extension Guidelines](#extension-guidelines)
  - [License](#license)
  - [Author](#author)

## Project Overview

Automates MSSQL database provisioning with:

- YAML-driven configuration
- Pre-deployment safety validations
- Standardized database settings
- Post-deployment verification

## Technical Requirements

- Ansible 2.9 or higher
- Python 3.8 or higher
- `community.general` Ansible collection
- MSSQL Server access and administrative credentials
- Network connectivity to target SQL instances

## Core Capabilities

1. Configuration-based deployment
2. Existing database protection
3. Parameter validation against allowed values
4. Automated post-deployment verification
5. Environment-specific configurations

## Usage Instructions

```bash
# Deploy with specified configuration
ansible-playbook site.yml -i inventory.yml -e runtime_config_file="config/your_scenario.yml"
```

## Scenario Configuration

To define a new deployment scenario, create a YAML file in the `config/` directory and copy or prepare variables file. You can do this by:

1. **Copying an example file**:

    ```bash
    cp config/scenario_test_dbs.example.yml config/your_scenario.yml
    ```

2. **Copying the `vars.example.yml` file**:
    For centralized validation rules, copy the example `vars.yml` file:

    ```bash
    cp group_vars/all/vars.example.yml group_vars/all/vars.yml
    ```

3. **Editing the new files**:
    Open `config/your_scenario.yml` and `group_vars/all/vars.yml` to replace the placeholder values with your environment-specific settings.

**Important**: Scenario files and `vars.yml` contain sensitive data and are ignored by Git. Only example files (`*.example.yml`) should be committed.

## Configuration Example

### Scenario File Example

```yaml
# config/scenario_name.yml
# MSSQL connection details (specific to this scenario)
runtime_mssql_host: "{{ lookup('env', 'MSSQL_HOST') }}"  # Hostname or IP address of the SQL Server
runtime_mssql_port: "{{ lookup('env', 'MSSQL_PORT') | default(1433) }}"  # Port number (default: 1433)
runtime_mssql_user: "{{ lookup('env', 'MSSQL_USER') }}"  # Username for SQL Server authentication
runtime_mssql_password: "{{ lookup('env', 'MSSQL_PASSWORD') }}"  # Password (use environment variable or Ansible Vault)

mssql_databases:
  # Example: Fixed-size growth with limited maximum size (Microsoft best practice)
  - name: "db-test1"  # Generic example database name
    owner: "dbuser1"  # Database owner (must match allowed_owners in vars.yml)
    collation: "Ukrainian_CI_AS"  # Must match allowed_collations in vars.yml
    compatibility: 140  # SQL Server 2017
    log_autogrowth:
      type: fixed_size  # Microsoft recommended best practice for production
      value: 64         # Growth increment in MB (recommended: 64-512MB)
      max_size_type: limited
      max_size_mb: 1024
```

### `vars.yml` Example

```yaml
# Database collation settings allowed by your organization
# Recommendation: Limit to collations actually used in your environment
allowed_collations:
  - Ukrainian_CI_AS        # Case-insensitive Ukrainian collation
  - Latin1_General_CI_AS   # SQL Server default collation

# Database owners allowed by your organization
# Security recommendation: Use dedicated service accounts, not sa or administrators
allowed_owners:
  - dbuser1   # Replace with actual database service accounts
  - dbuser2   # Example service account

# SQL Server compatibility levels permitted
# These values correspond to specific SQL Server versions:
# 130 = SQL Server 2016
# 140 = SQL Server 2017
# 150 = SQL Server 2019
# 160 = SQL Server 2022
allowed_compatibility_levels:
  - 130
  - 140
  - 150
  - 160

# Log file autogrowth strategies
# percent: Grows by percentage of current size (not recommended by Microsoft for production)
# fixed_size: Grows by fixed MB amount (Microsoft best practice for production)
allowed_log_autogrowth_types:
  - percent   # Not recommended by Microsoft for production environments
  - fixed_size  # Microsoft's recommended best practice for production

# Maximum size limit types
# limited: Sets a specific MB limit
# unlimited: Allows unlimited growth (use with caution in production)
allowed_max_size_types:
  - limited
  - unlimited

# Limits for log autogrowth values
# These help prevent performance issues from extreme settings
log_autogrowth_limits:
  percent:
    min: 1     # Minimum allowed percentage growth
    max: 50    # Maximum allowed percentage growth (larger can cause issues)
                # NOTE: Microsoft recommends fixed-size growth over percentage-based for production
  fixed_size:
    min_mb: 1      # Minimum growth increment in MB
    max_mb: 1024   # Microsoft recommends values between 64MB-512MB for most databases
```

### Log Autogrowth Configuration

The playbook supports two strategies for database log file growth and options for limiting or setting unlimited maximum size.

#### Percentage-based growth with limited maximum size (not recommended by Microsoft for production)

```yaml
log_autogrowth:
  type: percent       # NOT RECOMMENDED for production by Microsoft
  value: 10           # 10% growth each time
  max_size_type: limited
  max_size_mb: 2048   # Maximum size in MB
```

#### Fixed-size growth with limited maximum size

```yaml
log_autogrowth:
  type: fixed_size    # RECOMMENDED by Microsoft for production
  value: 64           # Grows in 64 MB increments (recommended: 64-512MB)
  max_size_type: limited
  max_size_mb: 1024   # Maximum size in MB
```

#### Fixed-size growth with unlimited maximum size

```yaml
log_autogrowth:
  type: fixed_size    # RECOMMENDED by Microsoft for production
  value: 64           # Grows in 64 MB increments (recommended: 64-512MB)
  max_size_type: unlimited  # No size limit - use with caution
```

#### Percentage-based growth with unlimited maximum size (not recommended by Microsoft for production)

```yaml
log_autogrowth:
  type: percent       # NOT RECOMMENDED for production by Microsoft
  value: 10           # 10% growth each time
  max_size_type: unlimited  # No size limit - use with caution
```

### SQL Server Unlimited Log Size Behavior

When configuring log files with `max_size_type: unlimited`, it's important to understand how SQL Server represents unlimited sizes:

1. SQL Server internally represents "UNLIMITED" as a very large number (268435456 pages or higher)
2. Using `MAXSIZE = UNLIMITED` in T-SQL is the proper way to set an unlimited log file size
3. The verification query in this playbook properly identifies unlimited sizes by checking for either `-1` or values greater than or equal to 268435456
4. In SQL Server Management Studio, an unlimited log file will show as "Unlimited" in the UI, but may appear as a large number (such as 2,097,152 MB) in some query results

This behavior is normal and correctly handled by the playbook's validation tasks.

## Validation and Standards

To enforce organizational standards, this project uses centralized validation rules defined in `group_vars/all/vars.yml`. These rules ensure that all provisioned databases adhere to predefined allowed values for:

- Collations
- Database owners
- Compatibility levels
- Growth settings

## Security and Idempotency

### Handling Sensitive Data

Scenario configuration files contain sensitive credentials. It is highly recommended to encrypt these files using **Ansible Vault**.

```bash
# Encrypt a scenario file
ansible-vault encrypt config/your_scenario.yml
```

When running the playbook, use the `--ask-vault-pass` flag to provide the password.

### Handling Sensitive Data in `vars.yml`

The `group_vars/all/vars.yml` file may contain sensitive data, such as allowed database owners. To ensure security:

- **Encrypt the file** using **Ansible Vault**:

    ```bash
    ansible-vault encrypt group_vars/all/vars.yml
    ```

- **Exclude sensitive files** from the repository by adding them to `.gitignore`.

- Replace sensitive values with placeholders and load them dynamically from encrypted vault files or environment variables.

### Idempotency Workaround

The `community.general.mssql_script` module does not always behave idempotently and may report a "changed" status even for read-only queries (like checking if a database exists). To prevent false positives and ensure true idempotency, this project explicitly sets `changed_when: false` on all read-only tasks. This guarantees that the playbook only reports changes when it actually modifies a database.

## Extension Guidelines

1. Add new parameters to scenario files
2. Implement validation rules in `group_vars/all/vars.yml`
3. Update deployment tasks in the `mssql_database` role
4. Add verification steps to confirm new settings

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Author

Created by Bohdan Potishuk.
