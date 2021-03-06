linux-system-roles-rsyslog
==========================

# Guidelines for Using Rsyslog Ansible Roles

This is the ansible rsyslog roles to deploy configuration files.

Table of Contents
=================

<!--ts-->
   * [linux-system-roles-rsyslog](#linux-system-roles-rsyslog)
   * [Table of contents](#table-of-contents)
   * [Deploy Rsyslog Configuration Files](#deploy)
      * [Inventory File](#inventory-file)
      * [vars.yaml](#vars)
      * [playbook.yaml](#playbook)
   * [Variables in vars.yaml](#variables)
      * [Common sub-variables](#common-variables)
      * [Viaq sub-variables](#viaw)
   * [Contents of Roles](#roles)
<!--te-->

Deploy Rsyslog Configuration Files
==================================

Typical ansible-playbook command line.

``` ansible-playbook [-vvv] -e@vars.yaml --become --become-user root -i inventory_file playbook.yaml ```

Two files - inventory_file and vars.yaml - in the command line are to be updated by the user.

Inventory File
--------------
inventory_file is used to specify the hosts to deploy the configuration files.

   Sample inventory file
```
[masters]
localhost ansible_user=YOUR_ANSIBLE_USER

[nodes]
localhost ansible_user=YOUR_ANSIBLE_USER
```

vars.yaml
---------
vars.yaml stores variables which are passed to ansible to control the tasks.  The contents of this file could be merged into the inventory file.

Currently, the role supports 3 types of logs collections ([input_roles](https://github.com/linux-system-roles/logging/tree/master/roles/rsyslog/roles/input_roles/)): `basics`, `ovirt`, and `viaq`.  And 3 types of log outputs ([output_roles](https://github.com/linux-system-roles/logging/tree/master/roles/rsyslog/roles/output_roles/)): `elasticsearch`, `files`, and `forwards`.  To deploy rsyslog configuration files with these input and output roles, first specify the output_role as `logging_outputs`, then input_role as `logs_collections` in each `logging_outputs`.  Multiple input roles could be required based on the use cases.

To make an effect with the following setting, vars.yaml has to have `rsyslog_enabled: true`.  Unless `rsyslog_enabled` is set to true, LSR/Logging does not deploy rsyslog config files.
```
rsyslog_enabled: true
logging_outputs:
  -name: <output_role_name0>
   type: <output_type0>
   logs_collections:
     - name: <input_role_nameA>
     - name: <input_role_nameB>
  -name: <output_role_name1>
   type: <output_type1>
   logs_collections:
     - name: <input_role_nameC>
```

See the [variables section](#variables) for each variable.

**vars.yaml examples:**

0. Deploying default /etc/rsyslog.conf.
```
rsyslog_enabled: true
```

0. Overriding existing /etc/rsyslog.conf and files in /etc/rsyslog.d with default rsyslog.conf.  Pre-existing config files are copied to the directory specified by `rsyslog_backup_dir`.
```
rsyslog_enabled: True
rsyslog_purge_original_conf: True
rsyslog_backup_dir: /tmp/rsyslog_backup
```

1. Deploying basic LSR/Logging config files in /etc/rsyslog.d, which handle inputs from the local system and outputs into the local files.  Pre-existing config files are copied to the directory specified by `rsyslog_backup_dir`.
```
rsyslog_enabled: True
rsyslog_default: False
rsyslog_purge_original_conf: True
rsyslog_backup_dir: /tmp/rsyslog_backup
logging_outputs:
  - name: local-files
    type: files
    logs_collections:
      - name: 'basics'
```

2. Deploying basic LSR/Logging config files in /etc/rsyslog.d, which handle inputs from the local system and remote rsyslog and outputs into the local files.
```
rsyslog_enabled: True
rsyslog_default: False
rsyslog_capabilities: [ 'network', 'remote-files' ]
rsyslog_purge_original_conf: True
logging_outputs:
  - name: local-files
    type: files
    logs_collections:
      - name: 'basics'
```

3. Sample vars.yaml file for the viaq case.
```
rsyslog_enabled: True
rsyslog_elasticsearch:
  - name: viaq-elasticsearch
    type: elasticsearch
    logs_collections:
      - name: 'viaq'
    server_host: es-hostname
    server_port: 9200
```

4. vars.yaml to configure to handle the inputs from openshift containers
```
rsyslog_enabled: true
# If 'viaq-k8s' is in logs collections, logging_mmk8s_* need to be specified.
logging_mmk8s_token: "{{rsyslog_viaq_config_dir}}/mmk8s.token"
logging_mmk8s_ca_cert: "{{rsyslog_viaq_config_dir}}/mmk8s.ca.crt"
# If use_omelasticsearch_cert is True, ca_cert, cert and key in rsyslog_outputs needs to be set.
use_omelasticsearch_cert: True
# If use_local_omelasticsearch_cert is True, local files ca_cert_src, cert_src and key_src will be deployed to the remote host.
use_local_omelasticsearch_cert: True
rsyslog_outputs:
  - name: viaq-elasticsearch
    type: elasticsearch
    rsyslog_logs_collections:
      - name: 'viaq'
        state: 'present'
      - name: 'viaq-k8s'
        state: 'present'
    server_host: logging-es
    server_port: 9200
    index_prefix: project.
    ca_cert: "{{rsyslog_viaq_config_dir}}/es-ca.crt"
    cert: "{{rsyslog_viaq_config_dir}}/es-cert.pem"
    key: "{{rsyslog_viaq_config_dir}}/es-key.pem"
    ca_cert_src : "/path/to/es-ca.crt"
    cert_src : "/path/to/es-cert.pem"
    key_src : "/path/to/es-key.pem"
  - name: viaq-elasticsearch-ops
    type: elasticsearch
    rsyslog_logs_collections:
      - name: 'viaq'
        state: 'present'
      - name: 'viaq-k8s'
        state: `present`
    server_host: logging-es-ops
    server_port: 9200
    index_prefix: .operations.
    ca_cert: "{{rsyslog_viaq_config_dir}}/es-ca.crt"
    cert: "{{rsyslog_viaq_config_dir}}/es-cert.pem"
    key: "{{rsyslog_viaq_config_dir}}/es-key.pem"
    ca_cert_src : "/path/to/es-ca.crt"
    cert_src : "/path/to/es-cert.pem"
    key_src : "/path/to/es-key.pem"
```
The key-value pairs in `rsyslog_elasticsearch` are used to configure rsyslog to send the logs to the Openshift Aggregated Logging ElasticSearch.  The elements are used in the output elasticsearch configuration 30-elasticsearch.conf as follows (note: not following the abstract syntax):
```
if index_prefix starts with "project." then {
  action( 
      type="omelasticsearch"
      name="viaq-elasticsearch"
      server="logging-es"
      serverport="9200"
      ....
      tls.cacert="/etc/rsyslog.d/viaq/es-ca.crt"
      tls.mycert="/etc/rsyslog.d/viaq/es-cert.pem"
      tls.myprivkey="/etc/rsyslog.d/viaq/es-key.pem"
  )
} else {
  action( 
      type="omelasticsearch"
      name="viaq-elasticsearch-ops"
      server="logging-es-ops"
      serverport="9200"
      ....
      tls.cacert="/etc/rsyslog.d/viaq/es-ca.crt"
      tls.mycert="/etc/rsyslog.d/viaq/es-cert.pem"
      tls.myprivkey="/etc/rsyslog.d/viaq/es-key.pem"
  )
}
```
The order of the list in `rsyslog_elasticsearch` is important.  The first item is in the first if clause with the index_prefix value and the last item is in the else clause.  Elements in between will be placed with else if clause.

The variables ca_cert, cert and key in `rsyslog_elasticsearch` specify the paths where the CA certificate, certificate and key are located in the remote host.  The variables ca_cert_src, cert_src, and key_src are paths of them to be deployed to the remote host.

5. vars.yaml to configure custom config files.

   To include existing config files in the new ansible deployment, add the paths to `rsyslog_custom_config_files` as follows.  The specified files are copied to /etc/rsyslog.d.
```
....
rsyslog_custom_config_files: [ '/path/to/custom_A.conf', '/path/to/custom_B.conf' ]
```

playbook.yaml
-------------

```
- name: install and configure rsyslog on the nodes
  hosts: nodes
  roles:
    - role: rsyslog
      tags: [ 'role::rsyslog' ]
```

Variables in vars.yaml
======================

- `rsyslog_enabled` : When 'True', rsyslog role will deploy specified configuration file set. Default to 'True'.
- `rsyslog_capabilities` : List of capabilities to configure.  [ 'network', 'remote-files', 'tls', 'gnutls', 'kernel-message', 'mark' ] are predefined.
   To receive remote input, you could add 'network' to `rsyslog_capabilities`, which will configure imudp as well as imptcp.
   To put logs from the remote input in the separate files, 'remote-files' is to be added to `rsyslog_capabilities`.
   To make the network communication safe, enable `rsyslog_pki` and add `tls` to `rsyslog_capabilities`.
   To log all kernel messages to the console, add `kernel-message` to `rsyslog_capabilities`.
   To add `-- MARK --` message every hour, add `mark` to `rsyslog_capabilities`.
- `rsyslog_default`: If set as `True`, rsyslog.conf will be configured with default configurations and rules.
- `rsyslog_pki` : When 'True', pki related variables are configured, which are `rsyslog_pki_path`, `rsyslog_pki_realm`, `rsyslog_pki_ca`, `rsyslog_pki_crt`, `rsyslog_pki_key`.  In addition, if 'tls' is included in `rsyslog_capabilities`, it enables to forward logs over TLS.  Default to 'False'.
- `rsyslog_send_over_tls_only` : When 'True', insecure connection is not allowed.  I.e., it requires `tls` in `rsyslog_capabilities`.  Default to 'False'.

Common sub-variables
--------------------
- `rsyslog_backup_dir`: By default, the Rsyslog backs up the pre-existing configuration files in a temp dir as tar-gz format - /tmp/rsyslog.d-XXXXXX/backup.tgz.  By setting a path to `rsyslog_backup_dir`, the path is used as the backup directory.  Note that the directory should exist and have the permission to create the backup file both in the file mode and the selinux.
- `rsyslog_config_dir`: Directory to store configuration files.  Default to '/etc/rsyslog.d'.
- `rsyslog_custom_config_files`: List of custom configuration files are deployed to /etc/rsyslog.d.  The format is an array which element is the full path to each custom configuration file.  Default to none.
- `rsyslog_forwards`: array of log type to forward target host to redirect the logs to the remote rsyslog servers using the rsyslog rule.  E.g., `rsyslog_forwards: [ '*.crit @10.10.10.1:514', '*.info @10.10.10.11:514' ]`, where @ means the logs are sent over udp; if there are two @@, they are sent over tcp.  Note: this syntax is deprecated.
- `rsyslog_group`: Owner group of rsyslogd.  Default to 'syslog' if `rsyslog_unprivileged` is True.  Otherwise, 'root'.
- `rsyslog_purge_original_conf`: By default, the Rsyslog configuration files are applied on top of pre-existing configuration files. To purge local files prior to setting new ones, set `rsyslog_purge_original_conf` variable to 'True', it will move all Rsyslog configuration files to a backup directory before deploying the new configuration files. Defaults to 'False'.
- `rsyslog_system_log_dir`: System log directory.  Default to '/var/log'.
- `rsyslog_unprivileged`: If set to True, you could specify non-root `rsyslog_user` and `rsyslog_group`.  If ansible_distribution is one of "CentOS", "RedHat", and "Fedora", default to True.  Otherwise, 'False'.
- `rsyslog_user`: Owner user of rsyslogd.  Default to 'syslog' if `rsyslog_unprivileged` is 'True'.  Otherwise, 'root'.
- `rsyslog_work_dir`: Working directory.  Default to '/var/lib/rsyslog'.

Viaq sub-variables
------------------
- `rsyslog_viaq_config_dir`: Directory to store viaq configuration files.  Default to '/etc/rsyslog.d/viaq'.
- `rsyslog_viaq_log_dir`: Viaq log directory.  Default to '/var/log/containers'.
- `logging_mmk8s_token`: Path to token for kubernetes.  Default to "/etc/rsyslog.d/viaq/mmk8s.token"
- `logging_mmk8s_ca_cert`: Path to CA cert for kubernetes.  Default to "/etc/rsyslog.d/viaq/mmk8s.ca.crt"
- `use_omelasticsearch_cert` : If set to 'True', omelasticsearch is configured to use the certificates specified in `rsyslog_elasticsearch`.  Default to 'False'.
- `use_local_omelasticsearch_cert` : If set to 'True', local files ca_cert_src, cert_src and key_src in `rsyslog_elasticsearch` will be deployed to the remote host.

- `rsyslog_elasticsearch`: A set of following variables to specify output elasticsearch configurations. It could be an array if multiple elasticsearch clusters to be configured.
   If set to 'True', ca_cert_src, cert_src and key_src must be set in each `rsyslog_elasticsearch` element. Otherwise, the deployment fails. Default to 'False'.
  - `name`: Name of the elasticsearch element.
  - `server_host`: Hostname elasticsearch is running on.
  - `server_port`: Port number elasticsearch is listening to.
  - `index_prefix`: Elasticsearch index prefix the particular log is to be indexed.
  - `ca_cert`: Path to CA cert for ElasticSearch.  Default to '/etc/rsyslog.d/elasticsearch/es-ca.crt'
  - `cert`: Path to cert for ElasticSearch.  Default to '/etc/rsyslog.d/elasticsearc/es-cert.pem'
  - `key`: Path to key for ElasticSearch.  Default to "/etc/rsyslog.d/elasticsearch/es-key.pem"
  - `ca_cert_src`: Path to the local CA cert file to deploy for ElasticSearch.
  - `cert_src`: Path to the local cert file to deploy for ElasticSearch.
  - `key_src`: Path to the local key file to deploy for ElasticSearch.

Contents of Roles
=================
It contains the framework and data for the configuration files to be deployed.

The basic framework is based on debops.rsyslog and adjusted to the RHEL/Fedora specification.

- templates have 2 template files, rsyslog.conf.j2 and rules.conf.j2.  The former is used to generate /etc/rsyslog.conf and the latter is for the other configuration files including mmnormalize rulebase and formatter which will be placed in ```{{rsyslog_config_dir}}``` (default to /etc/rsyslog.d) and its subdirectories.

- tasks/main.yaml contains the sceries of tasks to deploy specified set of configuration files.

If viaq is in `rsyslog_logs_collections`, the following tasks are executed.
```
TASK [rsyslog : Install/Update required packages]
TASK [rsyslog : Create required system group]
TASK [rsyslog : Create required system user]
TASK [rsyslog : Create a work directory]
TASK [rsyslog : Create a temp directory for rsyslog.d backup]
TASK [rsyslog : Set backup dir name]
TASK [rsyslog : Create a backup dir]
TASK [rsyslog : Moving the contents of /etc/rsyslog.d to the backup dir]
TASK [rsyslog : create rsyslog viaq subdir]
TASK [rsyslog : Update directory and file permissions]
TASK [rsyslog : Generate main rsyslog configuration]
TASK [rsyslog : Generate viaq configuration files in rsyslog.d]
TASK [rsyslog : Generate rsyslog viaq configuration files in rsyslog.d/viaq]
```
If basics is in `rsyslog_logs_collections` the following tasks are executed.
```
TASK [rsyslog : Install/Update required packages]
TASK [rsyslog : Create required system group]
TASK [rsyslog : Create required system user]
TASK [rsyslog : Create a work directory]
TASK [rsyslog : Create a temp directory for rsyslog.d backup]
TASK [rsyslog : Set backup dir name]
TASK [rsyslog : Create a backup dir]
TASK [rsyslog : Moving the contents of /etc/rsyslog.d to the backup dir]
TASK [rsyslog : create rsyslog viaq subdir]
TASK [rsyslog : Update directory and file permissions]
TASK [rsyslog : Generate main rsyslog configuration]
TASK [rsyslog : Generate exaple configuration files in rsyslog.d]
```
WARNING: Pre-existing rsyslog.conf and configuration files in /etc/rsyslog.d are moved to the backup directory /tmp/rsyslog.d-XXXXXX.  If the pre-existing files need to be merged with the newly deployed files, you need to do it manually.

-defaults/main.yaml defines variables to switch the deployment paths, variables to specify the locations to deploy and the configurations to be deployed.

Describing how the configuration files are defined to be deployed using the viaq case.

Viaq configuration files are defined in {{rsyslog_viaq_rules}} in /roles/input_rols/viaq/defaults/main.yaml.  The set is made from the generic modules{rsyslog_conf_global_options, rsyslog_conf_local_modules, rsyslog_conf_network_modules, rsyslog_conf_common} and viaq specific configurations.

To make a new configuration file installed in addition to the current {{rsyslog_viaq_rules}}, create an rsyslog config item based on the following skelton and add the title {{rsyslog_conf_yourname}} to {{rsyslog_viaq_rules}}.
```
rsyslog_conf_yourname:

  - name: 'somename'
    type: choose one of 'global' 'module' 'modules' 'template' 'templates' 'output' 'service' 'rule' 'rules' 'ruleset' 'input'
    path: path this configuration file to be placed if it's not {{rsyslog_config_dir}}.
    nocomment: 'true' if you want to avoid "# Ansible managed" to be added at the top of the file.
    sections:

      - options: |-
          # COMMENTS
          your rsyslog configuration

```
Type is for adding prefix to the file name to manage the order of the configuration loaded.  In the viaq case, only .conf files set type 'modules', 'output', and 'template' are set.  By setting 'modules', for instance, prefix "10-" is added to the "name".  I.e., if the name is "mmk8s" and type is "modules", the file name "10-mmk8s.conf" is constructed.  'template' is mapped to '20-'; 'output' is mapped to '30-'.  The digits ensure the configuration files are loaded in the correct order.  The type and prefix mapping is defined in rsyslog_weight_map in ./roles/rsyslog/defaults/main.yaml.

If the deploy destination is other than {{rsyslog_config_dir}}, the path is to be set to path.

By default, the generated configuration file starts with a comment "# Ansible managed".  It could break some type of configurations.  For instance, "version=2" must be the first line in a rulebase file.  To avoid having "# Ansible managed", set true to nocomment.

Some full path configuration may be referred from other configuration file, e.g., 20-viaq_formattiong.conf refers parse_json.rulebase as follows.
```
action(type="mmnormalize" ruleBase="{{ rsyslog_viaq_config_dir }}/parse_json.rulebase" variable="$!MESSAGE")
```
In this case, prefix is not needed.  Thus, by setting exact filename, the named configuration file "parse_json.rulebase" is generated.
```
  - name: 'parse_json'
    filename: 'parse_json.rulebase'
    nocomment: 'true'
    path: '{{ rsyslog_viaq_config_dir }}'
    sections:

      - options: |-
          version=2
          rule=:%.:json%
```
